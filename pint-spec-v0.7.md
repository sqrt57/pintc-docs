# Pint Language Spec v0.6

## Overview

### Philosophy

Explicit over implicit. Predictable over convenient. The programmer is in control.
No hidden allocations. No runtime. Every byte accounted for.
Designed for OS kernels, embedded firmware, and low-level systems work.

Pint is best read as: **Pascal type system + C block structure + Zig/Odin philosophy**.

```
Dominant influences:
  Pascal/Delphi  — ^T  p^  @x  var/const/type/record  module/export  and/or/not/xor
  C              — {}  //  ;  if()/for()/while()  ->  sizeof  flat enums
  Zig/Odin       — explicit philosophy  multiple returns  loop{}  named params  [noreturn]  _
```

LLM-friendly by design: the spec is small enough to fit in a context window, and
the language consistent enough that correct code can be generated from it with
minimal examples. Special cases and implicit rules are the enemy of both human
readers and code-generating models. Formal grammar: [pint-grammar.md](pint-grammar.md).

Error messages are actionable: they name the source location, the rule violated,
and what to do instead. A message that requires reading the spec to act on is a
compiler bug.

### Profile

| Decision | Choice |
|---|---|
| Primary goal | Systems / OS / embedded |
| Memory model | Manual |
| Type system | Strong static, fully explicit |
| Syntax | C-family curly braces, Pascal-style pointers |
| Error handling | Multiple return values |
| Concurrency | None built-in |
| Abstraction | Records only |
| Generics | None — monomorphic |
| Targets | Win32 native binary (x86) — current implementation only |
| C interop | FFI layer |
| Implicit conversions | None |
| Bounds checking | None |
| Forward declarations | Not required |
| Source encoding | UTF-8; identifiers are ASCII |

### Notation Differences from C

Pint looks C-adjacent but diverges significantly at the notation level. Key differences for readers familiar with C:

| C | Pint |
|---|---|
| `*T` pointer, `*p` deref, `&x` address-of | `^T` pointer, `p^` deref, `@x` address-of |
| Type before name: `int x = 42` | Name before type: `var x: i32 = 42` |
| `&&`, `\|\|`, `!` logical operators | `and`, `or`, `not` — `bool` only |
| `&`, `\|`, `~`, `^` bitwise operators | `&`, `\|`, `~` bitwise; `xor` keyword |
| Preprocessor (`#define`, `#include`) | None |
| Implicit integer promotion | No implicit conversions — all casts are explicit |

---

## Lexical

### Comments

```
// Single-line comment

/*
    Multi-line comment
*/
```

### Identifiers

Identifiers consist of letters (`a`–`z`, `A`–`Z`), digits (`0`–`9`), and underscores (`_`).

Valid: `MyType`, `value`, `int32`, `_sys_val`
Invalid: `3d` // leading digit, `my-type` // hyphen not allowed

### Keywords

The following identifiers are reserved:

**Control flow:** `if` `else` `for` `while` `loop` `break` `continue` `return`

**Declarations:** `fun` `var` `const` `record` `enum` `type` `module` `import` `export` `extern`

**Literals:** `true` `false` `nil`

**Operators:** `and` `or` `xor` `not`

**Primitive types:** `i8` `i16` `i32` `i64` `u8` `u16` `u32` `u64` `f32` `f64` `bool` `byte` `usize` `isize` `string`

**Discard:** `_` (reserved identifier, not a keyword)

Builtin identifiers — `sizeof`, `length`, `cast`, `divmod`, `mul`, and the narrowing
intrinsics `to_i8` … `to_u32` — are predefined names, not keywords. They may not be
shadowed by user-defined declarations.

### Statement Terminator

Statements are terminated by `;`; semicolons are never inferred from newlines.

---

## Types

### Primitive Types

