# pintc-cs — Implementation Snapshot

**As of Slice 11 (2026-06-09). 158 tests passing (138 unit, 4 integration, 16 e2e).**

This is a "what's actually built" reference, distinct from `pintc-cs-impl.md` (design
intent). Read this file at the start of any coding session to orient quickly without
re-reading the source.

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

`Pintc.E2eTests` has no project reference to `Pintc` — it treats the compiler as a
black box. It invokes `pintc` via `CompilerRunner` (uses `PINTC_EXE` env var if set;
falls back to `dotnet run --project Pintc.csproj`).

---

## Pipeline (Program.cs)

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

- `--dll` flag sets `isDll = true` → DLL output path.
- `-o path` sets output file. Default extension is `.exe` or `.dll`.
- Multiple input files allowed; all modules accumulate into `allModules`.

---

## AST (`Pintc/Ast.cs`)

All AST nodes are C# `record`s. No `SourceSpan` on nodes yet (planned).

### Expressions (abstract base: `Expr`)

| Record | Fields |
|--------|--------|
| `IntLiteralExpr` | `long Value` |
| `BoolLiteralExpr` | `bool Value` |
| `VarRefExpr` | `string Name` |
| `CallExpr` | `string? Qualifier, string Name, List<Expr> Args` |
| `BinaryExpr` | `BinaryOp Op, Expr Left, Expr Right` |
| `UnaryExpr` | `UnaryOp Op, Expr Operand` |
| `IndexExpr` | `string ArrayName, Expr Idx` |
| `FieldAccessExpr` | `string VarName, List<string> Path` |
| `AddressOfExpr` | `Expr Operand` |
| `DerefExpr` | `Expr Ptr` |
| `ArrowExpr` | `Expr Ptr, string Field` |

`CallExpr.Qualifier` is non-null for qualified calls (`C.add(...)` → Qualifier="C").

### Statements (abstract base: `Stmt`)

`CallStmt`, `ReturnStmt`, `LocalVarDecl`, `AssignStmt`, `IndexAssignStmt`,
`FieldAssignStmt`, `DerefAssignStmt`, `ArrowAssignStmt`, `IfStmt`, `WhileStmt`,
`LoopStmt`, `BreakStmt`, `ContinueStmt`, `ForStmt`

`ForStmt` carries: VarName, VarTypeName, VarInit, Condition, PostName, PostValue, Body.

### Declarations

```csharp
record Param(string Name, string TypeName);
record Attr(string Name, Dictionary<string,string> Args);
record RecordField(string Name, string TypeName);
record RecordDecl(string Name, List<RecordField> Fields);
record ExternFunDecl(List<Attr> Attributes, string Name, List<Param> Params, string ReturnType);
record FunDecl(List<Attr> Attributes, string Name, List<Param> Params, string ReturnType, List<Stmt> Body);
record ModuleVarDecl(string Name, string TypeName, Expr? Init);
record ImportDecl(string ModuleName, string Alias);
record ModuleDecl(string Name, List<ExternFunDecl> Externs, List<FunDecl> Funs,
                  List<ModuleVarDecl> Vars, List<RecordDecl> Records,
                  List<ImportDecl> Imports, List<string> Exports);
// 5-param compat ctor omits Imports/Exports (used by unit/integration tests)
```

### Known attributes

| Attribute | Meaning |
|-----------|---------|
| `[win32_entry]` | PE entry point; function emitted first at code offset 0 |
| `[noreturn]` | Suppresses epilogue emission |
| `[dll_import(dll="...", entry_point="...")]` | Extern function imported from DLL |
| `[dll_export]` | Export function from a `--dll` compilation; triggers stdcall |

### Type strings

Types are stored as plain strings — no type AST node yet:
- Primitives: `u8`, `u16`, `u32`, `u64`, `i8`, `i16`, `i32`, `i64`, `bool`, `byte`, `usize`, `isize`, `()`
- Arrays: `"[N]T"` e.g. `"[3]u32"`
- Pointers: `"^T"` e.g. `"^u32"`, `"^Point"`

---

## Lexer / Token

- `Token` is `(TokenKind Kind, SourceSpan Span, string Text)`.
- All ~40 keywords reserved including `return`, `export`, `import`.
- `as` is NOT a keyword — lexed as `Ident`.

