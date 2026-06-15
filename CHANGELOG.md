# Pintc Changelog

Tracks language spec versions and compiler releases.

## Compiler

#### Slice 15 — Multiple return values

- `(T1, T2)` tuple return types; functions return multiple values via a hidden pointer convention
- `var (a: T1, b: T2) = f(...)` — declares new typed variables bound to all return values; return buffer pre-allocated in the caller's frame
- `(a, b) = f(...)` — assigns to existing locals; temporary buffer allocated on the stack, values popped into their slots on return
- `_` discard position in both forms silently skips that return value
- Calling convention: hidden pointer pushed last before `call` (callee sees it at `[EBP+8]`); user args at `[EBP+12]`, `[EBP+16]`, …; callee writes `[ptr+0]`, `[ptr+4]`, …; all calls are cdecl (caller cleans up)
- `X86.LeaEaxEsp`, `X86.MovEcxEbpDisp8`, `X86.MovEaxEspDisp8` — new helpers
- 170 tests (140 unit, 4 integration, 26 e2e)

#### Slice 14 — Strings and char literals

- `'A'` char literals — type `byte`; `DecodeCharLit` handles `\n`, `\t`, `\\`, `\'`, `\0`, and other escapes
- `"hello"` string literals — decoded as UTF-8 bytes (no null terminator in the `StringLiteralExpr`); `\u{XXXXXX}` Unicode codepoint escapes supported
- String literals allocated into a `.rdata` PE section (null-terminated); `.ptr` yields the section VA, `.len` yields the byte count — both emitted as compile-time immediates
- `const s: string = "hello";` at function scope: `StringConstExpr` recorded in `FunCtx.Consts`; `s.ptr` / `s.len` field access paths resolved at codegen
- Module-scope string consts: `EvalConstExpr` allocates bytes into `rdataBytes` and returns a `StringConstExpr`
- `^byte` dereference: `mov al,[eax]; movzx eax,al` instead of `mov eax,[eax]` — necessary for byte-sized string indexing (`p^`)
- Pointer arithmetic uses `Stride()` (actual memory width: 1 for `byte/u8/i8/bool`, 2 for `u16/i16`, 4 otherwise) instead of `StackSlotSize()` (always 4); `GetExprType` extended to propagate type through `Add`/`Sub`
- `.rdata` section at RVA 0x2000; `.data` shifts to 0x3000 when rdata is present; `dataRva` computed dynamically at codegen and matched in PeWriter
- `X86.MovAlMemEax()` — new helper: `8A 00`
- 167 tests (140 unit, 4 integration, 23 e2e)

#### Slice 13 — Module constants

- `const NAME: T = expr;` at module scope
- Compile-time expression evaluator: literals, const-of-consts, unary (`-`, `~`, `not`), binary arithmetic/bitwise/comparison — all evaluated to a literal at compile time
- Forward references work; evaluation is memoized to handle out-of-order declarations
- Evaluated values seeded into `FunCtx.Consts` at function emit time; uses inside functions inline the literal (same mechanism as local literal consts from Slice 12)
- Local const names shadow module const names within a function body
- 163 tests (140 unit, 4 integration, 19 e2e)

#### Slice 12 — Local constants

- `const NAME: T = expr;` declarations inside function bodies
- Literal initializers (`true`, `false`, integer literals) inlined at every use site — no stack allocation
- Non-literal initializers (calls, expressions with side effects) allocated a stack slot and evaluated once at the declaration site; subsequent uses read the slot
- 162 tests (140 unit, 4 integration, 18 e2e)

#### Slice 11 — Modules

