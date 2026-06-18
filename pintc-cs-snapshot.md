# pintc-cs — Implementation Snapshot

**As of Slice 22 (2026-06-18). 190 tests passing (140 unit, 4 integration, 46 e2e).**

---

## TL;DR

- **Slices complete:** 1–22. Next: Slice 23.
- **Pipeline:** Lex → Parse → Resolve → TypeCheck → Codegen → PE emit. All phases wired end-to-end.
- **Output:** PE32 EXE or DLL written directly — no assembler/linker dependency.
- **Codegen:** stack-based x86, no register allocator. All values go through the stack (`push`/`pop`). EAX = expression result; ECX = right operand or scratch.
- **Non-obvious invariants:**
  - `[win32_entry]` function **must be emitted first** (PE `AddressOfEntryPoint = TextRva`, i.e. code offset 0).
  - EBP offsets are `sbyte` — locals limited to ~127 bytes, params to ~31.
  - Resolver and TypeChecker are intentionally narrow: they only check **top-level `CallStmt`** in function bodies. Nested blocks, `CallExpr`, and `return` types are not checked.
  - `FunCtx.Consts` is pre-seeded with evaluated module consts; local `const` decls add/shadow entries as statements execute.
  - Multi-return functions use a **hidden pointer** convention: caller allocates the return buffer, pushes the buffer address last (callee sees it at `[EBP+8]`), and user params shift right to `[EBP+12]`, `[EBP+16]`, etc.

---

## Known gaps (post Slice 15)

- `ident.ident(...)` as a **statement** is unhandled — only works in expression position.
- Resolver only checks top-level `CallStmt` callees — not nested blocks, not `CallExpr`.
- TypeChecker only checks `CallStmt` arity — not `CallExpr`, not `return` types.
- EBP offsets use `sbyte` — locals limited to ~127 bytes; params to ~127 bytes (~31 params).
- Duplicate epilogue bytes when a function ends with explicit `return` and is not `[noreturn]` (dead code, harmless).
- No type AST — types are plain strings throughout.
- No `SourceSpan` on AST nodes — error messages lack line/column.
- `CollectLocals` allocates one slot per name; sibling for-loop vars with the same name share the slot (harmless today).
- `AddEspImm8`/`SubEspImm8` use `imm8` — max 127 bytes; breaks with > 31 return values or args.
- Multi-return calls as expressions (not in `MultiVarDecl`/`MultiAssignStmt`) not supported.

---

## Slice implementation checklist

Every slice touches these files in this order:

| Step | File | What to do |
|------|------|------------|
| 1 | `Pintc/Ast.cs` | Add AST node(s) — `record` inheriting `Expr`, `Stmt`, or new top-level |
| 2 | `Pintc/Parser.cs` | Add parse method; hook into `ParseStmt` dispatch or `ParseModule` |
| 3 | `Pintc/Codegen.cs` | `CollectLocals` case if stack-allocated; `EmitStmts`/`EmitExpr` switch case |
| 4 | `Pintc.TestFixtures/SliceFixtures.cs` | Add `SliceNSource` constant (raw Pint source) |
| 5 | `Pintc.E2eTests/SliceNTests.cs` | New test class with `[Fact]` per scenario |
| 6 | `pintc-docs/` | Update snapshot, ROADMAP, CHANGELOG |

---

## Solution layout

```
pintc-cs/
  Pintc/                       compiler library + CLI (Program.cs)
  Pintc.Tests/                 unit tests — pure in-process, no OS interaction
  Pintc.IntegrationTests/      codegen + PE writer tests — spawns real EXEs
  Pintc.E2eTests/              full pipeline — spawns pintc subprocess, then EXE
  Pintc.TestFixtures/          shared source fixtures (SliceFixtures.cs)
```

`Pintc.E2eTests` has no project reference to `Pintc` — treats the compiler as a black box. Invokes `pintc` via `CompilerRunner` (uses `PINTC_EXE` env var; falls back to `dotnet run --project Pintc.csproj`).

---

## Pipeline (`Pintc/Program.cs`)

```
Source text
  → Lexer.Tokenize()                 → List<Token>
  → Parser.ParseProgram()            → List<ModuleDecl>
  → Resolver.Resolve(modules)        → ResolveResult
  → TypeChecker.Check(modules, res)  → List<Diagnostic>
  → Codegen.Emit(modules, isDll)     → CodeUnit
  → PeWriter.Write(unit, stream)     → PE32 EXE
  PeWriter.WriteDll(unit, name, stream) → PE32 DLL
```

