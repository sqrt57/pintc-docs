# Pint Language Spec v0.5

## Philosophy

Explicit over implicit. Predictable over convenient. The programmer is in control.
No hidden allocations. No runtime. No magic. Every byte accounted for.
Designed for OS kernels, embedded firmware, and low-level systems work.

LLM-friendly by design: the spec is small enough to fit in a context window, and
the language consistent enough that correct code can be generated from it with
minimal examples. Special cases, implicit rules, and magic behavior are the enemy
of both human readers and code-generating models.

Pint is best read as: **Pascal type system + C block structure + Zig/Odin philosophy**.

```
Dominant influences:
  Pascal/Delphi  — ^T  p^  @x  :=  var/const/type/record  module/export  and/or/not/xor
  C              — {}  //  ;  if()/for()/while()  ->  sizeof  flat enums
  Zig/Odin       — explicit philosophy  multiple returns  loop{}  named params  [noreturn]  _
```

---

## Profile

| Decision | Choice |
|---|---|
| Primary goal | Systems / OS / embedded |
| Memory model | Manual |
| Type system | Strong static, fully explicit |
| Syntax | C-family curly braces, Pascal-style pointers |
| Error handling | Multiple return values |
| Concurrency | None built-in |
| Abstraction | Records only |
| Metaprogramming | None |
| Generics | None — monomorphic |
| Targets | Win32 native binary (x86) |
| C interop | FFI layer |
| Implicit conversions | None |
| Bounds checking | None |
| Compiler pipeline | Multi-stage, collection pass before reference resolution |
| Forward declarations | Not required |
| Source encoding | UTF-8; identifiers are ASCII |

---

## Notation Differences from C

Pint looks C-adjacent but diverges at almost every low-level notation. Key differences for readers familiar with C:

| C | Pint |
|---|---|
| Type before name: `int x = 42` | Name before type: `var x: i32 := 42` |
| `=` assignment | `:=` assignment |
| `*T` pointer, `*p` deref, `&x` address-of | `^T` pointer, `p^` deref, `@x` address-of |
| `&&`, `\|\|`, `!` logical operators | `and`, `or`, `not` — `bool` only; short-circuit |
| `&`, `\|`, `~`, `^` bitwise operators | `&`, `\|`, `~` bitwise; `xor` keyword (no `^`) |
| Preprocessor (`#define`, `#include`) | None |
| Implicit integer promotion | No implicit conversions — all casts are explicit |

---

## Comments

```
// Single-line comment

/*
    Multi-line comment
*/
```

---

## Identifiers

Identifiers consist of letters (`a`–`z`, `A`–`Z`), digits (`0`–`9`), and underscores (`_`).
Must start with a letter or underscore.

Valid: `MyType`, `value`, `int32`, `_sys_val`
Invalid: `3d`, `my-type`

---

## Keywords

The following identifiers are reserved and may not be used as user-defined names:

**Control flow:** `if` `else` `for` `while` `loop` `break` `continue` `return`

**Declarations:** `fun` `var` `const` `record` `enum` `type` `module` `import` `export` `extern`

**Literals:** `true` `false` `nil`

**Operators:** `and` `or` `xor` `not`

**Primitive types:** `i8` `i16` `i32` `i64` `u8` `u16` `u32` `u64` `f32` `f64` `bool` `byte` `codepoint` `usize` `isize` `string`

**Discard:** `_`

Builtin identifiers — `sizeof`, `length`, `cast`, `divmod`, `mul`, and the narrowing
intrinsics `to_i8` … `to_u32` — are predefined names, not keywords. They may not be
shadowed by user-defined declarations.

---

## Statement Terminator

Statements are terminated by a semicolon (`;`). Semicolons are required and not
inferred from newlines.

---

## Primitive Types

| Type | Size | Description |
|---|---|---|
| `i8`, `i16`, `i32`, `i64` | 1–8 bytes | Signed integers |
| `u8`, `u16`, `u32`, `u64` | 1–8 bytes | Unsigned integers |
| `f32`, `f64` | 4–8 bytes | IEEE 754 floats |
| `bool` | 1 byte | `true` / `false` |
| `byte` | 1 byte | Alias for `u8` |
| `codepoint` | 4 bytes | Alias for `u32` — a Unicode scalar value |
| `usize` | 4 bytes | Alias for `u32` — unsigned size/count type; architecture-dependent |
| `isize` | 4 bytes | Alias for `i32` — signed size/offset type; architecture-dependent |
| `()` | — | Zero-element tuple; used where no value is produced or expected |

`true` and `false` are keywords and the only literals of type `bool`.

No implicit numeric conversions. All casts are explicit.

---

## Variables

`var` declares a mutable variable; `const` declares a compile-time constant.
`const` is valid at both module scope and function scope; the initializer must
be a compile-time expression in either case.

```
var x: i32 := 42;
const max: usize := 256;        // module scope

fun compute() -> i32 {
    const limit: i32 := 100;    // function scope — also allowed
    var n: i32 := limit * 2;
    return n;
}
```

A `var` declaration without an initializer leaves the variable **undefined** —
reading it before assignment is undefined behavior:

```
var x: i32;   // undefined — do not read before assigning
x := 10;      // now defined
```