---

## Parser (`Pintc/Parser.cs`)

Hand-written recursive descent.

### Key entry points

| Method | Returns | Notes |
|--------|---------|-------|
| `ParseProgram()` | `List<ModuleDecl>` | Loops `ParseModule` until EOF |
| `ParseModule()` | `ModuleDecl?` | Handles `fun`, `extern fun`, `var`, `record`, `import`, `export` |
| `ParseImportDecl()` | `ImportDecl?` | `import Mod as Alias;` |
| `ParseExportDecl()` | `string?` | `export Name;` |
| `ParseReturnStmt()` | `ReturnStmt?` | `return expr?;` |
| `ParseCallArgs(q, n)` | `CallExpr?` | Shared by qualified and unqualified call parsing |
| `ParseBlock()` | `List<Stmt>?` | `{ stmts }` |
| `PeekAt(int)` | `Token?` | Look-ahead beyond current position |

### `ParseStmt` dispatch

Lookahead determines which parser is called:

- `return` → `ParseReturnStmt`
- `var` → `ParseLocalVarDecl`
- `if` / `for` / `while` / `loop` / `break` / `continue` → respective parsers
- `ident . ident =` → `ParseFieldAssignStmt`
- `ident =` → `ParseAssignStmt`
- `ident [` → `ParseIndexAssignStmt`
- `ident ^` → `ParseDerefAssignStmt`
- `ident ->` → `ParseArrowAssignStmt`
- fallthrough → `ParseCallStmt` (`name(args);`)

**Known gap:** `ident.ident(...)` as a *statement* is not handled — only works in
expression position (as `CallExpr`).

### Expression precedence (13 levels, low to high)

`or > and > | > xor > & > == != > < <= > >= > << >> > + - > * / % > unary > primary`

### `ParsePrimaryExpr` handles

- Int/bool literals
- `@expr` → `AddressOfExpr`
- `ident(...)` → unqualified `CallExpr`
- `ident.ident(...)` → qualified `CallExpr`
- `ident[i]` → `IndexExpr`
- `ident.field.field...` → `FieldAccessExpr`
- Postfix `^` → `DerefExpr`
- Postfix `->field` → `ArrowExpr`

---

## Resolver (`Pintc/Resolver.cs`)

```csharp
Resolve(List<ModuleDecl>) → ResolveResult(Symbols, Diagnostics)
Resolve(ModuleDecl)       → wrapper: Resolve([module])
```

`ResolveResult.Symbols` = `Dictionary<string, FunSymbol>` keyed by function name.
`FunSymbol` variants: `ExternFunSymbol(ExternFunDecl)`, `LocalFunSymbol(FunDecl)`.

**What it checks:** only top-level `CallStmt` callees in each `fun.Body` are in the
symbol table. Does **not** scan nested blocks (if/while/for bodies) or `CallExpr`
in expression position.

---

## TypeChecker (`Pintc/TypeChecker.cs`)

```csharp
Check(List<ModuleDecl>, ResolveResult) → IReadOnlyList<Diagnostic>
Check(ModuleDecl, ResolveResult)       → wrapper
```

**What it checks:** only top-level `CallStmt` arity in `fun.Body`. Does not check
`CallExpr` arity, `ReturnStmt` type, or statements inside nested blocks.

---

## Codegen (`Pintc/Codegen.cs`)

### Key types

```csharp
record LocalCallRef(int PatchOffset, string ModuleName, string FuncName);

record FunCtx(
    Dictionary<string, ImportSpec>      ImportMap,
    Dictionary<string, uint>            VarVas,
    Dictionary<string, int>             Offsets,       // locals → neg EBP; params → pos EBP
    Dictionary<string, string>          Types,
    Dictionary<string, RecordDecl>      RecordMap,
    List<byte>                          Code,
    List<IatRef>                        IatRefs,
    List<ImportSpec>                    Imports,
    string                              ModuleName,
    IReadOnlyDictionary<string, string> AliasMap,      // alias → target module name
    List<LocalCallRef>                  LocalCallRefs,
    int                                 LocalBytes,
    bool                                NeedsFrame,
    bool                                IsStdcall,     // true for [dll_export] functions
    int                                 ParamStackBytes); // N for ret N (stdcall)
```