`--dll` sets `isDll = true`. `-o path` sets output file. Multiple input files accumulate into `allModules`.

---

## AST (`Pintc/Ast.cs`)

All nodes are C# `record`s. No `SourceSpan` yet.

### Expressions (`abstract record Expr`)

| Record | Fields |
|--------|--------|
| `IntLiteralExpr` | `long Value` |
| `BoolLiteralExpr` | `bool Value` |
| `VarRefExpr` | `string Name` |
| `CallExpr` | `string? Qualifier, string Name, List<Expr> Args, List<string?>? ArgNames` |
| `BinaryExpr` | `BinaryOp Op, Expr Left, Expr Right` |
| `UnaryExpr` | `UnaryOp Op, Expr Operand` |
| `IndexExpr` | `string ArrayName, Expr Idx` |
| `FieldAccessExpr` | `string VarName, List<string> Path` |
| `AddressOfExpr` | `Expr Operand` |
| `DerefExpr` | `Expr Ptr` |
| `ArrowExpr` | `Expr Ptr, string Field` |
| `CharLiteralExpr` | `byte Value` |
| `StringLiteralExpr` | `byte[] Bytes` |
| `StringConstExpr` | `uint RdataOffset, int ByteCount` |
| `CastExpr` | `Expr Value, string TargetType` |
| `SizeofExpr` | `string TypeName` |
| `LengthExpr` | `string ArrayName` |
| `ToTypeExpr` | `Expr Value, string TargetType` |
| `DivmodExpr` | `Expr A, Expr B` |
| `MulWideExpr` | `Expr A, Expr B` |
| `ArrayLiteralExpr` | `List<Expr> Elements` |

`CallExpr.Qualifier` non-null for qualified calls (`C.add(...)` → Qualifier=`"C"`).
`CallExpr.ArgNames` non-null when any argument is named; parallel to `Args` (null entry = positional). 3-arg compat constructor sets `ArgNames = null`.
`StringConstExpr` is an internal node produced by const-eval when a `StringLiteralExpr` is allocated into `.rdata`; never appears in the parsed AST.

### Statements (`abstract record Stmt`)

`CallStmt`, `ReturnStmt`, `LocalVarDecl`, `LocalConstDecl`, `AssignStmt`, `IndexAssignStmt`,
`FieldAssignStmt`, `DerefAssignStmt`, `ArrowAssignStmt`, `IfStmt`, `WhileStmt`,
`LoopStmt`, `BreakStmt`, `ContinueStmt`, `ForStmt`, `MultiVarDecl`, `MultiAssignStmt`

`ForStmt` carries: VarName, VarTypeName, VarInit, Condition, PostName, PostValue, Body.
`ReturnStmt` carries: `List<Expr> Values, List<string?>? ReturnNames` — `ReturnNames` non-null when named form (`return name: val, ...`).
`MultiVarDecl` carries: `List<(string? Name, string? TypeName)> Items, Expr Call` — `null` Name = discard; `Call` is `CallExpr`, `DivmodExpr`, or `MulWideExpr`.
`MultiAssignStmt` carries: `List<string?> Names, Expr Call, List<string?>? ReturnNames` — `ReturnNames` non-null when named form (`(ret: v, ...) = f()`); `null` Name = discard.
`FunDecl` carries: `List<string?>? ReturnNames` — non-null when the function's return list is named; parallel to the return types.

### Module-level declarations

```csharp
record Param(string Name, string TypeName);
record Attr(string Name, Dictionary<string,string> Args);
record RecordField(string Name, string TypeName);
record RecordDecl(string Name, List<RecordField> Fields);
record ExternFunDecl(List<Attr> Attributes, string Name, List<Param> Params, string ReturnType);
record FunDecl(List<Attr> Attributes, string Name, List<Param> Params, string ReturnType, List<Stmt> Body);
record ModuleVarDecl(string Name, string TypeName, Expr? Init);
record ModuleConstDecl(string Name, string TypeName, Expr Init);
record ImportDecl(string ModuleName, string Alias);
record ModuleDecl(string Name, List<ExternFunDecl> Externs, List<FunDecl> Funs,
                  List<ModuleVarDecl> Vars, List<RecordDecl> Records,
                  List<ImportDecl> Imports, List<string> Exports,
                  List<ModuleConstDecl> Consts, List<EnumDecl> Enums);
// 5-param compat ctor omits Imports/Exports/Consts/Enums (used by unit/integration tests)
```