- Multiple modules per source file; multiple source files per compilation (`ParseProgram` loops `ParseModule` until EOF; `Program.cs` accumulates all modules across files)
- `import Mod as Alias;` — makes a module's exported names callable as `Alias.fn(...)` in the current module
- `export name;` — marks a function as visible to importing modules
- `return expr;` statement in non-entry functions; frame teardown before `ret`
- Cross-module `CallExpr`: `Alias.fn(args)` resolved via alias map at codegen; emits `call rel32` with backpatch
- `[dll_import]` externs in expression position (`var x = f(a, b)`) — `EmitCallExpr` routes these through the IAT instead of `call rel32`
- `--dll` compiler flag — produces a PE32 DLL instead of an EXE:
  - `[dll_export]` attribute on functions — exported with stdcall convention (`ret N`)
  - `.edata` section with `IMAGE_EXPORT_DIRECTORY`; name pointer table sorted for binary search
  - COFF Characteristics `0x2102` (adds `IMAGE_FILE_DLL`); `AddressOfEntryPoint = 0` (no DllMain)
- 158 tests (138 unit, 4 integration, 16 e2e)

#### Slice 10 — Pointers

- `^T` pointer types; all pointers are 4 bytes (one stack slot) on IA-32
- Address-of: `@var`, `@arr`, `@arr[i]`, `@rec.field` — pushes the stack address via `LEA eax, [ebp+disp8]`
- Dereference read: `p^` as an expression — loads `[eax]` after popping the pointer into EAX
- Dereference write: `p^ = expr;` statement — pops pointer into ECX, value into EAX, stores via `MOV [ecx], eax`
- Arrow read: `p->field` as an expression — loads `[eax + fieldByteOffset]`
- Arrow write: `p->field = expr;` statement — stores via `MOV [ecx + fieldByteOffset], eax`
- Pointer arithmetic: `p + n` / `p - n` where `p` is a pointer — scales `n` by `sizeof(T)` using `IMUL ecx, ecx, stride` before adding; stride determined from the declared pointer type at codegen time
- 155 tests (138 unit, 4 integration, 13 e2e)

#### Slice 9 — Records

- `record Name { field: Type; ... }` declarations at module scope
- Local variables of record type; stack frame allocates the sum of all field sizes
- Field assignment: `r.field = expr;`
- Field read: `r.field` as an expression; works in conditions, initialisers, and arithmetic
- Nested records: `r.inner.field` resolves field paths of arbitrary depth; byte offset is the sum of preceding fields' sizes at each level
- Codegen: frame-relative addressing `[ebp + localOffset + fieldByteOffset]` for both reads and writes
- 154 tests (138 unit, 4 integration, 12 e2e)

#### Slice 8 — Arrays

- `[N]T` array types; stack frame allocates N × 4 bytes per array variable
- Index read: `a[i]` as an expression; SIB addressing `[ebp + ecx*4 + disp8]`
- Index assignment: `a[i] = v;` statement; same SIB encoding for the store
- Constant and variable indices both work; no bounds checking (out-of-bounds is undefined behavior)
- 153 tests (138 unit, 4 integration, 11 e2e)

#### Slice 7 — for

- `for (var i: T = init; cond; post = expr) { }` loop; loop variable scoped to the loop block
- `break` and `continue` work inside `for`; `continue` executes the post-step before rechecking the condition (unlike `while` where it jumps straight to the condition)
- Nested `for` loops; `break`/`continue` target the innermost enclosing loop
- Codegen: init emit → condition check (`jz rel32` exit) → body → post-step → back-edge `jmp rel32`; continue patches land at post-step
- 152 tests (138 unit, 4 integration, 10 e2e)

#### Slice 6 — while, loop, break, continue

- `while (cond) { }` loop with condition re-evaluated each iteration
- `loop { }` infinite loop
- `break` and `continue` inside both loop forms; correct backpatching through nested `if`/`else`
- Local variable assignment: `name = expr;`
- Codegen: loop-top label + `jz rel32` exit (while) / unconditional back-edge `jmp rel32`; break/continue patch lists propagated through if/else
- 151 tests (138 unit, 4 integration, 9 e2e)

#### Slice 5 — if/else

