# Pint Roadmap

## Pre-Implementation

Decisions and artifacts needed before writing the compiler:

- **Formal grammar** — decide whether a BNF/EBNF section is needed (for implementor reference and LLM use), or whether the spec prose is sufficient.
- **Command line** — define the compiler's CLI: input files, output file, target flags, error format.
- **Implementation platform** — ✓ C#. Project `pintc-cs`, binary `pintc`. Analysis: [implementation-platform.md](implementation-platform.md).

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