### Attributes

| Attribute | Meaning |
|-----------|---------|
| `[win32_entry]` | PE entry point; function emitted first at code offset 0 |
| `[noreturn]` | Suppresses epilogue emission |
| `[dll_import(dll="...", entry_point="...")]` | Extern imported from DLL |
| `[dll_export]` | Export from `--dll` compilation; triggers stdcall (`ret N`) |

### Type strings

Types are plain strings — no type AST node yet:
`u8/u16/u32/u64`, `i8/i16/i32/i64`, `bool`, `byte`, `usize`, `isize`, `()`,
`"[N]T"` (arrays), `"^T"` (pointers).

---

## Lexer (`Pintc/Lexer.cs`)

`Token` is `(TokenKind Kind, SourceSpan Span, string Text)`. ~40 keywords reserved. `as` is **not** a keyword — lexed as `Ident`.

---

## Parser (`Pintc/Parser.cs`)

Hand-written recursive descent.

### `ParseModule` dispatch

`fun` → `ParseFunDecl` | `extern` → `ParseExternFunDecl` | `var` → `ParseModuleVarDecl` | `const` → `ParseModuleConstDecl` | `record` → `ParseRecordDecl` | `enum` → `ParseEnumDecl` | `import` → `ParseImportDecl` | `export` → `ParseExportDecl`

### `ParseStmt` dispatch

`return` → `ParseReturnStmt` | `var (` → `ParseMultiVarDecl` | `var` → `ParseLocalVarDecl` | `const` → `ParseLocalConstDecl` | `if/for/while/loop/break/continue` → respective | `(` → `ParseMultiAssignStmt` | `ident . ident =` → `ParseFieldAssignStmt` | `ident =` → `ParseAssignStmt` | `ident [` → `ParseIndexAssignStmt` | `ident ^` → `ParseDerefAssignStmt` | `ident ->` → `ParseArrowAssignStmt` | fallthrough → `ParseCallStmt`

**Known gap:** `ident.ident(...)` as a statement is not handled.

### Expression precedence (13 levels, low → high)

`or > and > | > xor > & > == != > < <= > >= > << >> > + - > * / % > unary > primary`

---

## Resolver (`Pintc/Resolver.cs`)

```csharp
Resolve(List<ModuleDecl>) → ResolveResult(Symbols, Diagnostics)
```

`ResolveResult.Symbols` = `Dictionary<string, FunSymbol>` (`ExternFunSymbol` | `LocalFunSymbol`).

**Scope:** only top-level `CallStmt` callees per function body. Nested blocks and `CallExpr` not checked.

---

## TypeChecker (`Pintc/TypeChecker.cs`)

```csharp
Check(List<ModuleDecl>, ResolveResult) → IReadOnlyList<Diagnostic>
```

**Scope:** only top-level `CallStmt` arity. `CallExpr` arity, `return` types, and nested block statements not checked.

---

## Codegen (`Pintc/Codegen.cs`)

### `Emit` flow

1. Merge all Externs, Vars, Records, Consts from all modules.
2. Build `importMap`, `varVas`, `recordMap`, `moduleConsts` (all const inits evaluated to literals at compile time).
3. Build per-module alias maps.
4. EXE: emit `[win32_entry]` function first (code offset 0), then the rest. DLL: emit all in declaration order.
5. Backpatch all `LocalCallRef` displacements.

### `EmitFun`

- `CollectLocals` scans body recursively → allocates negative EBP slots.
- Params at positive EBP: `[ebp+8]` = first, `[ebp+12]` = second, etc.
- `FunCtx.Consts` seeded with `moduleConsts`; local `const` decls add/shadow entries.
- Prologue: `push ebp; mov ebp,esp; sub esp,N`. Epilogue: `leave`/`pop ebp` + `ret`/`ret N`.

### Module const evaluation

