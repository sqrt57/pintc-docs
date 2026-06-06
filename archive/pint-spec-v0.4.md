# Pint Language Spec v0.4

## Philosophy

Explicit over implicit. Predictable over convenient. The programmer is in control.
No hidden allocations. No runtime. No magic. Every byte accounted for.
Designed for OS kernels, embedded firmware, and low-level systems work.

LLM-friendly by design: the spec is small enough to fit in a context window, and
the language consistent enough that correct code can be generated from it with
minimal examples. Special cases, implicit rules, and magic behavior are the enemy
of both human readers and code-generating models.

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

fun compute() : () -> i32 {
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
fun f() : () -> () {
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
fun is_even(n) : (i32) -> bool {
    if (n == 0) { return true; }
    return is_odd(n - 1);   // valid — is_odd collected before resolution
}

fun is_odd(n) : (i32) -> bool {
    if (n == 0) { return false; }
    return is_even(n - 1);
}
```

Module-scope `var` initializers are sequential — they may only reference names
declared above them, since initialization order matters at runtime.

---

## Pointers

One pointer type:

```
^T       // raw pointer — nullable, no safety guarantees
```

`^()` is the untyped pointer — pointer to the unit type, used where C would use
`void *`. It can hold any address but cannot be dereferenced without a `cast`.

Dereference is postfix `p^`. Address-of is `@x`:

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

```
[N]T     // fixed-size array, N known at compile time
```

Array literals use square brackets. The element type is inferred from context:

```
var arr: [3]i32 := [1, 2, 3];
```

Element access and assignment use subscript notation:

```
var x: i32 := arr[1];   // read element
arr[0] := 10;           // write element
```

A pointer to the first element:

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
fun rect_area(r) : (^Rect) -> f64 {
    return r^.width * r^.height;
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
fun scale_by_value(p) : (Point) -> Point {
    return { x: p.x * 2.0, y: p.y * 2.0 };
}

fun scale_in_place(p) : (^Point) -> () {
    p^.x := p^.x * 2.0;
    p^.y := p^.y * 2.0;
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

`string` is the type of any string literal. The compiler stores the UTF-8 byte content
in the executable as read-only data and appends a null byte after the content.
`string` is not `^byte` — it is its own distinct type.

Named `string` declarations bind a string literal to a module-scope name.
String literals also appear inline in expressions — both forms have type `string`.

Two operations are defined on any `string` value:

- `@s` — returns `^byte`, a pointer to the first byte of the content
- `length(s)` — returns the byte count as `usize`, excluding the null terminator

```
const greeting: string := "Hello, world!\n";
var p: ^byte := @greeting;
var len: usize := length(greeting);   // compile-time constant

var q: ^byte  := @"inline";           // address of anonymous string literal
var n: usize  := length("inline");    // 6
```

Source files are UTF-8; string literal content may contain UTF-8 directly.
The `\u{XXXXXX}` escape inserts the UTF-8 encoding of a Unicode codepoint (1–4 bytes):

```
const msg: string := "caf\u{E9}";       // "café" — \u{E9} encodes é as 2 bytes
const emoji: string := "\u{1F600}";     // 😀 — 4 bytes
```

All escape sequences: `\n`, `\r`, `\t`, `\\`, `\"`, `\0`, `\u{XXXXXX}`.
The null terminator is appended automatically; `length` does not count it.

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

Signature style: parameter names first, then types separately:

```
fun rect_area(r) : (^Rect) -> f64 {
    return r^.width * r^.height;
}

fun newline() : () -> () {
}
```

Multiple return values — primary error handling mechanism:

```
fun open_file(path, len) : (^byte, usize) -> (^File, Error) {
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
var file, err := open_file(@config, length(config));
if (err != Error.None) { ... }

var file, _ := open_file(@config, length(config));   // discard error
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
var file, err := open_file(@config, length(config));
if (err != Error.None) { panic(); }
use_file(file);
```

A function with a non-`()` return type that reaches the end without a `return`
produces a compiler warning and undefined behavior at runtime.

### Attributes

Attributes are compile-time metadata attached to declarations, written in
square brackets immediately before the declaration they apply to. They affect
code generation, type checking, or linking — never runtime behavior.

`[noreturn]` declares that a function never returns to its caller. The compiler
treats any call to a `[noreturn]` function as a diverging control-flow path,
satisfying return requirements on all branches:

```
[noreturn]
fun panic() : () -> () {
    exit_process(1);
}

fun unwrap(p) : (^i32) -> i32 {
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

fun on_north(x) : (i32) -> () { ... }
fun on_south(x) : (i32) -> () { ... }
fun on_east(x)  : (i32) -> () { ... }
fun on_west(x)  : (i32) -> () { ... }

var handlers: [4]^Handler := [@on_north, @on_south, @on_east, @on_west];

var d: i32 := cast(i32, direction);
handlers[d](value);    // dispatch by index
```

---

## Control Flow

Conditions in `if`, `while`, and `for` require parentheses.

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
| Postfix | `a^` `a[i]` `a.b` `f(args)` | Dereference, subscript, field access, call |
| Unary | `-a` `not a` `@x` | Negation, bitwise/logical not, address-of |
| Multiplicative | `*` `/` `%` | |
| Additive | `+` `-` | |
| Bit shift | `<<` `>>` | |
| Comparison | `<` `>` `<=` `>=` | |
| Equality | `==` `!=` | |
| And | `a and b` | Bitwise and logical AND |
| Xor | `a xor b` | Bitwise and logical XOR |
| Or | `a or b` | Bitwise and logical OR |

`and`, `or`, `xor`, `not` are keywords. They apply to both integer operands
(bitwise) and boolean operands (logical) — the meaning is determined by operand type.

```
var a: i32  := 0xff and 0x0f;   // bitwise AND → 0x0f
var b: bool := true and false;  // logical AND → false

var c: i32  := not 0xff;        // bitwise NOT → -256 (all bits flipped)
var d: bool := not true;        // logical NOT → false

var e: i32  := 0b1010 xor 0b1100;  // bitwise XOR → 0b0110
var f: bool := true xor false;     // logical XOR → true
```

When both operands are `bool`, `and` and `or` short-circuit: the right operand
is not evaluated if the result is already determined by the left. `xor` on `bool`
always evaluates both operands. Bitwise operations on integer types never
short-circuit.

Unary `-` is only valid on signed integer and float types. Applying it to an
unsigned type is a compile error:

```
var x: u32 := 10;
var y: u32 := -x;   // error: unary `-` on unsigned type
```

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
- Arithmetic on the above: `c * 5 + 8`, `c and 0xff`

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
const mask: i32 := c and 0xff;

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
extern [dll_import(dll="kernel32.dll", entry_point="HeapAlloc")]
fun heap_alloc(heap, flags, size) : (^(), u32, u32) -> ^();

extern [dll_import(dll="kernel32.dll", entry_point="HeapFree")]
fun heap_free(heap, flags, ptr) : (^(), u32, ^()) -> bool;

extern [dll_import(dll="kernel32.dll", entry_point="GetProcessHeap")]
fun get_process_heap() : () -> ^();
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

    fun hello(name, len) : (^byte, usize) -> () {
    }

    export hello, answer, Node;
}

module Y {
    import X123 as X;

    fun greet() : () -> () {
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
    fun add(a, b) : (V2, V2) -> V2 {
        return { x: a.x + b.x, y: a.y + b.y };
    }
    export add, V2(*);
}

// main.pnt
module Main {
    import Vec as V;

    [entry]
    fun main() : () -> () {
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
extern [dll_import(dll="kernel32.dll", entry_point="ExitProcess")]
[noreturn]
fun exit_process(code) : (u32) -> ();

[entry]
fun main() : () -> () {
    // program logic
    exit_process(0);
}
```

`[entry]` is an attribute designating the Win32 entry point. Returning from the
entry function is undefined behavior — the developer must call `ExitProcess`
explicitly before returning.

---

## FFI / C Interop

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
extern [dll_import(dll="kernel32.dll", entry_point="GetLastError")]
fun get_last_error() : () -> u32;

extern [dll_import(dll="kernel32.dll", entry_point=5)]
fun some_ordinal() : () -> u32;

extern [dll_import(dll="msvcrt.dll", entry_point="printf", conv=cdecl)]
fun printf(fmt) : (^byte) -> i32;
```

### Exporting symbols

The compiler can produce an EXE or a DLL — selected by a compiler flag.
`[dll_export]` is an attribute that adds a function to the binary's export table;
it is only meaningful when building a DLL.

```
[dll_export]
fun my_add(a, b) : (i32, i32) -> i32 {
    return a + b;
}

[dll_export(conv=stdcall)]
fun my_stdcall_fn(x) : (i32) -> i32 {
    return x;
}
```

Note: module-level `export` (inside a `module` block) controls Pint-internal
visibility only — it does not affect the binary export table.

---

## Builtins

| Builtin | Description |
|---|---|
| `sizeof(T)` | `usize` — size of type or variable in bytes |
| `length(x)` | `usize` — element count of outermost array dimension, or byte count of `string` literal (excluding null terminator) |
| `cast(T, x)` | Explicit type cast (numeric conversions, pointer ↔ pointer, pointer ↔ integer) |
| `divmod(a, b)` | `(quotient, remainder)` — integer division and remainder in one op |
| `mul(a, b)` | `(high, low)` — full-width multiply (e.g. 32×32→64) |
| `to_i8(x)` … `to_u32(x)` | Narrowing integer cast → `(value, overflow)` |

```
var q, r := divmod(17, 5);          // q=3, r=2

var hi, lo := mul(cast(u32, a), cast(u32, b));  // 32×32→64 split into two u32
```

---

## Excluded

| Feature | Rationale |
|---|---|
| Generics | Complexity, compile time cost |
| Exceptions | Unpredictable control flow, hidden overhead |
| Garbage collector | Unacceptable for OS / embedded targets |
| Inheritance | Complexity without payoff |
| Operator overloading | Readability — operations should be explicit |
| Implicit conversions | Source of subtle bugs |
| Forward declarations | Unnecessary with a collection pass; adds surface area |