- `if (cond) { }` and `if (cond) { } else { }` statements
- `else if` chains; arbitrarily nested if/else
- Codegen: conditional jump (`jz rel32`) with backpatching; unconditional (`jmp rel32`) for else skip
- 143 tests (131 unit, 4 integration, 8 e2e)

#### Slice 4 — Expressions

- Binary operators: arithmetic (`+`, `-`, `*`, `/`, `%`), bitwise (`&`, `|`, `xor`), shift (`<<`, `>>`), comparison (`<`, `>`, `<=`, `>=`, `==`, `!=`), logical (`and`, `or`)
- Unary operators: `-` (negation), `~` (bitwise NOT), `not` (boolean NOT)
- `true` / `false` bool literals
- Full operator precedence table; parentheses override precedence
- 142 tests (131 unit, 4 integration, 7 e2e)

#### Slice 3 — Local variables

- `var` declarations inside function bodies; stack slot allocation via `[ebp−disp8]`
- 140 tests (131 unit, 4 integration, 5 e2e)

#### Slice 2 — Module variables

- Module-scope `var` for integer and bool types; `.data` section in PE32
- 129 tests (123 unit, 2 integration, 4 e2e)

#### Slice 1 — End-to-end skeleton

- Lex → parse → resolve → type-check → codegen → PE32 emit pipeline
- Win32 EXE output; `[dll_import]` / `[win32_entry]` / `[noreturn]` attributes; stdcall FFI
- 124 tests (119 unit, 2 integration, 3 e2e)

## Language Spec

### [v0.7](pint-spec-v0.7.md)

In progress.

### [v0.6](archive/pint-spec-v0.6.md)

#### Assignment operator

- `:=` replaced by `=` for all assignments and initializers — `var x: i32 = 42`, `x = 10`

#### Spec structure

- Reorganised from 27 flat sections into 8 top-level sections: Overview, Lexical, Types, Declarations, Expressions, Control Flow, Modules & FFI, Reference
- Builtins moved under Expressions; Memory Management moved under Modules & FFI; Scope moved under Declarations
- Philosophy: reordered — language-identity content (influences block) now precedes LLM/tooling notes; removed redundant "No magic" and duplicate "magic behavior"
- Profile: removed Metaprogramming row (implied by Generics + Abstraction); removed Compiler pipeline row (internal detail, not language behavior); Targets row notes Win32 x86 is current implementation only
- Notation Differences from C: pointer row moved first; removed redundant `(no ^)` parenthetical; removed "short-circuit" from `and`/`or`/`not` row (not a divergence from C)

#### Spec clarifications and deduplication

- **Bug fix:** `divmod` and `mul` examples corrected to use the typed `var (q: T, r: T) = ...` form; bare untyped form is not valid
- **Terminology:** "Alias for" → "Same as"; "zero-element tuple" → "unit type"; "opaque type" → "built-in type" (string); `behaviour` → `behavior`
- **Deduplication:** prose that restated table content, repeated nearby rules, or duplicated the Excluded section removed throughout; narrowing intrinsics consolidated to Builtins with cross-references from Type Casts and Keywords
- **Content placement:** Records memory layout default moved before overrides; module-scope `var` sequential rule moved before code example; Entry Point UB sentence replaced by the enforced compiler-error rule
- **Examples:** `p.field` removed from Pointers table (value access, not pointer syntax); Records methods example uses `Point` instead of undefined `Rect`; multi-file module example stripped of forward-reference to `[win32_entry]`; FFI `printf` replaced with `strlen` (non-variadic cdecl)

#### String type

- `string` is now opaque with two read-only fields: `.ptr: ^byte`, `.len: usize`; both compile-time
- `@s` and `length(s)` removed; `length()` is arrays only

#### Character literals and codepoint

- Character literals now have type `byte` (was `codepoint` / `u32`)
- `codepoint` type removed from the language
- `\u{XXXXXX}` escape removed from character literals; valid in string literals only
- `\'` added to string literal escapes; `\"` added to character literal escapes — both delimiters are now valid escapes in both literal kinds