### Integer literals

Integer literal types are inferred from context:

Supported formats: decimal (`42`), hexadecimal (`0xff`), binary (`0b1010`).

```
var a: i64  := 42;        // decimal
var b: u8   := 255;       // decimal
var c: i32  := 0xff;      // hexadecimal
var d: u8   := 0b1010;    // binary
```

A literal that does not fit in the inferred type is a compile-time error.
Using an integer literal in a context with no expected type (e.g. `var x := 42;`)
is an error — an explicit type annotation is required.

### Float literals

Float literal types are inferred from context — `f32` or `f64`.

Supported formats:
- Decimal with fractional part: `1.0`, `3.14`, `0.5`
- With exponent (`e` or `E`, optionally signed): `1.5e10`, `2.0E-3`, `6.022e+23`
- Combined: `1.5e-3`

At least one digit is required on each side of the decimal point: `1.0` and `0.5`
are valid; `.5` and `1.` are not.

```
var a: f64 := 3.14;
var b: f32 := 1.0e3;
var c: f64 := 6.022e+23;
```

Using a float literal without an `f32` or `f64` context (e.g. `var x := 1.0;`) is
an error — an explicit type annotation is required. The literal value is converted
to the nearest representable value in the target type at compile time.

---

## Scope

### Block scope

Each `{ }` block introduces a new scope. Variables declared inside a block are not
visible outside it. Loop bodies, `if`/`else` branches, and function bodies are all blocks:

```
{
    var x: i32 := 10;
}
// x is not accessible here
```

Variables declared in the init clause of a `for` belong to the same scope as the
loop body — they are not visible after the loop:

```
for (var i: usize := 0; i < len; i := i + 1) {
    // i is accessible here
}
// i is not accessible here
```

### Sequential visibility

Within any single scope, a name is only usable after its declaration point:

```
fun f() -> () {
    var y: i32 := x;   // error: x not yet declared
    var x: i32 := 10;
}
```

### Shadowing

An inner scope may redeclare a name from an outer scope. The inner declaration
creates a new binding; the outer is unchanged and becomes visible again when the
inner scope exits:

```
var x: i32 := 1;
{
    var x: i32 := 2;   // shadows outer x — both bindings are i32, but independent
    // x == 2 here
}
// x == 1 here
```

### Module-scope pre-collection

At module scope, all type names, function names, and constant names are collected
before any references are resolved. Functions may call each other regardless of
declaration order; mutual recursion requires no forward declarations:

```
fun is_even(n: i32) -> bool {
    if (n == 0) { return true; }
    return is_odd(n - 1);   // valid — is_odd collected before resolution
}

fun is_odd(n: i32) -> bool {
    if (n == 0) { return false; }
    return is_even(n - 1);
}
```

Module-scope `var` initializers are sequential — they may only reference names
declared above them, since initialization order matters at runtime.

---

## Pointers