`BuildModuleConstMap` evaluates all `ModuleConstDecl` inits to `IntLiteralExpr`/`BoolLiteralExpr` at compile time. Supports: literals, const-of-consts (forward refs resolved lazily with memoization), unary (`-`, `~`, `not`), binary (all arithmetic/bitwise/comparison ops).

### Calling conventions

| Context | Convention | Cleanup | Instruction |
|---------|-----------|---------|-------------|
| `CallStmt` extern | stdcall | callee `ret N` | `call [IAT]` indirect |
| `CallExpr` in importMap | stdcall | callee | `call [IAT]` indirect |
| `CallExpr` local Pint fn | cdecl | caller `add esp,N` | `call rel32` + backpatch |
| `MultiVarDecl` / `MultiAssignStmt` | cdecl + hidden ptr | caller `add esp,(1+N+1)*4` | `call rel32` + backpatch |

### Multiple return values (Slice 15)

- `fun f(...) -> (T1, T2)` return type parsed as `ParseReturnType`; `ReturnType` stored in `FunCtx`.
- `EmitFun` detects `IsMultiReturn` and shifts params: hidden ptr is pushed last before `call`, callee sees it at `[EBP+8]`; user params shift to `[EBP+12]`, `[EBP+16]`, etc. Callee writes `[ptr+0]`, `[ptr+4]`, … and returns void.
- `EmitMultiVarDecl`: pre-allocates return buffer in the caller's frame (`RetBufOffsets[stmt]`); LEA EAX → save → push user args R-L → reload → push as callee arg → CALL → ADD ESP.
- `EmitMultiAssignStmt`: SUB ESP to allocate a temp buffer → LEA EAX ESP → save → push user args R-L → reload → push as callee arg → CALL → ADD ESP twice → pop each return value into its existing local slot (or ADD ESP 4 for discards).
- `FunCtx` additions: `string? ReturnType`, `List<string> FunReturnTypes`, `Dictionary<MultiVarDecl, int> RetBufOffsets`.

### Named break/continue (Slice 22)

- AST: `BreakStmt(string? Label = null)`, `ContinueStmt(string? Label = null)`. `WhileStmt`, `ForStmt`, `LoopStmt` each gain `string? Label = null` as a trailing optional field (backward-compatible default).
- Parser: labeled loop syntax is `ident ':' (while|for|loop)` at statement position. `ParseStmt` detects this via a 3-token lookahead (`Ident`, `Colon`, `While|For|Loop`) before falling through to other dispatches. `ParseWhileStmt`, `ParseForStmt`, `ParseLoopStmt` each accept an optional `string? label` and set it on the returned node. `ParseBreakStmt`/`ParseContinueStmt` check for `Ident` before `;` and set `Label`.
- Codegen: `FunCtx` gains `Dictionary<string, List<int>> LabeledBreakPatches` and `LabeledContinuePatches` — maps from label name to the patch-site lists of the enclosing labeled loop. Each loop emit method registers its label (if any) into these dicts before emitting the body, and removes it after patching. `EmitStmts` routes `BreakStmt`/`ContinueStmt` with a non-null label to the labeled dict instead of the per-loop local list. Because the dict entries ARE the same `List<int>` objects used by the enclosing loop, all patch sites are collected and patched at the correct target automatically.
- `continue label` on a labeled `while` jumps to the condition re-evaluation (`whileTop`); on a labeled `for` it jumps to the post-step (`postOffset`), matching unlabeled `continue` semantics.
- 190 tests (140 unit, 4 integration, 46 e2e).

### Enums (Slice 21)

- `EnumVariant(string Name, Expr? Value)` and `EnumDecl(string Name, string? UnderlyingType, List<EnumVariant> Variants)` — new AST nodes. `ModuleDecl` gains `List<EnumDecl> Enums` (9th primary-ctor param; 5-param compat ctor passes `[]`).
- Parser: `ParseEnumDecl` — `enum Name [: PrimitiveType] { Variant [= Expr], ... }`. Hooked into `ParseModule` dispatch alongside `record`.
- Codegen: `record EnumInfo(string UnderlyingType, Dictionary<string, long> Variants)`. `BuildEnumMap` evaluates all variant values at compile time — auto-numbering uses `max(0, max(previous)+1)`. `FunCtx` gains `Dictionary<string, EnumInfo> EnumMap`.
- `EmitExpr`: `FieldAccessExpr` where `VarName` is in `EnumMap` pushes the variant's integer value as `imm8` (0–127) or `imm32`. This case is checked before the general field-access path so enum names are never looked up as locals.
- `CastExpr`: if `TargetType` is in `EnumMap`, the underlying type replaces it before the truncation switch — `cast(f, u8)` applies `AND EAX, 0xFF`; `cast(2, Direction)` with default `i32` underlying is a no-op.
- `StackSlotSize`: enum types not in `recordMap` fall through to the default `return 4` — no change needed; all enum values occupy one 4-byte stack slot.
- 186 tests (140 unit, 4 integration, 42 e2e).