### [v0.5](archive/pint-spec-v0.5.md)

#### Operators

- `>>` on signed integers: arithmetic right shift (sign-extending, SAR); `>>` on unsigned: logical (zero-fill, SHR); `<<` always logical on all types
- Integer overflow: wraps two's complement on both signed and unsigned types; no implicit trap
- Shift amount out of range (≥ bit width): undefined behavior; compile error when statically detectable
- Bitwise/logical operator split: `&`, `|`, `~` for bitwise on integers; `and`, `or`, `not` for logical on `bool`; `xor` remains dual-use (bitwise integer and logical bool) with no symbol equivalent
- `->` operator added: `p->field` is the preferred pointer field access form, sugar for `p^.field`
- `divmod` type rules: same integer type for both operands (signed or unsigned); returns `(T, T)`; truncates toward zero; division by zero is UB
- `mul` type rules: same unsigned integer type for both operands; returns `(T, T)` high/low; table of operand → product width

#### Types and Literals

- Float literals: formats (`1.0`, `1.5e-3`), digit rules (digit required on both sides of `.`), type inference from `f32`/`f64` context
- `sizeof(x)` on variables: equivalent to `sizeof(T)` for the declared type; compile-time constant

#### Functions

- Function signatures: inline parameter types — `fun f(x: T) -> R` replacing `fun f(x) : (T) -> R`; function types in aliases and pointer declarations remain unnamed: `(T) -> R`
- Return type forms documented: `-> X`, `-> (X)`, `-> (X, Y)`, `-> ()`, with optional names on each form
- Early `return;` in `() -> ()` functions: valid, optional; falling off the end is also well-defined
- Named parameters: call site is all-positional or all-named; named calls are order-independent; name must match signature
- Named return values: signature is all-named or all-unnamed; `return` remains positional; named unpack `(name: x, ...) := f()` assigns to existing variables and is order-independent
- Positional parenthesized unpack: `(x, y) := f()` assigns to existing, already-declared variables positionally; parallel to the named form; distinct from `var (x: T, y: U) := f()` which introduces new variables
- Multi-return declaration syntax: `var (file: ^File, err: Error) := f()` introduces new typed variables; `_` discards a position; always positional; bare `var file, err := f()` is not valid
- Names are declaration-level only, not part of function types; function types remain structural and carry no parameter or return names; calls through function pointers are always positional

#### FFI

- `extern` keyword formally described: marks a bodyless function declaration; attributes go before `extern`, which appears immediately before `fun`; `extern` requires `[dll_import]` and vice versa
- Fixed all examples to use `[dll_import(...)] extern fun ...;` order (was `extern [dll_import(...)] fun ...`)

#### Keywords and Syntax

- Reserved keywords section: full list of keywords; primitive type names are keywords; builtin identifiers (`sizeof`, `cast`, etc.) are predefined names, not keywords, and may not be shadowed
- Braces explicitly required for all block bodies — braceless single-statement forms are not permitted

#### Attributes

- `[entry]` renamed to `[win32_entry]`
- `[win32_entry]` without `[noreturn]` is a compiler error

#### Spec additions

- Complete Example section: two-module Win32 program (adapted from triton-a-spec) covering FFI bindings, a record, multi-return error handling, a loop, and entry point — all assembled
- Quick-reference tables added: pointer/address-of disambiguation (`^T`, `@x`, `p^`, `p->field`); `string` type summary (storage, `@s`, `length`, encoding); multi-return unpack forms (declare new vars, assign existing, named, discard)

#### Spec readability

- Pointers and Arrays: replaced type-only code blocks (`^T`, `[N]T`) with quick-reference tables; `^()` moved into Pointers table
- Pointers: redundant prose repeating table content removed
- Arrays: weak prose bridges between table and code examples removed
- String Literals: split single table into Syntax and Property tables; section reordered (syntax table leads, property table follows, escape sequences consolidated); redundant intro paragraph and duplicate bullet list removed
- Functions: misplaced return-semantics paragraphs moved from `### Multi-return declaration syntax` to `### Return type`