### `Emit(List<ModuleDecl>, isDll)` flow

**EXE mode (`isDll=false`):**
1. Merge all Externs, Vars, Records from all modules.
2. Build `importMap` (extern name → ImportSpec), `varVas`, `recordMap`.
3. Build per-module alias maps (`alias → moduleName`).
4. Find `[win32_entry]` function → emit it **first** (code offset 0 = PE entry point).
5. Emit remaining functions in declaration order.
6. Backpatch all `LocalCallRef` displacements.
7. Return `CodeUnit` with `ExportedFuns = []`.

**DLL mode (`isDll=true`):**
- No `[win32_entry]` requirement.
- Emit all functions in declaration order.
- After backpatching, collect `[dll_export]` functions with their code offsets into `ExportedFuns`.

### `EmitFun`

- `CollectLocals` scans body recursively (`LocalVarDecl`, `ForStmt`, nested blocks)
  → allocates negative EBP slots in declaration order.
- Parameters added to `Offsets` at positive EBP: `[ebp+8]` = first, `[ebp+12]` = second, etc.
- `needsFrame = !isEntryPoint || localBytes > 0` — always true for non-entry functions.
- Frame prologue: `push ebp; mov ebp, esp; sub esp, N` (omitted if `!needsFrame`).
- Epilogue (skipped if `[noreturn]`): `leave` or `pop ebp`, then `ret` or `ret N`.
- `[dll_export]` → `isStdcall = true`, `paramStackBytes = params.Count * 4` → `ret N`.

**Note:** functions with explicit `return` stmts followed by `!noreturn` epilogue emit
dead bytes (trailing ret). Harmless, not worth fixing yet.

### Calling conventions

| Context | Convention | Stack cleanup | Call instruction |
|---------|-----------|---------------|-----------------|
| `CallStmt` (extern/DLL) | stdcall | callee (`ret N`) | `call [IAT_slot]` indirect |
| `CallExpr` with name in `importMap` | stdcall | callee | `call [IAT_slot]` indirect |
| `CallExpr` local Pint function | cdecl | caller (`add esp, N`) | `call rel32` + backpatch |

`EmitCallExpr` checks `importMap` first: if the unqualified callee name is an extern,
use IAT indirect. Otherwise use `call rel32` (intra-binary, cdecl).

### Stack slot sizes

- All scalars and pointers: 4 bytes.
- Arrays `[N]T`: N × slot(T).
- Records: sum of field slot sizes, recursive.
- `StackSlotSize(type, recordMap)` implements this.

---

## CodeUnit (`Pintc/CodeUnit.cs`)

```csharp
record ExportedFun(string Name, int CodeOffset);
record ImportSpec(string DllName, string EntryPoint);
record IatRef(int CodeOffset, ImportSpec Import);

class CodeUnit {
    byte[]            Code;         // raw x86 machine code
    List<IatRef>      IatRefs;      // sites where PeWriter writes 4-byte IAT VAs
    List<ImportSpec>  Imports;      // unique DLL imports referenced
    byte[]            Data;         // .data section content; empty = no .data
    List<ExportedFun> ExportedFuns; // DLL exports; empty for EXE
}
```

---

## PE Writer (`Pintc/PeWriter.cs`)

### EXE (`PeWriter.Write`)

Fixed layout, no relocations:

| | Value |
|--|--|
| `ImageBase` | `0x00400000` |
| `TextRva` | `0x1000` |
| `DataRva` | `0x2000` (only if module-scope vars exist) |
| `IdataRva` | `0x2000` (no .data) or `0x3000` (with .data) |
| `AddressOfEntryPoint` | `TextRva` (entry function **must** be at code offset 0) |
| Sections | `.text`, `.data` (optional), `.idata` |
| COFF Characteristics | `0x0102` (EXECUTABLE_IMAGE \| 32BIT_MACHINE) |

### DLL (`PeWriter.WriteDll`)

