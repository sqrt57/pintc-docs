# Pint Roadmap

## Pre-Implementation

Decisions and artifacts needed before writing the compiler:

- **Formal grammar** — ✓ [pint-grammar.md](pint-grammar.md); Go-style EBNF for implementor reference and LLM use.
- **Command line** — ✓ [pintc-cli.md](pintc-cli.md); input files, output file, target flags, error format.
- **Implementation platform** — ✓ C#. Project `pintc-cs`, binary `pintc`. Analysis: [implementation-platform.md](implementation-platform.md).

## Implementation

Design: [pintc-cs-impl.md](pintc-cs-impl.md)

Slices are vertical — each adds one feature group end-to-end, touching every phase from
lexer to PE emit. This surfaces integration bugs early, when they are cheap to fix, and
keeps the compiler buildable and runnable throughout. The alternative (completing one
phase at a time) defers integration to the end and leaves intermediate phases untestable
in isolation.

Slice 1 produces a real, runnable PE32 exe from a narrow subset of Pint.
Each subsequent slice adds one feature group end-to-end — touching every phase from
lexer to PE emit. The compiler always builds and runs between slices.

### Testing

#### Unit tests

The lexer and parser both take source text as input, so their tests follow the same shape:
feed in a source string, check the output. Lexer tests are table-driven — one row per
scenario. Parser tests use golden snapshots of the serialized AST. The PE32 writer is tested by feeding it a pre-built `CodeUnit` and verifying the output
structure with `dumpbin` or direct header byte assertions.

#### Integration tests

The **resolver** and **type checker** are tested by running source text through
lex → parse → resolve → type-check and asserting on the collected diagnostics. This is
cheaper than end-to-end (no codegen or PE emit) and more realistic than constructing
AST nodes by hand. Both valid-program cases (no diagnostics) and error cases (undefined
name, wrong arity, type mismatch) are covered this way.

#### End-to-end tests

One per slice: compile a `.pnt` source file, run the resulting EXE, check the exit code.
These are the primary correctness signal for codegen and PE emit, and the final check
that all phases integrate correctly. As the slice count grows, the suite doubles as a
regression guard — a passing slice N test confirms that slice N−1 features still work.

#### Not tested in isolation

**Codegen** has no dedicated tests — byte-level output comparison is brittle.
End-to-end tests cover its correctness through behavior.

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
| ✓ Scaffolding | `Pintc.slnx`, `Pintc/`, `Pintc.Tests/`, `Program.cs` CLI stub |
| ✓ CLI | `pintc source.pnt` → `source.exe`; stub pipeline, no real compilation yet |
| ✓ PE32 writer (subset) | DOS stub + COFF + Optional header + `.text` + `.idata` (single import DLL); emit hardcoded EXE and verify it runs |
| ✓ Codegen (subset) | Function prologue/epilogue, push literal, stdcall call, `ret`; feed hardcoded AST into PE32 writer |
| ✓ Lexer (subset) | Keywords (`module`, `extern`, `fun`), identifiers, integer literals, string literals, punctuation |
| ✓ Parser (subset) | Module decl, extern decl with `[dll_import]`, fun decl with `[win32_entry]`/`[noreturn]`, call expr, int literal |
| ✓ Resolver (subset) | Single-module only; bind call targets to extern decls |
| ✓ Type checker (subset) | Structural pass-through; verify call arity only |

#### Tests

| Step | How to verify |
|---|---|
| ✓ Scaffolding | `dotnet build` succeeds; `dotnet test` runs with zero failures |
| ✓ CLI | `pintc foo.pnt` runs without crashing; exits with a clear error (not implemented) |
| ✓ PE32 writer | Hardcoded EXE runs on Windows; exit code matches expected value; `dumpbin /headers` and `dumpbin /imports` show correct structure |
| ✓ Codegen | Compile target program from hardcoded AST; run EXE; assert exit code 0; unit-test emitted x86 byte sequences |
| ✓ Lexer | Table-driven unit tests: source string → expected token list; cover every token type needed by the target program |
| ✓ Parser | Unit tests: source string → expected AST; slice 1 round-trip test confirms parse → codegen produces matching output |
| ✓ Resolver | Assert call to `exit_process` binds to the extern decl; assert unknown identifier produces an error |
| ✓ Type checker | Assert target program passes with no errors; assert wrong-arity call produces the correct error |
| ✓ End-to-end | Compile the target program source file; run the EXE; assert exit code 0 |

Slice 1 is complete. All steps done; 124 tests passing (119 unit, 2 integration, 3 e2e).

### Slice 2 — Module variables ✓

Module-scope `var` for integer and bool types; `.data` section in PE32.
129 tests passing (123 unit, 2 integration, 4 e2e).

### Slice 3 — Local variables ✓

`var` declarations inside function bodies; stack slot allocation.
140 tests passing (131 unit, 4 integration, 5 e2e).

### Slice 4 — Expressions ✓

Binary operators: arithmetic (`+`, `-`, `*`, `/`, `%`), comparison (`<`, `>`, `<=`, `>=`, `==`, `!=`),
boolean logic (`and`, `or`), bitwise (`&`, `|`, `xor`), shift (`<<`, `>>`).
Unary operators: `-`, `~`, `not`. Bool literals (`true`, `false`). Full operator precedence table.
142 tests passing (131 unit, 4 integration, 7 e2e).

### Slice 5 — if/else ✓

`if`/`else` statements; `else if` chains; nested blocks.
143 tests passing (131 unit, 4 integration, 8 e2e).

### Slice 6 — while, loop, break, continue

`while`, `loop`, `break`, `continue`.

### Slice 7 — for

`for` statements.

### Slice 8 — Arrays

Array types (`[N]T`), index expressions (`a[i]`), element assignment (`a[i] = v`).

### Slice 9 — Records

Record declarations, field access (`.`).

### Slice 10 — Pointers

Pointer types (`^T`), address-of (`@`), dereference (`p^`), arrow (`->`), pointer arithmetic.

### Slice 11 — Modules

`import`/`export`, cross-module name resolution, multi-file compilation, DLL output (`.edata`).

### Slice 12 — Local constants

`const` at function scope.

### Slice 13 — Module constants

`const` at module scope; compile-time expression evaluation.

### Slice 14 — Strings and char literals

String literals (`.rdata`), `string` type (`.ptr`, `.len`), char literals (`byte`).

### Slice 15 — Multiple return values

Tuple return types, multi-assign statements, hidden pointer calling convention.

### Slice 16 — Builtins

`cast`, `sizeof`, `divmod`, `mul`, `length`, `to_i8`…`to_u32`.

### Slice 17 — Named arguments

Named call syntax (`f(a: x, b: y)`); order-independent argument passing.

### Slice 18 — Named return values

Named return list in function signatures; named unpack at call site.

### Slice 19 — Array literals

Array literal expressions (`[1, 2, 3]`); element type inferred from context.

### Slice 20 — Record literals

Record literal expressions (`{ x: 1, y: 2 }`); type inferred from context.

### Slice 21 — Enums

Enum declarations, variant references (`E.Member`), equality, cast to/from underlying type.

### Slice 22 — Named break/continue

Labels on loops; `break label` and `continue label` for nested loop control.

### Slice 23 — Floats

`f32`/`f64` literals, arithmetic, and comparisons; FPU stack codegen (x87).

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
