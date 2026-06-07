# Pint Roadmap

## Pre-Implementation

Decisions and artifacts needed before writing the compiler:

- **Formal grammar** — ✓ [pint-grammar.md](pint-grammar.md); Go-style EBNF for implementor reference and LLM use.
- **Command line** — ✓ [pintc-cli.md](pintc-cli.md); input files, output file, target flags, error format.
- **Implementation platform** — ✓ C#. Project `pintc-cs`, binary `pintc`. Analysis: [implementation-platform.md](implementation-platform.md).

## Implementation

Design: [pintc-impl.md](pintc-impl.md)

Slice 1 produces a real, runnable PE32 exe from a narrow subset of Pint.
Each subsequent slice adds one feature group end-to-end — touching every phase from
lexer to PE emit. The compiler always builds and runs between slices.

### Slice 1 — End-to-end skeleton

Target program:

```pint
module main {

    [dll_import(dll="kernel32.dll", entry_point="ExitProcess")]
    [noreturn]
    extern fun exit_process(code: u32) -> ();

    [win32_entry]
    [noreturn]
    fun main() -> () {
        exit_process(0);
    }
}
```

Produces a real Windows EXE that exits cleanly.

| Step | Deliverable |
|---|---|
| Scaffolding | `Pintc.sln`, `Pintc/`, `Pintc.Tests/`, `Program.cs` CLI stub |
| Lexer (subset) | Keywords (`module`, `extern`, `fun`), identifiers, integer literals, string literals, punctuation |
| Parser (subset) | Module decl, extern decl with `[dll_import]`, fun decl with `[win32_entry]`/`[noreturn]`, call expr, int literal |
| Resolver (subset) | Single-module only; bind call targets to extern decls |
| Type checker (subset) | Structural pass-through; verify call arity only |
| Codegen (subset) | Function prologue/epilogue, push literal, stdcall call, `ret` |
| PE32 writer (subset) | DOS stub + COFF + Optional header + `.text` + `.idata` (single import DLL) |
| CLI | `pintc source.pnt` → `source.exe` |

### Slice 2 — Arithmetic and locals

Integer arithmetic (`+`, `-`, `*`, `/`, `%`), comparison operators (`<`, `>`, `<=`, `>=`,
`==`, `!=`), boolean literals and logic (`and`, `or`, `not`), local variables (`var`),
module-level constants (`const`).

### Slice 3 — Conditionals

`if`/`else` statements.

### Slice 4 — Loops

`while`, `loop`, `break`, `continue`.

### Slice 5 — `for`

`for` statements.

### Slice 6 — Records and type aliases

Record declarations, record literals, field access (`.`), `type` aliases.

### Slice 7 — Enums

Enum declarations, variant references, enum equality.

### Slice 8 — Arrays

Array types, array literals, index expressions.

### Slice 9 — Pointers

Pointer types (`^T`), address-of, dereference, arrow (`->`), pointer arithmetic.

### Slice 10 — Multiple return values

Tuple return types, multi-assign statements, hidden pointer calling convention.

### Slice 11 — Strings, chars, floats

String literals (`.rdata`), char literals, `f32`/`f64` arithmetic.

### Slice 12 — Multi-module

`import`/`export`, cross-module name resolution, module-scope `var` (`.data` section),
DLL output (`.edata`).

### Slice 13 — Builtins

`cast`, `sizeof`, `divmod`, `mul`, `length`, `to_*` conversion builtins.

## Future Work

Features worth revisiting once the initial version is working:

| Feature | Notes |
|---|---|
| Static linking | Foundation for .lib / .obj interop and self-contained binaries without DLL dependencies |
| Macros | Compile-time code generation; deferred to keep the language simple and readable in v1 |
| Bounds checking | Optional debug mode; overhead unacceptable in production but useful for development |
| Fat pointers / slices | Length-carrying pointer types for safer array passing; conflicts with "no hidden state in pointers" — needs design work |
| Concurrency primitives | Language-level threads or async; use OS APIs for now |
| Additional targets | x86-64, Linux/ELF, WASM, ARM/ARM64; calling conventions and ABI details are platform-specific |
| Packed records | `[packed]` attribute — no alignment padding; useful for network packets, binary file formats, hardware registers |
| Bit fields | Sub-byte fields packed into an integer; essential for hardware register definitions and protocol headers; requires new read/write semantics |
| Heap alignment helpers | Wrappers or intrinsics for aligned heap allocation; currently requires manual use of platform APIs (`VirtualAlloc`, `_aligned_malloc`) |
| `fastcall` | Register-based calling convention attribute for hot inner functions; requires ABI design for register assignment and platform-specific handling |
| `switch`/match | Enum exhaustiveness checking; `if`/`else if` is adequate for v1 but a match construct with exhaustiveness would improve correctness for enum dispatch |
