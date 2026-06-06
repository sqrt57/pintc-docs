# Pint Compiler: Implementation Platform

**Decision: C#. Project `pintc-cs`. Compiler binary `pintc`.**
Naming convention: repo suffix identifies the implementation language (`pintc-cs`, `pintc-rs`, etc.); the binary name `pintc` is implementation-agnostic.

Analysis for choosing the implementation language, focused on the no-libraries constraint.

## Constraints

**No compiler libraries.** Lexer, parser, codegen, and PE/EXE writing are all hand-written.
Standard library is fair game (file I/O, string handling, byte buffers, CLI parsing).
No parser generators, no LLVM bindings, no codegen helpers.

**Target.** Win32 native binary (x86 PE). Codegen = emit raw x86 bytes into a PE file.
The PE format is a struct-filling exercise; x86 encoding is a lookup table. Both
are manageable in any language.

**Success metric.** Simplicity and speed of implementation with Claude Code assistance.

---

## What changes with "no libraries"

The library ecosystem advantage (nom/winnow for Rust, FParsec for F#, LLVM for C++) disappears.
What remains as differentiators:

1. **AST representation** — sum types / discriminated unions vs. interface-based dispatch
2. **Pattern matching** — for codegen visitor, type-checking cases, expression reduction
3. **Binary output** — writing PE headers and x86 bytes into a buffer
4. **Error threading** — propagating source locations and messages through compiler passes
5. **Claude Code confidence** — training data density, feedback loop speed

---

## Candidates

### Rust

**AST.** Recursive enums with `Box<T>` or arena allocation. Pattern matching is clean.
The AST node problem: Rust's borrow checker creates friction for tree structures
— `Box<Node>` works but adds wrapping; arena allocators (typed-arena crate, which is
infrastructure, not compiler-specific) eliminate lifetime fights.

**Binary I/O.** `Vec<u8>` + `write_u32_le!` macros or a simple `ByteWriter` struct.
Very clean for PE and x86 encoding.

**Error handling.** `Result<T, CompileError>` propagates cleanly through all passes
via `?`. Best error threading of any candidate.

**Feedback loop.** `cargo check` is the fastest compiler-roundtrip of any option.
Claude handles borrow checker errors accurately.

**Friction.** Box<> wrapping on every recursive AST node adds noise. For a pint-sized
compiler (pun intended) this is manageable, not showstopping.

**Verdict.** Excellent once past the initial AST shape. Best long-term maintainability.

---

### C#

**AST.** Abstract records with `sealed` subclasses, `switch` expression dispatch.
C# 12 pattern matching is expressive enough for a simple grammar like Pint's.
No sum-type syntax but the pattern is well understood and Claude generates it fluently.

**Binary I/O.** `BinaryWriter` / `MemoryStream` — probably the cleanest of any candidate
for PE writing. Writing a `struct IMAGE_NT_HEADERS` equivalent as a series of
`Write(uint)` calls is natural and readable.

**Error handling.** Exceptions or a hand-rolled `Result<T,E>`. Claude defaults to
exceptions; a discriminated-union-style result type requires nudging.

**Claude confidence.** Highest of any candidate for this kind of structured work.
Roslyn's own architecture is in training data — Claude has deeply internalized
C# compiler patterns.

**Friction.** GC is irrelevant for compiler speed. NativeAOT available if startup time
matters. Main friction: no native sum types, so AST dispatch is slightly more
boilerplate than Rust/F#.

**Existing precedent.** Roslyn (C#'s compiler) is C#. TypeScript compiler is the
same model in a GC'd language.

**Verdict.** Fastest time-to-working-compiler. Best pick purely on speed-of-implementation.

---

### F#

**AST.** Discriminated unions are first-class — the cleanest AST representation of any
candidate:

```
type Expr =
    | IntLit of int32
    | BinOp of Expr * BinOp * Expr
    | Call of string * Expr list
    | Deref of Expr
```

**Pattern matching.** Better than C# for deep nested patterns. Exhaustiveness checking
catches missed cases at compile time.

**Pipeline style.** The `|>` operator makes pass chaining read exactly like a compiler
pipeline: `tokens |> parse |> typecheck |> codegen`.

**Binary I/O.** Same as C# — `BinaryWriter` / `MemoryStream`.

**Error handling.** `Result<'T,'E>` with `result { }` computation expression, or
Railway-oriented programming style.

**Claude confidence.** Decent but noticeably thinner corpus than C#. Claude occasionally
drifts toward C#-isms (mutable variables, class hierarchies) inside F# files.

**Existing precedent.** The traf compiler (Triton-A → x86) is F# — its architecture
and design decisions are available at [github.com/sqrt57/traf](https://github.com/sqrt57/traf).
This is the strongest argument for F#: there is already a working F# compiler for
a closely related language to reference.

**Verdict.** Best language-level fit. The traf reference tips this above C# if you want
to start from a working blueprint.

---

### Go

**AST.** No sum types — AST nodes are `interface{}`-typed with type assertions or
an explicit `Kind` field. Claude generates verbose `switch node.(type)` dispatch.
Not a dealbreaker but adds boilerplate in the codegen pass.

**Binary I/O.** `encoding/binary` + `bytes.Buffer`. Clean and idiomatic.

**Error handling.** `(result, error)` multiple returns — maps well to Pint's own
multiple-return design. Verbose but predictable.

**Claude confidence.** Very high. Go is one of Claude's strongest languages.

**Friction.** No exhaustiveness checking on type switches. Missing a case in the
codegen pass is a silent bug, not a compile error.

**Verdict.** Fast to start, gets verbose on a type-heavy compiler pass. Acceptable
choice if you know Go well; not the strongest pick otherwise.

---

### Zig

**AST.** Tagged unions (sum types) — very clean, exhaustiveness-checked.

**Binary I/O.** Just write bytes — `std.io.Writer`, natural for low-level work.
`extern struct` maps directly to PE header layouts.

**Error handling.** `!T` return type with `try` propagation — nearly identical to
Pint's own design philosophy. There is a certain symmetry to implementing Pint in Zig.

**Comptime.** Powerful but complex for Claude — training data is thinner and
comptime errors are harder to diagnose in agentic loops.

**Claude confidence.** Noticeably lower than Rust, C#, or Go. Claude's Zig is correct
in the simple case but less confident on lifetime/allocator patterns.

**Verdict.** Philosophically the most aligned with Pint's design (no hidden allocation,
explicit everything). Riskier due to weaker Claude support. Better as a rewrite
target once the compiler is working in another language.

---

### OCaml

**AST.** Algebraic types — equivalent to F# in quality. Pattern matching is exhaustive.

**Claude confidence.** Lower than F# or C#. Training data exists but is thinner;
`dune` build system is less familiar to Claude than `cargo` or `dotnet`.

**Verdict.** The canonical academic compiler language. Strong fit but weaker Claude
synergy. Pass unless you already know OCaml well.

---

### Haskell

**AST.** Algebraic types, lazy evaluation. Transformation pipelines are natural.

**Friction.** Laziness makes allocation and performance reasoning hard.
Type errors from deeply composed monadic code are difficult to act on in an agentic loop.

**Verdict.** Theoretically elegant but the worst "speed of implementation" profile
of the serious candidates. Not recommended here.

---

### C

**AST.** Tagged union structs — works but requires manual tagging, no exhaustiveness.
Lots of boilerplate for every node type.

**Binary I/O.** `fwrite`, `memcpy` into a buffer — as low-level as you want.

**Claude confidence.** Very high. But runtime bugs (null dereferences, off-by-one in
buffer writes) are silent and surface late. Feedback loop is noisier than typed languages.

**Verdict.** Maximum control, minimum guardrails. Reasonable if the goal is a tiny,
dependency-free compiler binary. Slower development pace than any typed candidate.

---

## Summary

| Language | AST quality | Binary I/O | Error threading | Claude speed | Verdict |
|---|---|---|---|---|---|
| **C#** | ★★★★☆ | ★★★★★ | ★★★☆☆ | ★★★★★ | Fastest implementation |
| **F#** | ★★★★★ | ★★★★★ | ★★★★☆ | ★★★★☆ | Best fit + traf reference |
| **Rust** | ★★★★★ | ★★★★★ | ★★★★★ | ★★★★★ | Best long-term, some tree friction |
| **Go** | ★★★☆☆ | ★★★★☆ | ★★★★☆ | ★★★★★ | Fast start, verbose compiler |
| **Zig** | ★★★★★ | ★★★★★ | ★★★★☆ | ★★★☆☆ | Aligned, weaker Claude support |
| **OCaml** | ★★★★★ | ★★★★☆ | ★★★★☆ | ★★★☆☆ | Great fit, thin Claude corpus |
| **C** | ★★★☆☆ | ★★★★★ | ★★☆☆☆ | ★★★★☆ | Control, no safety net |
| **Haskell** | ★★★★★ | ★★★☆☆ | ★★★☆☆ | ★★★☆☆ | Theoretical fit, impractical here |

---

## Recommendation

**F# or C#**, with F# slightly ahead if the traf codebase will be consulted
as a reference.

- F# gives better language ergonomics (DUs, exhaustive match, pipeline style) and
  the traf compiler is a working blueprint for a nearly identical problem.
- C# gives marginally faster Claude iteration and slightly more idiomatic binary I/O.
  If traf is not a reference point, C# closes the gap.

**Rust** is the right choice if correctness and long-term maintainability outweigh
initial velocity — the borrow checker pays dividends as the codebase grows. It
remains the general top pick from the reference article, and the "no libraries"
constraint doesn't hurt Rust much since its standard library is excellent.

**Avoid Haskell, Zig, and OCaml** unless you have prior experience — all three
have weaker Claude support, which is the dominant cost driver here.
