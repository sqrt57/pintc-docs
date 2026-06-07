# Pint Roadmap

## Pre-Implementation

Decisions and artifacts needed before writing the compiler:

- **Formal grammar** — separate file `pint-grammar.md`; EBNF for implementor reference and LLM use.
- **Command line** — separate file `pintc-cli.md` (in `pintc-docs`); input files, output file, target flags, error format.
- **Implementation platform** — ✓ C#. Project `pintc-cs`, binary `pintc`. Analysis: [implementation-platform.md](implementation-platform.md).

## Implementation

Design: [pintc-impl.md](pintc-impl.md)

### Phases

Single-pass pipeline — each phase transforms one representation into the next.
No optimisation passes in v1.

| Phase | Input | Output |
|---|---|---|
| Lex | Source text | Token stream |
| Parse | Token stream | AST |
| Resolve | AST | AST (names bound, imports wired) |
| Type-check | AST (names bound) | Typed AST |
| Codegen | Typed AST | x86 machine code |
| PE emit | x86 machine code + metadata | PE32 binary |

### Intermediate representations

- **AST** — untyped; direct parser output; nodes carry source spans for error reporting.
- **Typed AST** — AST annotated with resolved types and name bindings; the only IR in v1; both type-checker and codegen operate on it.

No SSA or three-address IR in v1 — the language is simple enough that direct stack-based codegen from the typed AST is tractable.

### Code generation strategy

Stack-based x86 for v1: all temporaries spill to the stack; no register allocator.
Produces correct code; register allocation and peephole optimisation deferred to future work.

### Output

Writes PE32 directly — no dependency on an assembler or linker.

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