| Syntax | Meaning |
|---|---|
| `^T` | Pointer type to `T` — raw, nullable, no safety guarantees |
| `^()` | Untyped pointer (like C's `void *`) — can hold any address, cannot be dereferenced without `cast` |
| `@x` | Address of `x` — produces `^T` |
| `p^` | Dereference `p` — produces the `T` value |
| `p.field` | Field access on a record value |
| `p->field` | Field access through pointer — sugar for `p^.field` |

```
var x: i32;
var p: ^i32 := @x;
p^ := 10;
```

The address-of operator applies to variables, constants, functions, string literals,
record fields, and array elements. It always produces a valid runtime pointer:

```
var rec: Point;
var pf: ^f64 := @rec.x;     // address of field — runtime pointer

var arr: [10]i32;
var pe: ^i32 := @arr[3];    // address of element — runtime pointer
```

Whether `@x` is also usable as a compile-time constant depends on whether `x` is at
module scope — see [Compile-Time Expressions](#compile-time-expressions).

Pointer arithmetic is explicit. Adding or subtracting an integer advances by that
many elements — stride equals `sizeof(T)` for a `^T` pointer:

```
var buf: [16]u8;
var p: ^u8  := @buf;
var q: ^u8  := p + 1;    // advance 1 byte

var arr: [8]i32;
var a: ^i32 := @arr;
var b: ^i32 := a + 1;    // advance 4 bytes (sizeof(i32))
var c: ^i32 := a + 3;    // advance 12 bytes
```

No references, no smart pointers, no RAII.

`nil` is the null pointer literal. It is valid wherever a `^T` is expected —
in assignments, comparisons, and return values. The type of `nil` in context is
the expected pointer type; using `nil` without a pointer context is an error.

```
var p: ^i32 := nil;
if (p == nil) { ... }
return nil, Error.NotFound;   // in a function returning (^File, Error)
```

Dereferencing `nil` is undefined behavior.

---

## Arrays

| Syntax | Meaning |
|---|---|
| `[N]T` | Array type — `N` elements of type `T`, `N` known at compile time |
| `arr[i]` | Read element at index `i` |
| `arr[i] := v` | Write element at index `i` |
| `@arr[i]` | Address of element — produces `^T` |

Array literals use square brackets. The element type is inferred from context:

```
var arr: [3]i32 := [1, 2, 3];
```

```
var x: i32 := arr[1];   // read element
arr[0] := 10;           // write element
```

```
var p: ^i32 := @arr[0];
```

No bounds checking. Accessing out-of-range indices is undefined behavior.

---

## Records

The only compound type. No inheritance, no vtables.

```
record Point {
    x: f64,
    y: f64,
}
```

Methods are free functions taking a record pointer — no method syntax sugar:

```
fun rect_area(r: ^Rect) -> f64 {
    return r->width * r->height;   // -> is sugar for ^.
}
```

### Initialization

Record values are initialized with a brace expression. Field names are required;
order is irrelevant. The type is inferred from context:

```
var p: Point := { x: 1.0, y: 2.0 };
var q: Point := { y: 2.0, x: 1.0 };   // same value — order does not matter
```

Nested records use the same syntax recursively:

```
var r: Rect := { origin: { x: 0.0, y: 0.0 }, width: 10.0, height: 5.0 };
```

Omitting a field leaves it undefined — reading it before assignment is undefined
behavior. Using a brace expression without a typed context is an error.

### Passing records

Records are passed by value (copied). Pass a pointer to avoid the copy:

```
fun scale_by_value(p: Point) -> Point {
    return { x: p.x * 2.0, y: p.y * 2.0 };
}

fun scale_in_place(p: ^Point) -> () {
    p->x := p->x * 2.0;
    p->y := p->y * 2.0;
}

var pt: Point := { x: 1.0, y: 2.0 };
pt := scale_by_value(pt);    // pt is copied into the function
scale_in_place(@pt);         // pointer to pt passed — modifies in place
```

### Memory layout control

For interop and systems work, exact layout can be specified. Layout attributes
on a record and its fields can be combined freely:

| Attribute | Applies to | Effect |
|---|---|---|
| `[size=N]` | record | Fix total size to N bytes |
| `[align=N]` | record or field | Require address to be a multiple of N |
| `[offset=N]` | field | Place field at byte offset N from record start |

```
record Sys [size=4] {
    [offset=0] x: i16,
    [offset=2] y: i16,
    [offset=0] value: i32,   // union-like overlap
}

record SimdVec [align=16] {
    x: f32,
    y: f32,
    z: f32,
    w: f32,
}

record Mixed {
    tag: u8,
    [align=4] value: i32,   // field alignment — 3 bytes padding inserted before value
}
```

The compiler enforces `[align=N]` for stack variables and module-scope globals.
For heap-allocated instances, choosing an allocator that satisfies the required
alignment is the programmer's responsibility.

Overlapping fields share the same bytes — writing one affects the other:

```
var s: Sys;
s.x := 1;
s.y := 2;
// s.value now reads both x and y as one i32 (layout is little-endian on x86):
var v: i32 := s.value;   // low 16 bits = 1, high 16 bits = 2

s.value := 0x00030004;
// s.x == 4, s.y == 3
```

`[size]` on the record and `[offset]` on each field are both optional. Without
them the compiler uses C-compatible layout — fields in declaration order with
natural alignment padding. The `[size]`/`[offset]` attributes override that
default for interop or union-like use cases. Specifying a field that extends
past `[size]` is undefined behavior.

### Nominal vs structural typing

Records are nominal — two records with identical fields are distinct incompatible types.
Arrays, pointers, and function types are structural — two definitions with identical
structure are the same type:

```
type A = (i32, ^i32) -> i32;
type B = (i32, ^i32) -> i32;

var x: ^A;
var y: ^B;
x := y;   // valid — A and B are structurally identical function types
```

### Self-referential and mutually recursive records

The compiler collects all top-level type names before resolving any references, so both
self-reference and mutual recursion work without forward declarations:

```
record Node {
    next: ^Node,
    value: i32,
}

record A { b: ^B, }
record B { a: ^A, }
```

---

## Type Aliases

The `type` keyword defines an alias for an existing type. It cannot define a new named
record — use `record Name { ... }` for that:

```
type NodePtr = ^Node;
type Buffer  = [256]byte;
type Handler = (^(), u32) -> bool;
```

---

## Enums

Flat, C-style enums. No attached data:

```
enum Direction { North, South, East, West, }

enum Error {
    None,
    NotFound,
    PermissionDenied,
    OutOfMemory,
}
```

The default underlying type is `i32`. Any signed or unsigned integer type can
be specified explicitly:

```
enum Direction { North, South, East, West, }         // underlying type: i32
enum SmallFlags : u8 { A, B, C, }                    // underlying type: u8
```

Explicit values are supported. Auto-numbering picks `max(0, max(all previous values) + 1)`,
so values never go backwards and duplicates cannot arise from mixing explicit and
auto-numbered members:

```
enum Color { Red = 10, Green, Blue, }     // Green=11, Blue=12

enum Flags : u8 { Read = 4, Write = 2, Exec = 1, }  // all explicit

enum Mixed { A = 5, B, C = 2, D, }       // B=6 (5+1), D=7 (max(5,6,2)+1)
```

Enum values are referenced as `EnumType.Member`. Comparison uses `==` and `!=`:

```
var d: Direction := Direction.North;

if (d == Direction.North) {
    // heading north
}

if (d != Direction.South) {
    // not heading south
}
```

Cast to and from the underlying integer type with `cast`:

```
var i: i32       := cast(i32, d);
var d2: Direction := cast(Direction, 1);  // Direction.South
```

---

## String Literals

`string` is the type of any string literal — a distinct type, not `^byte`.

| Syntax | Meaning |
|---|---|
| `"hello"` | String literal — type `string` |
| `@s` | `^byte` — pointer to first byte of content |
| `length(s)` | `usize` — byte count, excluding null terminator |

| Property | |
|---|---|
| Type | `string` — distinct from `^byte`; not pointer-arithmetic-compatible directly |
| Storage | Read-only data section; null byte appended automatically |
| Encoding | UTF-8; not codepoint-indexed — byte iteration only |

```
const greeting: string := "Hello, world!\n";
var p: ^byte := @greeting;
var len: usize := length(greeting);   // compile-time constant

var q: ^byte  := @"inline";           // address of anonymous string literal
var n: usize  := length("inline");    // 6
```

Escape sequences: `\n`, `\r`, `\t`, `\\`, `\"`, `\0`, `\u{XXXXXX}`.
Source files are UTF-8; literal content may contain UTF-8 directly.
The `\u{XXXXXX}` escape inserts the UTF-8 encoding of a Unicode codepoint (1–4 bytes):

```
const msg: string := "caf\u{E9}";       // "café" — \u{E9} encodes é as 2 bytes
const emoji: string := "\u{1F600}";     // 😀 — 4 bytes
```

Identical string literals are deduplicated in the executable — multiple occurrences of
the same content share a single copy in the read-only data section.

### Byte iteration

Strings are UTF-8 byte arrays. Walk bytes via pointer arithmetic:

```
const greeting: string := "Hello";
var p: ^byte  := @greeting;
var len: usize := length(greeting);
var i: usize  := 0;

loop {
    if (i == len) { break; }
    var b: byte := (p + i)^;
    // process byte b
    i := i + 1;
}
```

Because UTF-8 encodes non-ASCII codepoints as 2–4 bytes, iterating bytes
does not give codepoints directly. Decoding UTF-8 requires inspecting the
high bits of each byte — no built-in decoder is provided.

### Character literals

Character literals have type `codepoint` (`u32`):

```
var a: codepoint := 'A';          // 65
var e: codepoint := '\u{E9}';     // 233 — é
var nl: codepoint := '\n';        // 10
```

To obtain a raw byte value, cast explicitly: `cast(u8, 'A')`.

---

## Functions

Parameters are declared with inline type annotations:

```
fun rect_area(r: ^Rect) -> f64 {
    return r->width * r->height;
}

fun newline() -> () {
}
```

### Return type

The return type follows `->`. Available forms:

| Form | Meaning |
|---|---|
| `-> X` | Single return value of type `X` |
| `-> (X)` | Same — parenthesized single return |
| `-> (X, Y)` | Two return values |
| `-> ()` | No return value (unit) |

Return values may be named; either all are named or none are:

| Form | Meaning |
|---|---|
| `-> (name: X)` | Single named return |
| `-> (name: X, name2: Y)` | Multiple named returns |

Return names are informational — see [Named return values](#named-return-values).

A function with a non-`()` return type that reaches the end without a `return`
produces a compiler warning and undefined behavior at runtime.

`return;` — a return with no value — is valid in `() -> ()` functions and exits
immediately. Falling off the end of a `() -> ()` function is also well-defined;
a bare `return;` is optional.

### Named parameters

At the call site, arguments may be passed by name instead of position.
Named calls are order-independent. Either all arguments are named or all are
positional — mixing is a compile error.

```
fun add(a: i32, b: i32) -> i32 { return a + b; }

var x: i32 := add(1, 2);           // positional
var y: i32 := add(b: 2, a: 1);     // named — order independent; same result
```

The name at the call site must match the parameter name in the function signature.
Named calls require a direct call to a named function. Calls through function
pointers are always positional — function types carry no parameter names.

### Multiple return values

| Use case | Syntax |
|---|---|
| Declare new variables (positional) | `var (f: ^File, e: Error) := g();` |
| Declare new variables, discard one | `var (f: ^File, _) := g();` |
| Assign to existing variables (positional) | `(f, e) := g();` |
| Assign to existing variables (named) | `(file: f, err: e) := g();` |

Multiple return values are the primary error-handling mechanism:

```
fun open_file(path: ^byte, len: usize) -> (^File, Error) {
    if (not fs_exists(path)) {
        return nil, Error.NotFound;
    }

    var file: ^File := ...;
    ...
    return file, Error.None;
}
```

Call site unpacking:

```
const config: string := "config.txt";
var (file: ^File, err: Error) := open_file(@config, length(config));
if (err != Error.None) { ... }

var (file: ^File, _) := open_file(@config, length(config));   // discard error
```

`_` is the discard — a return value unpacked into `_` is dropped and cannot be
referenced. Multiple positions in the same unpacking may use `_`.

No exceptions. No `Result` type. Errors are plain enum values, checked explicitly
at every call site.

Multiple return values must be unpacked before use — they cannot be passed
directly into another call:

```
const config: string := "config.txt";

// error: cannot pass multi-return directly
// use_file(open_file(@config, length(config)));

// correct: unpack first
var (file: ^File, err: Error) := open_file(@config, length(config));
if (err != Error.None) { panic(); }
use_file(file);
```

### Named return values

Return values in a function signature may optionally be named. Either all returns
are named or none are — partial naming is a compile error.

```
fun open_file(path: ^byte, len: usize) -> (file: ^File, err: Error) {
    ...
    return nil, Error.NotFound;
}
```

Return names are informational — the `return` statement remains positional.

Call site unpacking may use named or positional form, but not both. Named unpacking
is only valid when the called function's return list is named; the names in the
unpack must match the return names exactly. `_` is valid in either form.

```
// positional unpack — var form introduces new variables:
var (f: ^File, e: Error) := open_file(@path, len);

// named unpack — parenthesized form assigns to existing variables:
var f: ^File;
var e: Error;
(file: f, err: e) := open_file(@path, len);

// named unpack — order independent; discard one value:
(err: _, file: f) := open_file(@path, len);
```

The parenthesized forms — `(x, y) := g()` and `(name: x, ...) := g()` — always
assign to existing, already-declared variables. To introduce new variables, use
the parenthesized `var` form — see
[Multi-return declaration syntax](#multi-return-declaration-syntax).

### Multi-return declaration syntax

To introduce new variables from a multi-return call with explicit types, use the
parenthesized `var` form:

```
var (file: ^File, err: Error) := open_file(@path, len);
```

Each position carries a name and a type annotation. `_` discards the value at
that position — no variable is introduced and no type annotation is required:

```
var (file: ^File, _) := open_file(@path, len);   // discard second return
```

The parenthesized `var` form is always positional — variable names are chosen by
the caller and are independent of any names in the function's return list.

The untyped form `var file, err := f()` is not valid.

Named unpacking requires a direct call to a function whose declaration names its
returns. Calls through function pointers are always positional — function types
carry no return names.

### Attributes

Attributes are compile-time metadata attached to declarations, written in
square brackets immediately before the declaration they apply to. They affect
code generation, type checking, or linking — never runtime behavior.

`[noreturn]` declares that a function never returns to its caller. The compiler
treats any call to a `[noreturn]` function as a diverging control-flow path,
satisfying return requirements on all branches:

```
[noreturn]
fun panic() -> () {
    exit_process(1);
}

fun unwrap(p: ^i32) -> i32 {
    if (p == nil) { panic(); }   // [noreturn] — branch is covered
    return p^;
}
```

Returning from a `[noreturn]` function is undefined behavior.

### Function pointers

No closures. Address operator works on defined functions:

```
type Callback = (^()) -> ();

var cb: ^Callback := @some_fn;
cb(data);      // implicit dereference — permitted exception to explicit-over-implicit
cb^(data);     // explicit dereference — both are valid
```

A dispatch table is an array of function pointers:

```
type Handler = (i32) -> ();

fun on_north(x: i32) -> () { ... }
fun on_south(x: i32) -> () { ... }
fun on_east(x: i32)  -> () { ... }
fun on_west(x: i32)  -> () { ... }

var handlers: [4]^Handler := [@on_north, @on_south, @on_east, @on_west];

var d: i32 := cast(i32, direction);
handlers[d](value);    // dispatch by index
```

Function types carry no parameter or return names — names belong to function
declarations, not to types. Function types are structural: `(i32, i32) -> i32`
and `(i32, i32) -> i32` are the same type regardless of what any declaration
calls its parameters. Calls through function pointers are always positional:

```
fun open_file(path: ^byte, len: usize) -> (file: ^File, err: Error) { ... }

type OpenFileFn = (^byte, usize) -> (^File, Error);
var fp: ^OpenFileFn := @open_file;

var (f: ^File, e: Error) := fp(@path, len);   // ok — positional
fp(path: @path, len: len);                    // error — no names in function type
(file: f, err: e) := fp(@path, len);          // error — no names in function type
```

---

## Control Flow

Conditions in `if`, `while`, and `for` require parentheses. All block bodies
require braces — braceless single-statement bodies are not permitted.

```
if (x > 0) {
    ...
} else if (x < 0) {
    ...
} else {
    ...
}

for (var i: usize := 0; i < len; i := i + 1) {
    ...
}

while (x > 0) {
    x := x - 1;
    if (x == 5) { continue; }
    if (x == 2) { break; }
}

loop {
    if (done) { break; }
    if (skip) { continue; }
}
```

`break` and `continue` work in all three loop forms. Labels allow breaking or
continuing an outer loop from a nested one:

```
outer: while (cond) {
    for (var i: usize := 0; i < len; i := i + 1) {
        if (done) { break outer; }
        if (skip) { continue outer; }
    }
}
```

A label is an identifier followed by `:`, placed immediately before the loop it names.

No `switch`. No ternary. No exceptions.

---

## Expressions

Operator precedence, highest to lowest:

| Level | Operators | Notes |
|---|---|---|
| Postfix | `a^` `a[i]` `a.b` `a->b` `f(args)` | Dereference, subscript, field access, pointer-field access, call |
| Unary | `-a` `~a` `not a` `@x` | Numeric negation, bitwise NOT, logical NOT, address-of |
| Multiplicative | `*` `/` `%` | |
| Additive | `+` `-` | |
| Bit shift | `<<` `>>` | |
| Comparison | `<` `>` `<=` `>=` | |
| Equality | `==` `!=` | |
| Bitwise AND | `a & b` | Integer types only |
| Xor | `a xor b` | Bitwise (integer) or logical (`bool`) XOR |
| Bitwise OR | `a \| b` | Integer types only |
| Logical AND | `a and b` | `bool` only; short-circuits |
| Logical OR | `a or b` | `bool` only; short-circuits |

`&`, `|`, `~` operate on integer types (bitwise). `and`, `or`, `not` operate on
`bool` (logical). `xor` operates on both — bitwise for integers, logical for booleans.

```
var a: i32  := 0xff & 0x0f;     // bitwise AND → 0x0f
var b: bool := true and false;  // logical AND → false

var c: i32  := ~0xff;           // bitwise NOT → -256 (all bits flipped)
var d: bool := not true;        // logical NOT → false

var e: i32  := 0b1010 xor 0b1100;  // bitwise XOR → 0b0110
var f: bool := true xor false;     // logical XOR → true
```

`and` and `or` short-circuit: the right operand is not evaluated if the result is
already determined by the left. `xor` always evaluates both operands.
`&`, `|`, and `~` on integer types never short-circuit.

Unary `-` is only valid on signed integer and float types. Applying it to an
unsigned type is a compile error:

```
var x: u32 := 10;
var y: u32 := -x;   // error: unary `-` on unsigned type
```

### Shift semantics

`<<` is a logical left shift on all integer types: vacated bits are filled with
zero.

`>>` behavior depends on the operand type:

- **Unsigned types** (`u8`, `u16`, `u32`, `u64`, `usize`, `byte`, `codepoint`):
  logical right shift — vacated bits filled with zero (SHR).
- **Signed types** (`i8`, `i16`, `i32`, `i64`, `isize`): arithmetic right shift —
  vacated bits filled with the sign bit (SAR).

```
var a: u32 := 0x80000000 >> 1;                       // logical:    0x40000000
var b: i32 := cast(i32, 0x80000000) >> 1;            // arithmetic: 0xC0000000 (-1073741824)
```

The shift amount must be in the range `[0, bit_width − 1]`. A shift amount
outside this range (e.g. `x >> 32` for a 32-bit type) is undefined behavior.
A compile-time-constant shift amount outside the valid range is a compile error.

### Integer overflow semantics

Integer arithmetic wraps silently on overflow using two's complement modular
arithmetic. This applies to both signed and unsigned types, and to `+`, `-`,
`*`, and unary `-`.

```
var a: u8  := 255;
var b: u8  := a + 1;    // wraps → 0

var c: i8  := 127;
var d: i8  := c + 1;    // wraps → -128

var e: i8  := -128;
var f: i8  := -e;       // wraps → -128 (negation of MIN wraps)
```

There is no implicit overflow trap. Code that needs to detect overflow must use
the narrowing intrinsics (`to_i8`, `to_u8`, …) or check bounds explicitly before
the operation.

---

## Type Casts

All explicit with `cast(T, x)`:

```
var x: i32 := 42;
var y: i64 := cast(i64, x);    // numeric widening
var z: i16 := cast(i16, x);    // numeric narrowing — no overflow check
var u: u32 := cast(u32, x);    // signed → unsigned

var p: ^i32 := @x;
var q: ^()  := cast(^(), p);   // pointer → void pointer
var r: ^i32 := cast(^i32, q);  // void pointer → typed pointer
```

Enums can be cast to and from their underlying integer type:

```
var d: Direction := Direction.North;
var i: i32       := cast(i32, d);         // enum → int: always valid
var d2: Direction := cast(Direction, i);  // int → enum: no range check
```

Casting an integer value that does not correspond to any enum member is
undefined behavior.

Narrowing with overflow detection via builtin intrinsics:

```
var y, overflow := to_i16(x);    // (i16, bool)
var y, overflow := to_u8(x);     // (u8, bool)
```

Available narrowing intrinsics: `to_i8`, `to_u8`, `to_i16`, `to_u16`, `to_i32`, `to_u32`.

---

## Compile-Time Expressions

Compile-time expressions are valid as initializers for `const` and `var` declarations
at module scope, and as array size operands.

**Scalar compile-time expressions:**
- Integer literals: `15`, `0xff`, `0b1010`
- Character literals: `'A'`, `'\n'`, `'\u{E9}'` — type `codepoint`
- Any module-scope constant (order of declaration does not matter)
- `sizeof(T)` and `length(x)`
- Arithmetic on the above: `c * 5 + 8`, `c & 0xff`

**Pointer compile-time expressions** — `@x` where `x` is a module-scope declaration:
- A module-scope variable or constant
- A function
- A string literal
- A field of a module-scope record: `@g.field`
- An element of a module-scope array: `@arr[i]` where `i` is a compile-time integer

`@` on a local variable produces a runtime pointer and is never a compile-time expression.

```
const c: i32 := 10;
var x: i32 := c * 5 + 8;
const mask: i32 := c & 0xff;

const msg: string := "hello";
const p: ^byte := @msg;             // address of string literal — compile-time

var buf: [16]byte;                  // module-scope array
const mid: ^byte := @buf[8];        // address of module-scope element — compile-time
```

Runtime expressions are not valid as initializers.

---

## Memory Management

No allocator built into the language. Use WinAPI directly via FFI:

```
[dll_import(dll="kernel32.dll", entry_point="HeapAlloc")]
extern fun heap_alloc(heap: ^(), flags: u32, size: u32) -> ^();

[dll_import(dll="kernel32.dll", entry_point="HeapFree")]
extern fun heap_free(heap: ^(), flags: u32, ptr: ^()) -> bool;

[dll_import(dll="kernel32.dll", entry_point="GetProcessHeap")]
extern fun get_process_heap() -> ^();
```

No GC. No destructor calls. No RAII. You free what you allocate.

---

## Module System

Each source file contains one or more named module definitions. Modules have explicit
export and import lists:

```
module X123 {
    record Node {
        value: i32,
        tag:   i32,
    }

    const answer: i32 := 42;
    var count: i32 := 15;

    fun hello(name: ^byte, len: usize) -> () {
    }

    export hello, answer, Node;
}

module Y {
    import X123 as X;

    fun greet() -> () {
        X.hello(@"Johnny", length("Johnny"));
    }
}
```

- Symbols not in `export` are private to the module.
- `import M as alias` makes symbols available as `alias.symbol` only — unqualified access is not permitted.

### Selective record field export

Record field visibility is controlled per export entry:

```
export Node(tag);    // export Node type; only 'tag' field visible to importers
export Node(*);      // export Node type with all fields visible
export Node;         // export Node type; no fields visible (opaque)
```

A type exported without field visibility is opaque to importers — they can hold
pointers to it and pass it between functions, but cannot read or write its fields.

Enums are exported by name — all variants are visible to importers:

```
export Direction;    // all variants (North, South, ...) visible to importers
```

### Multi-file modules

A module is identified by its name and the file path it is declared in.
Two files can declare modules with the same name — they are distinct modules.
The compiler takes a list of source files; all files are compiled together.

```
// math/vec.pnt
module Vec {
    record V2 { x: f64, y: f64, }
    fun add(a: V2, b: V2) -> V2 {
        return { x: a.x + b.x, y: a.y + b.y };
    }
    export add, V2(*);
}

// main.pnt
module Main {
    import Vec as V;

    [win32_entry]
    fun main() -> () {
        var a: V.V2 := { x: 1.0, y: 2.0 };
        var b: V.V2 := { x: 3.0, y: 4.0 };
        var c: V.V2 := V.add(a, b);
        exit_process(0);
    }
}
```

---

## Entry Point

```
[dll_import(dll="kernel32.dll", entry_point="ExitProcess")]
[noreturn]
extern fun exit_process(code: u32) -> ();

[win32_entry]
fun main() -> () {
    // program logic
    exit_process(0);
}
```

`[win32_entry]` is an attribute designating the Win32 entry point. Returning from the
entry function is undefined behavior — the developer must call `ExitProcess`
explicitly before returning. A `[win32_entry]` function that is not also marked
`[noreturn]` is a compiler error.

---

## FFI / C Interop

### extern declarations

`extern` is a keyword that marks a function declaration as externally defined —
the compiler emits no code for it, and the body is omitted. The declaration ends
with `;` instead of a block.

Attributes are written before `extern`; `extern` appears immediately before `fun`:

```
[dll_import(dll="kernel32.dll", entry_point="GetLastError")]
extern fun get_last_error() -> u32;
```

Multiple attributes stack in any order before `extern`:

```
[dll_import(dll="kernel32.dll", entry_point="ExitProcess")]
[noreturn]
extern fun exit_process(code: u32) -> ();
```

`extern` requires a `[dll_import]` attribute — a bare `extern fun` with no import
attribute is an error. Conversely, `[dll_import]` requires `extern`; a function
with a `[dll_import]` attribute must not have a body.

### Calling conventions

Two conventions are relevant on x86 Windows:

| Convention | Caller/callee cleans stack | Used by |
|---|---|---|
| `stdcall` | Callee | Win32 API (kernel32, user32, etc.) |
| `cdecl` | Caller | C runtime; variadic functions |

The default convention for both `[dll_import]` and `[dll_export]` is `stdcall`.
Override by passing `conv` as a parameter: `[dll_import(dll="...", entry_point="...", conv=cdecl)]`
or `[dll_export(conv=cdecl)]`.

### Importing DLL symbols

```
[dll_import(dll="kernel32.dll", entry_point="GetLastError")]
extern fun get_last_error() -> u32;

[dll_import(dll="kernel32.dll", entry_point=5)]
extern fun some_ordinal() -> u32;

[dll_import(dll="msvcrt.dll", entry_point="printf", conv=cdecl)]
extern fun printf(fmt: ^byte) -> i32;
```

### Exporting symbols

The compiler can produce an EXE or a DLL — selected by a compiler flag.
`[dll_export]` is an attribute that adds a function to the binary's export table;
it is only meaningful when building a DLL.

```
[dll_export]
fun my_add(a: i32, b: i32) -> i32 {
    return a + b;
}

[dll_export(conv=stdcall)]
fun my_stdcall_fn(x: i32) -> i32 {
    return x;
}
```

Note: module-level `export` (inside a `module` block) controls Pint-internal
visibility only — it does not affect the binary export table.

---

## Builtins

| Builtin | Description |
|---|---|
| `sizeof(T)` | `usize` — size of type in bytes; also accepts a variable or constant name |
| `length(x)` | `usize` — element count of outermost array dimension, or byte count of `string` literal (excluding null terminator) |
| `cast(T, x)` | Explicit type cast (numeric conversions, pointer ↔ pointer, pointer ↔ integer) |
| `divmod(a, b)` | `(T, T)` — quotient and remainder; operands must be the same integer type |
| `mul(a, b)` | `(T, T)` — full-width multiply; operands must be the same unsigned integer type |
| `to_i8(x)` … `to_u32(x)` | Narrowing integer cast → `(value, overflow: bool)` |

### sizeof

`sizeof(T)` returns the size of type `T` in bytes as a `usize` compile-time constant.
`sizeof(x)` where `x` is a variable or constant is equivalent to `sizeof(T)` for the
declared type `T` of `x` — it is also a compile-time constant.

```
var n: i32;
const s: usize := sizeof(i32);    // 4
const t: usize := sizeof(n);      // 4 — same as sizeof(i32)
```

### divmod

`divmod(a, b)` performs integer division and remainder in a single operation.
Both operands must have the same integer type `T` (signed or unsigned); the result
is `(T, T)` — quotient first, remainder second. Division truncates toward zero,
matching the behaviour of `/` and `%`. Division by zero is undefined behavior.

```
var q, r := divmod(17, 5);     // q=3, r=2
var q, r := divmod(-7, 2);     // q=-3, r=-1  (truncation toward zero)
```

### mul

`mul(a, b)` computes the full-width product of two values of the same unsigned
integer type `T`, returning `(T, T)` — the high half first, the low half second.

| Operand type | Result | Product width |
|---|---|---|
| `u8` | `(u8, u8)` | 16 bits |
| `u16` | `(u16, u16)` | 32 bits |
| `u32` | `(u32, u32)` | 64 bits |
| `u64` | `(u64, u64)` | 128 bits |

Signed operands are not accepted; cast to unsigned first.

```
var hi, lo := mul(cast(u32, a), cast(u32, b));  // 32×32→64 split into two u32
```

---

## Complete Example

A two-module Win32 program. Covers the module system, FFI bindings, records,
multi-return error handling, and a loop — all assembled together.

```
// win32-hello.pnt — "Hello, World" for Win32.

module Console {

    [dll_import(dll="kernel32.dll", entry_point="GetStdHandle")]
    extern fun get_std_handle(n_std_handle: i32) -> ^();

    [dll_import(dll="kernel32.dll", entry_point="WriteFile")]
    extern fun write_file(
        h_file:           ^(),
        lp_buffer:        ^byte,
        n_bytes_to_write: u32,
        lp_bytes_written: ^u32,
        lp_overlapped:    ^()
    ) -> bool;

    const STD_OUTPUT_HANDLE: i32 := -11;

    record Buf {
        ptr: ^byte,
        len: u32,
    }

    fun write_buf(handle: ^(), b: Buf) -> (ok: bool, written: u32) {
        var n: u32;
        var success: bool := write_file(handle, b.ptr, b.len, @n, nil);
        return success, n;
    }

    export get_std_handle, write_buf, STD_OUTPUT_HANDLE, Buf(*);
}

module Main {
    import Console as Con;

    [dll_import(dll="kernel32.dll", entry_point="ExitProcess")]
    [noreturn]
    extern fun exit_process(exit_code: u32) -> ();

    [win32_entry]
    [noreturn]
    fun main() -> () {
        var stdout: ^() := Con.get_std_handle(Con.STD_OUTPUT_HANDLE);

        const greeting: string := "Hello, Pint!\n";
        var buf: Con.Buf := { ptr: @greeting, len: cast(u32, length(greeting)) };

        var i: usize := 0;
        while (i < 3) {
            var (ok: bool, _) := Con.write_buf(stdout, buf);
            if (not ok) { exit_process(1); }
            i := i + 1;
        }

        exit_process(0);
    }
}
```

---

## Excluded

| Feature | Rationale |
|---|---|
| Generics / comptime | Complexity, compile time cost |
| Exceptions | Unpredictable control flow, hidden overhead |
| Garbage collector | Unacceptable for OS / embedded targets |
| Inheritance | Complexity without payoff |
| Operator overloading | Readability — operations should be explicit |
| Implicit conversions | Source of subtle bugs |
| Forward declarations | Unnecessary with a collection pass; adds surface area |
| Ownership and borrowing | Complexity; manual memory is the model |
| Sum types / tagged unions | Not needed without pattern matching |
| Pattern matching | Excluded with sum types |
| Closures | No captures; function pointers cover the use cases |
| Error unions (`!T`) | Multiple return values serve the same purpose |
