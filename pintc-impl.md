# pintc Implementation

C# compiler for the Pint language. Produces Win32 native binaries (x86 PE32).
See [pint-spec-v0.7.md](pint-spec-v0.7.md) for the language,
[pint-grammar.md](pint-grammar.md) for the formal grammar, and
[pintc-cli.md](pintc-cli.md) for the command-line interface.

---

## Repository layout

```
pintc-cs/
  Pintc.sln
  Pintc/
    Lexer/
      Token.cs          ← token kinds and the Token record
      Lexer.cs
    Parser/
      Parser.cs
    Ast/
      Nodes.cs          ← all AST node types
    Resolver/
      Resolver.cs       ← name binding, module wiring
    TypeChecker/
      TypeChecker.cs
      TypeContext.cs    ← symbol tables and type annotations
    CodeGen/
      CodeGen.cs        ← typed AST → x86 bytes
      X86.cs            ← instruction encoding helpers
    Pe/
      PeWriter.cs       ← PE32 binary writer
    Diagnostics/
      Diagnostic.cs
    Program.cs          ← CLI entry point
  Pintc.Tests/
    ...
```

---

## Pipeline

Each phase runs to completion before the next begins. Errors are collected within
a phase and reported in bulk; a phase that produces errors aborts the pipeline.

| Phase | Input | Output |
|---|---|---|
| Lex | Source text | `Token[]` |
| Parse | `Token[]` | `Module[]` (AST) |
| Resolve | `Module[]` | `Module[]` (names bound) |
| Type-check | `Module[]` (names bound) | `TypeContext` |
| Codegen | `Module[]` + `TypeContext` | `CodeUnit` |
| PE emit | `CodeUnit` | PE32 file |

---

## AST

C# records. Every node carries a `SourceSpan` (file index, byte offset, length)
for error reporting and future tooling.

```csharp
abstract record Node(SourceSpan Span);

abstract record Decl(SourceSpan Span) : Node(Span);
abstract record Stmt(SourceSpan Span) : Node(Span);
abstract record Expr(SourceSpan Span) : Node(Span);
abstract record TypeRef(SourceSpan Span) : Node(Span);
```

Concrete nodes are sealed records that inherit from these bases. Tree walking uses
C# `switch` expressions with pattern matching — no visitor pattern.

The parser assigns each `Expr` node a unique `int NodeId` used as the key into
`TypeContext` during type-checking.

---

## Name resolution

The resolver runs in two sub-passes:

1. **Collection** — scan all module-scope declarations in all files; populate a
   global name table keyed by `(moduleName, symbolName)`.
2. **Binding** — walk the full AST; resolve every identifier to its declaration;
   wire `import` statements to their target modules.

After resolution every identifier node carries a reference to its `Decl` node.
Unresolved names are errors reported in bulk at the end of the binding pass.

---

## Type checker

Walks the bound AST bottom-up, inferring and checking types. Produces a
`TypeContext`:

- `Dictionary<int, PintType> NodeTypes` — resolved type of every `Expr` node
- `Dictionary<Decl, PintType> DeclTypes` — type of every declaration

`PintType` is a discriminated union (C# abstract record hierarchy):
`PrimitiveType`, `PointerType`, `ArrayType`, `RecordType`, `EnumType`,
`FunctionType`, `UnitType`.

The type checker enforces all spec rules: no implicit conversions, no implicit
deref except through function pointers, attribute constraints (`[win32_entry]`
requires `[noreturn]`, etc.).

---

## Code generation

### Strategy

Stack-based x86 for v1: every sub-expression pushes its result onto the stack;
operators pop and push. No register allocator. Correct; not fast.

Codegen walks the typed AST, consulting `TypeContext` for types, and emits bytes
into a `CodeUnit` (list of named sections with relocation records).

### Calling conventions

| Convention | Used for | Stack cleanup |
|---|---|---|
| `stdcall` | Win32 API; Pint functions (default) | Callee |
| `cdecl` | C runtime; variadic FFI | Caller |

Convention follows the `conv=` attribute on the declaration, defaulting to `stdcall`.

### Multiple return values

Caller allocates space on the stack for all return values, then passes a hidden
pointer as the first argument (before user arguments). Callee writes return values
through the pointer. Single-return functions return the value in `EAX` (integer
and pointer types) or `ST0` (floats) — no hidden pointer.

### Local variable layout

Stack frame layout (high to low address):

```
[saved EBP]
[return address]        ← pushed by CALL
[hidden ret ptr]        ← for multi-return functions only
[user arguments]        ← pushed right-to-left by caller
--- EBP points here ---
[local variables]       ← allocated in declaration order
[temporaries]           ← grows downward during expression eval
```

---

## PE32 output

`PeWriter` assembles a minimal PE32 file:

- **Headers:** DOS stub, PE signature, COFF header, optional header (PE32), section table.
- **Sections:** `.text` (code), `.rdata` (read-only data — string literals), `.data`
  (writable globals), `.idata` (import directory — DLL import table).
- **Relocations:** applied at write time for EXE (fixed base address `0x00400000`);
  relocation table retained for DLL output.
- **Base address:** `0x00400000` for EXE; `0x10000000` for DLL.

No dependency on MASM, NASM, or `link.exe`.

---

## Error reporting

Errors implement `Diagnostic`:

```csharp
record Diagnostic(Severity Severity, SourceSpan Span, string Message);
enum Severity { Error, Warning }
```

Output format (to stderr):

```
path/to/file.pnt:10:5: error: message text
```

Line and column numbers are 1-based. The compiler reports all errors within a
phase before stopping.