| | Value |
|--|--|
| `ImageBase` | `0x00400000` |
| `TextRva` | `0x1000` |
| `EdataRva` | `0x2000` |
| `DataRva` | `0x3000` (if module-scope vars present) |
| `IdataRva` | `0x3000` or `0x4000` (depending on .data presence) |
| `AddressOfEntryPoint` | `0` (no DllMain) |
| Sections | `.text`, `.edata`, `.data` (optional), `.idata` (optional) |
| COFF Characteristics | `0x2102` (EXECUTABLE_IMAGE \| 32BIT_MACHINE \| DLL) |
| Data dir [0] | Export directory RVA + size |

`BuildEdata` constructs `IMAGE_EXPORT_DIRECTORY`:
- Address table: function RVAs in declaration order.
- Name pointer table: name RVAs sorted alphabetically (PE spec binary-search requirement).
- Ordinal table: maps sorted name index → 0-based index into address table.

IAT patching works the same in both modes: `IatRef` records are resolved to absolute
VAs and written into the cloned code buffer before writing to the output stream.

---

## X86 helpers (`Pintc/X86.cs`)

Selected (not exhaustive):

```
PushImm8(byte) / PushImm32(uint) / PushMem32(uint)
PushEbp() / PopEbp() / PushEax() / PushEdx() / PopEax() / PopEcx()
PushEbpDisp8(sbyte)   — load local/param [ebp±disp]
PopToEbpDisp8(sbyte)  — store to local/param
MovEbpEsp() / SubEspImm8(byte) / Leave()
Ret() / RetN(ushort)  — C3 vs C2 nn nn (stdcall cleanup)
CallIndirectMem()     — FF 15 xx xx xx xx; CallIndirectMemAddressOffset = 2
CallRel32()           — E8 00 00 00 00; CallRel32DispAt = 1
AddEspImm8(byte)      — 83 C4 imm8  (cdecl caller cleanup)
Backpatch(code, patchAt, target)  — writes rel32 displacement
JzRel32() / JmpRel32()            — with 4-byte placeholder
TestEaxEax()
MovEaxEbpEcx4Disp8(sbyte) / MovEbpEcx4Disp8Eax(sbyte)  — SIB array access
LeaEaxEbpDisp8(sbyte) / LeaEaxEbpEcx4Disp8(sbyte)        — address-of
MovEaxMemEax() / MovEaxMemEaxDisp8(sbyte)                 — pointer deref read
MovMemEcxEax() / MovMemEcxDisp8Eax(sbyte)                 — pointer deref write
ImulEcxImm8(byte)     — pointer arithmetic stride scaling
Arithmetic: AddEaxEcx / SubEaxEcx / ImulEaxEcx / DivEcx / NegEax / NotEax
Bitwise:    AndEaxEcx / OrEaxEcx / XorEaxEcx / ShlEaxCl / ShrEaxCl
Compare:    CmpEaxEcx + Sete/Setne/Setb/Setbe/Seta/Setae Al + MovzxEaxAl
```

---

## E2e test helpers (`Pintc.E2eTests/Helpers/CompilerRunner.cs`)

```csharp
Compile(sourceFile, outputFile)        // pintc "f.pnt" -o "out.exe"
Compile(sourceFiles[], outputFile)     // pintc "a.pnt" "b.pnt" -o "out.exe"
CompileDll(sourceFile, outputFile)     // pintc --dll "f.pnt" -o "out.dll"
Execute(exePath)                       // runs EXE, returns ProcessResult(ExitCode, Stdout, Stderr)
```

Source fixtures live in `Pintc.TestFixtures/SliceFixtures.cs`.
Each e2e test class implements `IDisposable` with a temp directory.

---

## Known limitations / open gaps (post Slice 11)

- `ident.ident(...)` as a **statement** is unhandled (only works in expression position).
- Resolver only checks top-level `CallStmt` callees — not nested blocks, not `CallExpr`.
- TypeChecker only checks `CallStmt` arity — not `CallExpr`, not `return` types.
- Local variable EBP offsets use `sbyte` (fits up to ~127 bytes of locals).
- Parameter EBP offsets use `sbyte` (fits up to ~127 bytes of params, i.e. ~31 params).
- Duplicate epilogue bytes emitted when function ends with explicit `return` and is `!noreturn` (dead code, harmless).
- No type AST — types are plain strings throughout.
- No `SourceSpan` on AST nodes — error messages lack line/column.
- `CollectLocals` allocates one slot per name; sibling for-loop vars with the same name share the slot (harmless today).