### [v0.4](archive/pint-spec-v0.4.md)

#### Types

- `nil` — null pointer literal; type inferred from pointer context; dereferencing is undefined behavior
- `usize` / `isize` — pointer-sized integer types (4 bytes on x86)
- `^()` — untyped pointer (C's `void *`); cannot be dereferenced without `cast`
- `true` / `false` are `bool` literals and keywords

#### Variables

- Uninitialized `var`: reading before assignment is undefined behavior
- Integer literal type inferred from context; literal without expected type is an error
- Integer literal formats: decimal, `0x`, `0b`
- `:=`-only variable introduction — no separate declaration/assignment split

#### Pointers

- Pointer arithmetic: stride equals `sizeof(T)` per element — `p + 1` on `^i32` advances 4 bytes

#### Arrays

- Array literal syntax: `[1, 2, 3]`; element access via subscript: `arr[i]`

#### Records

- Record initialization: brace syntax with named fields; order irrelevant; type inferred from context
- Records pass by value (copied); pass `^Record` to avoid the copy
- `[align=N]` attribute for records and fields

#### Enums

- Underlying type: default `i32`; explicit: `enum E : u8 { ... }`
- Explicit discriminant values; auto-numbering rule: `max(0, max(previous)+1)`
- Cast enum ↔ underlying integer type with `cast`; casting out-of-range integer is undefined behavior

#### Expressions

- `and` / `or` short-circuit when both operands are `bool`; `xor` and bitwise ops never short-circuit
- Unary `-` only valid on signed and float types; applying to unsigned type is a compile error

#### Functions

- `[noreturn]` attribute: call is a diverging control-flow path; returning from `[noreturn]` is undefined behavior; attribute terminology formalized
- `_` is the discard position in multi-return unpacking; multiple `_` in one unpack is allowed
- Missing return in non-`()` function: compiler warning; reaching end is undefined behavior
- Labeled `break` / `continue`: break or continue an outer loop from a nested one
- Conditions in `if`, `while`, `for` require parentheses

#### FFI

- `dll_import` / `dll_export` now use `(key=value)` attribute syntax
- Calling conventions: `stdcall` (default) and `cdecl`; override with `conv=cdecl`
- `[dll_export]` only meaningful for DLL targets; EXE vs DLL is a compiler flag

#### Module system

- Module-scope `const` declarations: order does not matter; forward references work
- Enum export: all variants visible to importers
- Type aliases promoted to top-level section (was nested under Records)

#### Spec additions

- Semicolons required; not inferred from newlines (now explicitly stated)
- LLM-friendly philosophy note
- Examples throughout all sections
- Future Directions section

### [v0.3](archive/pint-spec-v0.3.md)

#### Syntax

- Comment syntax (`//` and `/* */`)
- Identifier rules: ASCII letters, digits, underscores; must start with letter or underscore
- Operator precedence table
- UTF-8 source encoding specified

#### Types

- `codepoint` — 4-byte alias for `u32`; represents a Unicode scalar value

#### Strings

- `string` type: named module-scope constants, null terminator semantics
- Character literals: type `codepoint` (`u32`); `\u{XXXXXX}` Unicode escape

#### Expressions

- Address-of on subexpressions: `@arr[i]`, `@record.field`

#### Compile-time expressions

- Valid as initializers for `const` and module-scope `var`; valid as array size operands
- Scalar: literals, previously defined constants, `sizeof`, `length`, arithmetic on these
- Pointer: `@x` where `x` is a module-scope variable, constant, function, string, or its field/element

#### Module system

- Selective record field export: `export Node(field)`, `export Node(*)`, `export Node`

#### Entry point

- `[entry]` attribute marks the Win32 entry point; `exit_process` pattern; returning from entry is undefined behavior

### [v0.2](archive/pint-spec-v0.2.md)

Merged from v0.1 and triton-a-spec into a single unified draft.