### Record literals (Slice 20)

- `RecordLiteralExpr(List<(string Field, Expr Value)> Fields)` — parsed from `{ field: expr, ... }` in primary-expression position.
- `EmitLocalVarDecl` special-cases `RecordLiteralExpr`: delegates to `EmitRecordLiteralInto(lit, typeName, baseOff, ctx)`.
- `EmitRecordLiteralInto` iterates the record's declared fields in order, looks each up by name in the literal, computes `fieldOff` by accumulating `StackSlotSize`, stores each scalar value via `EmitExpr` + `PopToEbpDisp8((sbyte)(baseOff + fieldOff))`. Nested `RecordLiteralExpr` field values recurse.
- Field-order independence falls out naturally: iteration is over the declared field list; the literal's field order is irrelevant.
- 182 tests (140 unit, 4 integration, 38 e2e).

### Array literals (Slice 19)

- `ArrayLiteralExpr(List<Expr> Elements)` — parsed from `[e, e, ...]` in primary-expression position.
- `EmitLocalVarDecl` special-cases `ArrayLiteralExpr`: parses the element type from `decl.TypeName` (`[N]T`), computes `stride = StackSlotSize(T)`, and emits each element into `[EBP + base + i*stride]`. All other contexts leave `ArrayLiteralExpr` unsupported (element type is not available without a typed context).
- 179 tests (140 unit, 4 integration, 35 e2e).

### Named return values (Slice 18)