| Type | Size | Description |
|---|---|---|
| `i8`, `i16`, `i32`, `i64` | 1–8 bytes | Signed integers |
| `u8`, `u16`, `u32`, `u64` | 1–8 bytes | Unsigned integers |
| `f32`, `f64` | 4–8 bytes | IEEE 754 floats |
| `bool` | 1 byte | `true` / `false` |
| `byte` | 1 byte | Same as `u8` |
| `usize` | 4 bytes | Same as `u32` — unsigned size/count type |
| `isize` | 4 bytes | Same as `i32` — signed size/offset type |
| `()` | — | Unit type; used where no value is produced or expected |

`true` and `false` are keywords and the only literals of type `bool`.

No implicit numeric conversions. All casts are explicit.

### Pointers

| Syntax | Meaning |
|---|---|
| `^T` | Pointer type to `T` — raw, nullable, no safety guarantees |
| `^()` | Untyped pointer (like C's `void *`) — can hold any address, cannot be dereferenced without `cast` |
| `@x` | Address of `x` — produces `^T` |
| `p^` | Dereference `p` — produces the `T` value |
| `p->field` | Field access through pointer — sugar for `p^.field` |

```
var x: i32;
var p: ^i32 = @x;
p^ = 10;
```

The address-of operator applies to variables, constants, functions, record fields,
and array elements. It always produces a valid runtime pointer:

```
var rec: Point;
var pf: ^f64 = @rec.x;     // address of field — runtime pointer

var arr: [10]i32;
var pe: ^i32 = @arr[3];    // address of element — runtime pointer
```

Whether `@x` is also usable as a compile-time constant depends on whether `x` is at
module scope — see [Compile-Time Expressions](#compile-time-expressions).

Pointer arithmetic is explicit. Adding or subtracting an integer advances by that
many elements — stride equals `sizeof(T)` for a `^T` pointer:

```
var buf: [16]u8;
var p: ^u8  = @buf;
var q: ^u8  = p + 1;    // advance 1 byte

var arr: [8]i32;
var a: ^i32 = @arr;
var b: ^i32 = a + 1;    // advance 4 bytes (sizeof(i32))
var c: ^i32 = a + 3;    // advance 12 bytes
```

`nil` is the null pointer literal. In any pointer-typed context — assignments,
comparisons, and return values — it takes the expected pointer type. Using `nil`
outside a pointer-typed context is an error.

```
var p: ^i32 = nil;
if (p == nil) { ... }
return nil, Error.NotFound;   // in a function returning (^File, Error)
```

Dereferencing `nil` is undefined behavior.

### Arrays

| Syntax | Meaning |
|---|---|
| `[N]T` | Array type — `N` elements of type `T`, `N` known at compile time |
| `arr[i]` | Read element at index `i` |
| `arr[i] = v` | Write element at index `i` |
| `@arr[i]` | Address of element — produces `^T` |

Array literals use square brackets. The element type is inferred from context:

```
var arr: [3]i32 = [1, 2, 3];
var x: i32 = arr[1];   // read element
arr[0] = 10;           // write element
```

No bounds checking. Accessing out-of-range indices is undefined behavior.

### Records

The only compound type.

```
record Point {
    x: f64,
    y: f64,
}
```

Methods are free functions taking a record pointer — no method syntax sugar:

```
fun point_scale(p: ^Point, factor: f64) -> () {
    p->x = p->x * factor;
    p->y = p->y * factor;
}
```

#### Initialization

Using a brace expression without a typed context is an error. Field names are required;
order is irrelevant. The type is inferred from context:

```
var p: Point = { x: 1.0, y: 2.0 };
var q: Point = { y: 2.0, x: 1.0 };   // same value — order does not matter
```

Nested records use the same syntax recursively:

```
var r: Rect = { origin: { x: 0.0, y: 0.0 }, width: 10.0, height: 5.0 };
```

Omitting a field leaves it undefined — reading it before assignment is undefined behavior.

#### Passing records

Records are passed by value (copied). Pass a pointer to avoid the copy:

```
fun scale_by_value(p: Point) -> Point {
    return { x: p.x * 2.0, y: p.y * 2.0 };
}

fun scale_in_place(p: ^Point) -> () {
    p->x = p->x * 2.0;
    p->y = p->y * 2.0;
}

var pt: Point = { x: 1.0, y: 2.0 };
pt = scale_by_value(pt);    // pt is copied into the function
scale_in_place(@pt);         // pointer to pt passed — modifies in place
```

#### Memory layout control

The default layout is C-compatible — fields in declaration order with natural alignment
padding. The attributes below override the default for interop or union-like use cases:

| Attribute | Applies to | Effect |
|---|---|---|
| `[size=N]` | record | Fix total size to N bytes; a field extending past `[size]` is undefined behavior |
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
s.x = 1;
s.y = 2;
// s.value now reads both x and y as one i32 (layout is little-endian on x86):
var v: i32 = s.value;   // low 16 bits = 1, high 16 bits = 2

s.value = 0x00030004;
// s.x == 4, s.y == 3
```

#### Nominal vs structural typing

Records are nominal — two records with identical fields are distinct incompatible types.
Arrays, pointers, and function types are structural — two definitions with identical
structure are the same type:

```
var p: ^((i32, ^i32) -> i32);
var q: ^((i32, ^i32) -> i32);
p = q;   // valid — structurally identical function pointer types
```

#### Self-referential and mutually recursive records

The compiler pre-collects all module-scope names before resolving references (see
[Scope](#scope)), so self-reference and mutual recursion work without forward declarations:

```
record Node {
    next: ^Node,
    value: i32,
}

record A { b: ^B, }
record B { a: ^A, }
```

### Type Aliases

The `type` keyword defines an alias for an existing type:

```
type NodePtr = ^Node;
type Buffer  = [256]byte;
type Handler = (^(), u32) -> bool;
```

### Enums

Flat, C-style enums. No attached data:

```
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

Explicit values are supported. Auto-numbering picks `max(0, max(all previous values) + 1)`:

```
enum Color { Red = 10, Green, Blue, }     // Green=11, Blue=12

enum Flags : u8 { Read = 4, Write = 2, Exec = 1, }  // all explicit

enum Mixed { A = 5, B, C = 2, D, }       // B=6 (5+1), D=7 (max(5,6,2)+1)
```

Enum values are referenced as `EnumType.Member`. Comparison uses `==` and `!=`:

```
var d: Direction = Direction.North;
if (d == Direction.North) { ... }
if (d != Direction.South) { ... }
```

Cast to and from the underlying integer type with `cast`:

```
var i: i32        = cast(i32, d);
var d2: Direction = cast(Direction, 1);  // Direction.South
```

### String Literals

`string` is a built-in type with two read-only fields — `.ptr: ^byte` and `.len: usize`.
Literals are stored in the read-only data section, UTF-8 encoded, null-terminated.

| Syntax | Meaning |
|---|---|
| `"hello"` | String literal — type `string` |
| `s.ptr` | `^byte` — pointer to first byte; null-terminated, usable as C `const char*` |
| `s.len` | `usize` — byte count, excluding null terminator |

```
const greeting: string = "Hello, world!\n";
var p: ^byte   = greeting.ptr;
var len: usize = greeting.len;       // compile-time constant
```

Escape sequences: `\n`, `\r`, `\t`, `\\`, `\"`, `\'`, `\0`, `\u{XXXXXX}`.
Source files are UTF-8; literal content may contain UTF-8 directly.
The `\u{XXXXXX}` escape inserts the UTF-8 encoding of a Unicode codepoint (1–4 bytes):

```
const msg: string   = "caf\u{E9}";   // "café" — \u{E9} encodes é as 2 bytes
const emoji: string = "\u{1F600}";   // 😀 — 4 bytes
```

#### Byte iteration

Strings are UTF-8 byte arrays. Walk bytes via pointer arithmetic:

```
const greeting: string = "Hello";
var p: ^byte   = greeting.ptr;
var len: usize = greeting.len;
var i: usize   = 0;

loop {
    if (i == len) { break; }
    var b: byte = (p + i)^;
    // process byte b
    i = i + 1;
}
```

Because UTF-8 encodes non-ASCII characters as 2–4 bytes, iterating bytes
does not give Unicode scalar values directly — no built-in decoder is provided.

#### Character literals

Character literals have type `byte`. Only code points 0x00–0x7F are valid; use
string literals for multi-byte sequences.

```
var a: byte  = 'A';    // 65
var nl: byte = '\n';   // 10
```

Valid escapes: `\n`, `\r`, `\t`, `\\`, `\"`, `\'`, `\0` — same as string literals except
`\u{XXXXXX}` is not valid.

---

## Declarations

### Variables

`var` declares a mutable variable; `const` declares a compile-time constant.

```
var x: i32 = 42;
const max: usize = 256;        // module scope

fun compute() -> i32 {
    const limit: i32 = 100;    // function scope — also allowed
    var n: i32 = limit * 2;
    return n;
}
```

`const` is valid at both module scope and function scope; the initializer must
be a compile-time expression in either case.

A `var` declaration without an initializer leaves the variable **undefined** —
reading it before assignment is undefined behavior:

```
var x: i32;   // undefined — do not read before assigning
x = 10;      // now defined
```

#### Integer literals

Supported formats: decimal (`42`), hexadecimal (`0xff`), binary (`0b1010`).
The type is inferred from context:

```
var a: i64 = 42;        // decimal
var b: u8  = 255;       // decimal
var c: i32 = 0xff;      // hexadecimal
var d: u8  = 0b1010;    // binary
```

A literal that does not fit in the inferred type is a compile-time error.
Using an integer literal without an explicit type annotation (e.g. `var x = 42;`)
is an error.

#### Float literals

Supported formats:
- Decimal with fractional part: `1.0`, `3.14`, `0.5`
- With exponent (`e` or `E`, optionally signed): `1.5e10`, `2.0E-3`, `6.022e+23`
- Combined: `1.5e-3`

At least one digit is required on each side of the decimal point: `1.0` and `0.5`
are valid; `.5` and `1.` are not. The type is inferred from context — `f32` or `f64`:

```
var a: f64 = 3.14;
var b: f32 = 1.0e3;
var c: f64 = 6.022e+23;
```

Using a float literal without an explicit type annotation (e.g. `var x = 1.0;`) is
an error. The literal value is converted to the nearest representable value in the
target type at compile time.

### Scope

#### Block scope

Each `{ }` block introduces a new scope. Variables declared inside a block are not
visible outside it. Loop bodies, `if`/`else` branches, and function bodies are all blocks:

```
{
    var x: i32 = 10;
}
// x is not accessible here
```

The init clause of a `for` belongs to the loop's block scope:

```
for (var i: usize = 0; i < len; i = i + 1) {
    // i is accessible here
}
// i is not accessible here
```

#### Sequential visibility

Within any single scope, a name is only usable after its declaration point:

```
fun f() -> () {
    var y: i32 = x;   // error: x not yet declared
    var x: i32 = 10;
}
```

#### Shadowing

An inner scope may redeclare a name from an outer scope. The inner declaration
creates a new binding; the outer is unchanged and becomes visible again when the
inner scope exits:

```
var x: i32 = 1;
{
    var x: i32 = 2;   // shadows outer x — both bindings are i32, but independent
    // x == 2 here
}
// x == 1 here
```

#### Module-scope pre-collection

At module scope, all type names, function names, and constant names are collected
before any references are resolved; `var` initializers are sequential and may only
reference names declared above them. Functions may call each other in any order:

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

### Functions

Parameters are declared with inline type annotations:

```
fun rect_area(r: ^Rect) -> f64 {
    return r->width * r->height;
}

fun newline() -> () {
}
```

#### Return type

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

A function with a non-`()` return type that reaches the end without a `return`
produces a compiler warning and undefined behavior at runtime.

`return;` is valid in `() -> ()` functions. Falling off the end of a `() -> ()`
function is also well-defined — a bare `return;` is optional.

#### Named parameters

At the call site, arguments may be passed by name instead of position.
Named calls are order-independent. Either all arguments are named or all are
positional — mixing is a compile error.

```
fun add(a: i32, b: i32) -> i32 { return a + b; }

var x: i32 = add(1, 2);           // positional
var y: i32 = add(b: 2, a: 1);     // named — order independent; same result
```

The name at the call site must match the parameter name in the function signature.

#### Multiple return values

| Use case | Syntax |
|---|---|
| Declare new variables (positional) | `var (f: ^File, e: Error) = g();` |
| Declare new variables, discard one | `var (f: ^File, _) = g();` |
| Assign to existing variables (positional) | `(f, e) = g();` |
| Assign to existing variables (named) | `(file: f, err: e) = g();` |

Multiple return values are the primary error-handling mechanism:

```
fun open_file(path: ^byte, len: usize) -> (^File, Error) {
    if (not fs_exists(path)) {
        return nil, Error.NotFound;
    }

    var file: ^File = ...;
    ...
    return file, Error.None;
}
```

Call site unpacking:

```
const config: string = "config.txt";
var (file: ^File, err: Error) = open_file(config.ptr, config.len);
if (err != Error.None) { ... }

var (file: ^File, _) = open_file(config.ptr, config.len);   // discard error
```

`_` is the discard — a return value unpacked into `_` is dropped and cannot be
referenced. Multiple positions in the same unpacking may use `_`.

Multiple return values must be unpacked before use — they cannot be passed
directly into another call:

```
const config: string = "config.txt";

// error: cannot pass multi-return directly
// use_file(open_file(config.ptr, config.len));

// correct: unpack first
var (file: ^File, err: Error) = open_file(config.ptr, config.len);
if (err != Error.None) { panic(); }
use_file(file);
```

#### Named return values

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
var (f: ^File, e: Error) = open_file(@path, len);

// named unpack — parenthesized form assigns to existing variables:
var f: ^File;
var e: Error;
(file: f, err: e) = open_file(@path, len);

// named unpack — order independent; discard one value:
(err: _, file: f) = open_file(@path, len);
```

Named unpacking requires a direct call to a function whose declaration names its
returns. Calls through function pointers are always positional — function types
carry no return names.

#### Multi-return declaration syntax

To introduce new variables from a multi-return call with explicit types, use the
parenthesized `var` form:

```
var (file: ^File, err: Error) = open_file(@path, len);
```

Each position carries a name and a type annotation. `_` discards the value at
that position — no variable is introduced and no type annotation is required:

```
var (file: ^File, _) = open_file(@path, len);   // discard second return
```

The parenthesized `var` form is always positional — variable names are chosen by
the caller and are independent of any names in the function's return list.

The untyped form `var file, err = f()` is not valid.

#### Attributes

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

#### Function pointers

No closures. Address operator works on defined functions.

Calling through a function pointer implicitly dereferences it — the only
permitted exception to the explicit-over-implicit rule:

```
type Callback = (^()) -> ();

var cb: ^Callback = @some_fn;
cb(data);      // implicit dereference
cb^(data);     // explicit dereference — both are valid
```

A dispatch table is an array of function pointers:

```
type Handler = (i32) -> ();

fun on_north(x: i32) -> () { ... }
fun on_south(x: i32) -> () { ... }
fun on_east(x: i32)  -> () { ... }
fun on_west(x: i32)  -> () { ... }

var handlers: [4]^Handler = [@on_north, @on_south, @on_east, @on_west];

var d: i32 = cast(i32, direction);
handlers[d](value);    // dispatch by index
```

Function types carry no parameter or return names — names belong to function
declarations, not to types. Function types are structural: `(i32, i32) -> i32`
and `(i32, i32) -> i32` are the same type regardless of what any declaration
calls its parameters. Calls through function pointers are always positional:

```
fun open_file(path: ^byte, len: usize) -> (file: ^File, err: Error) { ... }

type OpenFileFn = (^byte, usize) -> (^File, Error);
var fp: ^OpenFileFn = @open_file;

var (f: ^File, e: Error) = fp(@path, len);   // ok — positional
fp(path: @path, len: len);                    // error — no names in function type
(file: f, err: e) = fp(@path, len);          // error — no names in function type
```

---

## Expressions

### Operators

Operator precedence, highest to lowest:

| Level | Operators | Notes |
|---|---|---|
| Postfix | `a^` `a[i]` `a.b` `a->b` `f(args)` | |
| Unary | `-a` `~a` `not a` `@x` | |
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

```
var a: i32  = 0xff & 0x0f;     // bitwise AND → 0x0f
var b: bool = true and false;  // logical AND → false

var c: i32  = ~0xff;           // bitwise NOT → -256 (all bits flipped)
var d: bool = not true;        // logical NOT → false

var e: i32  = 0b1010 xor 0b1100;  // bitwise XOR → 0b0110
var f: bool = true xor false;     // logical XOR → true
```

`and` and `or` short-circuit; `xor` always evaluates both operands.

Unary `-` is only valid on signed integer and float types. Applying it to an
unsigned type is a compile error:

```
var x: u32 = 10;
var y: u32 = -x;   // error: unary `-` on unsigned type
```

#### Shift semantics

`<<` is a logical left shift on all integer types.

`>>` behavior depends on the operand type:

- **Unsigned types** (`u8`, `u16`, `u32`, `u64`, `usize`, `byte`): logical right shift (SHR).
- **Signed types** (`i8`, `i16`, `i32`, `i64`, `isize`): arithmetic right shift (SAR).

```
var a: u32 = 0x80000000 >> 1;                       // logical:    0x40000000
var b: i32 = cast(i32, 0x80000000) >> 1;            // arithmetic: 0xC0000000 (-1073741824)
```

A shift amount outside `[0, bit_width − 1]` is undefined behavior; a constant
amount outside this range is a compile error.

#### Integer overflow semantics

Integer arithmetic wraps silently on overflow using two's complement modular
arithmetic. This applies to both signed and unsigned types, and to `+`, `-`,
`*`, and unary `-`.

```
var a: u8 = 255;
var b: u8 = a + 1;    // wraps → 0

var c: i8 = 127;
var d: i8 = c + 1;    // wraps → -128

var e: i8 = -128;
var f: i8 = -e;       // wraps → -128 (negation of MIN wraps)
```

There is no implicit overflow trap. Code that needs to detect overflow must use
the narrowing intrinsics (`to_i8`, `to_u8`, …) or check bounds explicitly before
the operation.

### Type Casts

All explicit with `cast(T, x)`:

```
var x: i32 = 42;
var y: i64 = cast(i64, x);    // numeric widening
var z: i16 = cast(i16, x);    // numeric narrowing — no overflow check
var u: u32 = cast(u32, x);    // signed → unsigned

var p: ^i32 = @x;
var q: ^()  = cast(^(), p);   // pointer → void pointer
var r: ^i32 = cast(^i32, q);  // void pointer → typed pointer
```

Enums can be cast to and from their underlying integer type:

```
var d: Direction  = Direction.North;
var i: i32        = cast(i32, d);         // enum → int: always valid
var d2: Direction = cast(Direction, i);   // int → enum: no range check
```

Casting an integer value that does not correspond to any enum member is
undefined behavior.

Narrowing with overflow detection is available via the `to_i8`…`to_u32` intrinsics —
see [Builtins](#builtins).

### Compile-Time Expressions

Compile-time expressions are valid as initializers for `const` and `var` declarations
at module scope, and as array size operands.

**Scalar compile-time expressions:**
- Integer literals: `15`, `0xff`, `0b1010`
- Character literals: `'A'`, `'\n'` — type `byte`
- Any module-scope constant (order of declaration does not matter)
- `sizeof(T)` and `length(x)`
- `s.len` for any `string` expression `s`
- Arithmetic on the above: `c * 5 + 8`, `c & 0xff`

**Pointer compile-time expressions:**
- `@x` where `x` is a module-scope variable, constant, or function
- `@g.field` — address of a module-scope record field
- `@arr[i]` — address of a module-scope array element; `i` must be a compile-time integer
- `s.ptr` for any `string` expression `s`

`@` on a local variable produces a runtime pointer and is never a compile-time expression.

```
const c: i32 = 10;
var x: i32 = c * 5 + 8;
const mask: i32 = c & 0xff;

const msg: string = "hello";
const p: ^byte = msg.ptr;          // compile-time
const n: usize = msg.len;          // compile-time

var buf: [16]byte;                  // module-scope array
const mid: ^byte = @buf[8];        // address of module-scope element — compile-time
```

### Builtins

Type casts use `cast(T, x)` — see [Type Casts](#type-casts).

| Builtin | Description |
|---|---|
| `sizeof(T)` | `usize` — size of type in bytes; also accepts a variable or constant name |
| `length(x)` | `usize` — element count of outermost array dimension |
| `divmod(a, b)` | `(T, T)` — quotient and remainder; operands must be the same integer type |
| `mul(a, b)` | `(T, T)` — full-width multiply; operands must be the same unsigned integer type |
| `to_i8(x)` … `to_u32(x)` | Narrowing integer cast → `(value, bool)` |

#### sizeof

`sizeof(T)` returns the size of type `T` in bytes as a `usize` compile-time constant.
`sizeof(x)` where `x` is a variable or constant is equivalent to `sizeof(T)` for the
declared type `T` of `x`.

```
var n: i32;
const s: usize = sizeof(i32);    // 4
const t: usize = sizeof(n);      // 4 — same as sizeof(i32)
```

#### divmod

`divmod(a, b)` performs integer division and remainder in a single operation.
Both operands must have the same integer type `T` (signed or unsigned); the result
is `(T, T)` — quotient first, remainder second. Division truncates toward zero,
matching the behavior of `/` and `%`. Division by zero is undefined behavior.

```
var (q: i32, r: i32) = divmod(17, 5);     // q=3, r=2
var (q: i32, r: i32) = divmod(-7, 2);     // q=-3, r=-1  (truncation toward zero)
```

#### mul

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
var (hi: u32, lo: u32) = mul(cast(u32, a), cast(u32, b));  // 32×32→64 split into two u32
```

---

## Control Flow

Conditions require parentheses. Block bodies require braces.

```
if (x > 0) {
    ...
} else if (x < 0) {
    ...
} else {
    ...
}

for (var i: usize = 0; i < len; i = i + 1) {
    ...
}

while (x > 0) {
    x = x - 1;
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
    for (var i: usize = 0; i < len; i = i + 1) {
        if (done) { break outer; }
        if (skip) { continue outer; }
    }
}
```

A label is an identifier followed by `:`, placed immediately before the loop it names.

---

## Modules & FFI

### Module System

A source file defines one or more modules. Modules have explicit export and
import lists:

```
module X123 {
    record Node {
        value: i32,
        tag:   i32,
    }

    const answer: i32 = 42;
    var count: i32 = 15;

    fun hello(name: ^byte, len: usize) -> () {
    }

    export hello, answer, Node;
}

module Y {
    import X123 as X;

    fun greet() -> () {
        X.hello("Johnny".ptr, "Johnny".len);
    }
}
```

- Symbols not in `export` are private to the module.
- `import M as alias` makes symbols available as `alias.symbol` only — unqualified access is not permitted.

#### Selective record field export

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

#### Multi-file modules

A module is identified by its name and the file path it is declared in.
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

    fun main() -> () {
        var a: V.V2 = { x: 1.0, y: 2.0 };
        var b: V.V2 = { x: 3.0, y: 4.0 };
        var c: V.V2 = V.add(a, b);
    }
}
```

### Entry Point

```
[dll_import(dll="kernel32.dll", entry_point="ExitProcess")]
[noreturn]
extern fun exit_process(code: u32) -> ();

[win32_entry]
[noreturn]
fun main() -> () {
    // program logic
    exit_process(0);
}
```

`[win32_entry]` is an attribute designating the Win32 entry point. The entry function
must call `ExitProcess` before returning; a `[win32_entry]` function that is not also
marked `[noreturn]` is a compiler error.

### FFI / C Interop

#### extern declarations

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

`extern` and `[dll_import]` are mutually required — neither is valid without the
other, and a `[dll_import]` function must have no body.

#### Calling conventions

| Convention | Caller/callee cleans stack | Used by |
|---|---|---|
| `stdcall` | Callee | Win32 API (kernel32, user32, etc.) |
| `cdecl` | Caller | C runtime; variadic functions |

The default convention for both `[dll_import]` and `[dll_export]` is `stdcall`.
Override with `conv`: `[dll_import(dll="...", entry_point="...", conv=cdecl)]`
or `[dll_export(conv=cdecl)]`.

#### Importing DLL symbols

```
[dll_import(dll="kernel32.dll", entry_point="GetLastError")]
extern fun get_last_error() -> u32;

[dll_import(dll="kernel32.dll", entry_point=5)]
extern fun some_ordinal() -> u32;

[dll_import(dll="msvcrt.dll", entry_point="strlen", conv=cdecl)]
extern fun strlen(s: ^byte) -> usize;
```

#### Exporting symbols

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

### Memory Management

No allocator built into the language. Use WinAPI directly via FFI:

```
[dll_import(dll="kernel32.dll", entry_point="HeapAlloc")]
extern fun heap_alloc(heap: ^(), flags: u32, size: u32) -> ^();

[dll_import(dll="kernel32.dll", entry_point="HeapFree")]
extern fun heap_free(heap: ^(), flags: u32, ptr: ^()) -> bool;

[dll_import(dll="kernel32.dll", entry_point="GetProcessHeap")]
extern fun get_process_heap() -> ^();
```

---

## Reference

### Complete Example

A two-module Win32 program. Covers the module system, FFI bindings, records,
multi-return error handling, and a loop.

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

    const STD_OUTPUT_HANDLE: i32 = -11;

    record Buf {
        ptr: ^byte,
        len: u32,
    }

    fun write_buf(handle: ^(), b: Buf) -> (ok: bool, written: u32) {
        var n: u32;
        var success: bool = write_file(handle, b.ptr, b.len, @n, nil);
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
        var stdout: ^() = Con.get_std_handle(Con.STD_OUTPUT_HANDLE);

        const greeting: string = "Hello, Pint!\n";
        var buf: Con.Buf = { ptr: greeting.ptr, len: cast(u32, greeting.len) };

        var i: usize = 0;
        while (i < 3) {
            var (ok: bool, _) = Con.write_buf(stdout, buf);
            if (not ok) { exit_process(1); }
            i = i + 1;
        }

        exit_process(0);
    }
}
```

### Excluded

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
| Sum types / tagged unions | Storage overhead; multiple returns handle the error case |
| Pattern matching | Complexity; no sum types means nothing to match on |
| Closures | Complexity; function pointers cover the use cases |
| Error unions (`!T`) | Multiple return values serve the same purpose |
