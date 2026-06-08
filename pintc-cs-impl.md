# pintc Implementation

C# compiler for the Pint language. Produces Win32 native binaries (x86 PE32).
See [pint-spec-v0.7.md](pint-spec-v0.7.md) for the language,
[pint-grammar.md](pint-grammar.md) for the formal grammar, and
[pintc-cli.md](pintc-cli.md) for the command-line interface.

---

## Toolchain

| Concern | Choice |
|---|---|
| **.NET version** | .NET 10 (LTS) |
| **Build** | `dotnet` CLI, standard `.sln` / `.csproj` — no build orchestration layer |
| **CLI parsing** | Manual — the CLI is simple enough that a library would be dead weight |
| **Unit tests** | xUnit |
| **Mocking** | NSubstitute — add when first needed |
| **Assertions** | Shouldly — add when first needed |
| **Parser** | Hand-written recursive descent — no parser generator |
| **Codegen / PE32** | Written by hand — no assembler or linker dependency |

---

## Repository layout

```
pintc-cs/
  Pintc.slnx
  Pintc/
    Token.cs          ← token kinds and the Token record
    Lexer.cs
    Ast.cs            ← all AST node types
    Parser.cs
    Resolver.cs       ← name binding, module wiring
    TypeChecker.cs
    TypeContext.cs    ← symbol tables and type annotations
    Codegen.cs        ← typed AST → x86 bytes
    X86.cs            ← instruction encoding helpers
    PeWriter.cs       ← PE32 binary writer
    Diagnostic.cs
    Program.cs        ← CLI entry point
  Pintc.Tests/
    ...                     ← unit tests — pure in-process, no OS interaction
  Pintc.IntegrationTests/
    ...                     ← integration tests — compiler internals + OS (runs EXEs)
  Pintc.E2eTests/
    Helpers/
      CompilerRunner.cs     ← invokes pintc, runs output EXE, returns exit code + output
    Slice1Tests.cs
    ...                     ← one test class per slice
```

### Test project boundaries

| Project | References `Pintc`? | OS interaction? | What goes here |
|---|---|---|---|
| `Pintc.Tests` | yes | no | Lexer, parser, resolver, type checker, byte assertions |
| `Pintc.IntegrationTests` | yes | yes | Components that produce real files or run EXEs (e.g. PE writer) |
| `Pintc.E2eTests` | no | yes | Full pipeline: source file → `pintc` subprocess → run output EXE |

### E2e test project

`Pintc.E2eTests` has no project reference to `Pintc` — it treats the compiler as a black box.
Each test:

1. Writes a source string to a temp directory.
2. Invokes `pintc` via `CompilerRunner.Compile` — uses `PINTC_EXE` env var if set
   (CI with a published binary), otherwise falls back to `dotnet run --project Pintc`.
3. Asserts compile exit code 0.
4. Runs the resulting EXE via `CompilerRunner.Execute`.
5. Asserts run exit code.

Temp directories are created in the test constructor and deleted in `Dispose`.
Source text is inlined in each test class so the test is self-contained.

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

- `Dictionary<int, PintType> NodeTypes` — key: `NodeId` (unique int stamped on each `Expr` by the parser); value: the resolved `PintType` of that expression. Codegen consults this to know the type at each point in the tree.
- `Dictionary<Decl, PintType> DeclTypes` — key: a `Decl` node (function, variable, parameter, etc.); value: the declared or inferred `PintType` of that declaration. Used by the resolver and codegen to look up the type of any named entity.

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