- `fun f() -> (quot: u32, rem: u32)` — return names parsed by `ParseReturnType(out var retNames)`; stored in `FunDecl.ReturnNames` (null when positional).
- Named return statement: `return rem: a%b, quot: a/b;` — parser detects `ident:` prefix on first value; `ReturnStmt.ReturnNames` holds the given names. `EmitReturnStmt` calls `ReorderReturnValues` to put values in declared order before emission.
- Named assign-unpack: `(quot: q, rem: r) = f();` — parser detects `ident:` at first element; `MultiAssignStmt.ReturnNames` holds return-side names. `EmitMultiAssignStmt` calls `ReorderAssignNames` to put local-variable names in declared return order before popping from the buffer.
- `FunCtx` additions: `List<string?> ReturnNames` (this function's return names), `Dictionary<string, List<string?>> FunReturnNames` (all named-return Pint funs, keyed by name).
- The `var` declaration form is always positional — no named declaration form.
- 177 tests (140 unit, 4 integration, 33 e2e).

### Named arguments (Slice 17)

- Parser: `ParseCallArgs` detects `ident:` at argument position via one-token lookahead; collects names in a parallel `List<string?>` and sets `CallExpr.ArgNames` (null when all positional).
- `FunCtx` gains `Dictionary<string, List<Param>> FunParamLists` — built in `Emit` from all `FunDecl`s, passed through `EmitFun`.
- `ReorderArgs(CallExpr, FunCtx)` — maps named args to declared parameter positions; returns a reordered `List<Expr>` ready for R-L emission.
- Applied in `EmitCallExpr`, `EmitMultiVarDecl`, and `EmitMultiAssignStmt`; positional calls are unaffected (null check, no reorder).

### Builtins (Slice 16)

All builtins are fully inline — no stubs or helper functions.

- `cast(expr, T)` → emit expr; `AND EAX, mask` for `u8`/`u16`; no-op for `u32`.
- `to_u8/u16` → `AND EAX, 0xFF/0xFFFF`; `to_i8/i16` → `MOVSX EAX, AL/AX`; `to_u32/i32` → no-op.
- `sizeof(T)` → `PUSH imm(ByteSize(T))` at compile time; `ByteSize([N]T) = N * Stride(elem)`.
- `length(arr)` → `PUSH imm(N)` from `[N]T` type; compile-time constant.
- `DivmodExpr`/`MulWideExpr` in `MultiVarDecl`/`MultiAssignStmt`: inline `XOR EDX,EDX; DIV ECX` or `MUL ECX`; results written directly to pre-allocated frame slots with `MOV [EBP+disp8], EAX/EDX`. No hidden-pointer protocol needed.

New X86 helpers: `AndEaxImm32`, `MovsxEaxAl`, `MovsxEaxAx`, `MulEcx`, `MovEbpDisp8Eax`, `MovEbpDisp8Edx`.
New Codegen helper: `ByteSize(typeName, recordMap)` — actual byte width (vs. `StackSlotSize` which gives stack-slot width).

### String and char literals (Slice 14)

- `CharLiteralExpr(byte)` → `push imm8`.
- `StringLiteralExpr(byte[])` allocated into `.rdata` (null-terminated) → `StringConstExpr(rdataOffset, byteCount)`.
- `const s: string = "hello"` → `s.ptr` pushes `ImageBase + RdataRva + offset`; `s.len` pushes byte count.
- Pointer arithmetic uses `Stride()` (actual byte width: 1 for `byte/u8/i8/bool`, 2 for `u16/i16`, 4 for everything else) instead of `StackSlotSize()` (always 4).
- Deref of `^byte` emits `mov al,[eax]; movzx eax,al` instead of `mov eax,[eax]`.

### Stack slot sizes

Scalars/pointers: 4 bytes on the stack. Arrays `[N]T`: N × slot(T). Records: sum of fields (recursive). Implemented by `StackSlotSize(type, recordMap)`. For pointer arithmetic and `^byte` dereference, `Stride(type, recordMap)` returns the actual memory width (1, 2, or 4).

---

## CodeUnit (`Pintc/CodeUnit.cs`)

```csharp
class CodeUnit {
    byte[]            Code;         // raw x86
    List<IatRef>      IatRefs;      // patched to absolute IAT VAs by PeWriter
    List<ImportSpec>  Imports;
    byte[]            Data;         // .data section; empty = no .data section
    byte[]            ReadOnly;     // .rdata section (string literals); empty = no section
    List<ExportedFun> ExportedFuns; // DLL exports; empty for EXE
}
```

---

## PE Writer (`Pintc/PeWriter.cs`)

Fixed layout, no relocations. `ImageBase = 0x00400000`, `TextRva = 0x1000`.

EXE: sections `.text`, `.rdata` (optional, RVA 0x2000), `.data` (optional, RVA 0x2000 or 0x3000), `.idata`. Entry point = `TextRva` (code offset 0). `.rdata` shift: when string literals exist, `.data` moves to 0x3000 and `.idata` to 0x4000; `Codegen` computes the dynamic `dataRva` to match.

DLL: sections `.text`, `.edata`, `.data` (optional), `.idata` (optional). `AddressOfEntryPoint = 0`. Export name pointer table sorted alphabetically (PE binary-search requirement).

---

## X86 helpers (`Pintc/X86.cs`)

Slice 15 additions: `LeaEaxEsp` (`8D 04 24`), `MovEcxEbpDisp8` (`8B 4D disp`), `MovEaxEspDisp8` (`8B 44 24 disp`).

See source for the full list. Non-obvious constants: `CallIndirectMemAddressOffset = 2` (byte offset within the `FF 15` encoding where the VA is written), `CallRel32DispAt = 1` (byte offset within `E8` encoding for the rel32 displacement). Used by `EmitCallExpr` and `EmitCallStmt` when recording `IatRef`/`LocalCallRef` patch sites. `MovAlMemEax` (`8A 00`) loads a single byte from `[eax]` into `al`; used with `MovzxEaxAl` for `^byte` dereference.

---

## E2e test helpers

```csharp
// Pintc.E2eTests/Helpers/CompilerRunner.cs
Compile(sourceFile, outputFile)
Compile(sourceFiles[], outputFile)
CompileDll(sourceFile, outputFile)
Execute(exePath) → ProcessResult(ExitCode, Stdout, Stderr)
```

Source fixtures: `Pintc.TestFixtures/SliceFixtures.cs` (`SliceNSource` constants).
Each e2e test class: `IDisposable` + temp directory; `[Fact]` per scenario.
