# Pint Language Spec v0.2 — Combined Draft

## Philosophy

Explicit over implicit. Predictable over convenient. The programmer is in control.
No hidden allocations. No runtime. No magic. Every byte accounted for.
Designed for OS kernels, embedded firmware, and low-level systems work.

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

---

## Primitive Types

| Type | Size | Description |
|---|---|---|
| `i8`, `i16`, `i32`, `i64` | 1–8 bytes | Signed integers |
| `u8`, `u16`, `u32`, `u64` | 1–8 bytes | Unsigned integers |
| `f32`, `f64` | 4–8 bytes | IEEE 754 floats |
| `bool` | 1 byte | `true` / `false` |
| `byte` | 1 byte | Alias for `u8` |
| `()` | — | Zero-element tuple; used where no value is produced or expected |

No implicit numeric conversions. All casts are explicit.

---

## Pointers

One pointer type:

```
^T       // raw pointer — nullable, no safety guarantees
```

Dereference is postfix `p^`. Address-of is `@x`:

```
var x: i32;
var p: ^i32 := @x;
p^ := 10;
```

Pointer arithmetic is explicit:

```
var p: ^u8 := @buf;
var next: ^u8 := p + 1;
```

No references, no smart pointers, no RAII.

---

## Arrays

```
[N]T     // fixed-size array, N known at compile time
```

No bounds checking.

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

### Memory layout control

For interop and systems work, exact layout can be specified:

```
record Sys [size=4] {
    [offset=0] x: i16,
    [offset=2] y: i16,
    [offset=0] value: i32,   // union-like overlap
}
```

### Nominal vs structural typing

Records are nominal — two records with identical fields are distinct incompatible types.
Arrays, pointers, and function types are structural.

### Self-referential and mutually recursive records

The compiler collects all top-level type names before resolving any references, so both self-reference and mutual recursion work without forward declarations:

```
record Node {
    next: ^Node,
    value: i32,
}

record A { b: ^B, }
record B { a: ^A, }
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
fun open_file(path, len) : (^byte, u32) -> (^File, Error) {
    if !fs_exists(path) {
        return nil, Error.NotFound;
    }
    return f, Error.None;
}
```

Call site unpacking:

```
var file, err := open_file("config.txt");
if err != Error.None { ... }

var file, _ := open_file("config.txt");   // discard error
```

No exceptions. No `Result` type. Errors are plain enum values, checked explicitly at every call site.

### Function pointers

No closures. Address operator works on defined functions:

```
type Callback = (^()) -> ();

var cb: ^Callback := @some_fn;
cb(data);      // implicit dereference
cb^(data);     // explicit dereference — both are valid
```

---

## Control Flow

```
if x > 0 {
    ...
} else if x < 0 {
    ...
} else {
    ...
}

for i: u32 = 0; i < len; i += 1 {
    ...
}

loop {
    if done { break; }
    if skip { continue; }
}
```

No `switch`. No ternary. No exceptions.

---

## Type Casts

All explicit with `cast(T, x)`:

```
var y: i64 := cast(i64, x);
var p: ^() := cast(^(), ptr);
```

Narrowing with overflow detection via builtin intrinsics:

```
var y, overflow := to_i16(x);    // (i16, bool)
var y, overflow := to_u8(x);     // (u8, bool)
```

Available narrowing intrinsics: `to_i8`, `to_u8`, `to_i16`, `to_u16`, `to_i32`, `to_u32`.

---

## Memory Management

No allocator built into the language. Use WinAPI directly via FFI:

```
extern [dll_import="kernel32.dll", entry_point="HeapAlloc"]
fun heap_alloc(heap: ^(), flags: u32, size: u32) : (^(), u32, u32) -> ^();

extern [dll_import="kernel32.dll", entry_point="HeapFree"]
fun heap_free(heap: ^(), flags: u32, ptr: ^()) : (^(), u32, ^()) -> bool;

extern [dll_import="kernel32.dll", entry_point="GetProcessHeap"]
fun get_process_heap() : () -> ^();
```

No GC. No destructor calls. No RAII. You free what you allocate.

---

## Module System

Each source file contains one or more named module definitions. Modules have explicit export and import lists:

```
module X123 {
    type Node = record {
        value: i32,
        tag:   i32,
    }

    const answer: i32 := 42;
    var count: i32 := 15;

    fun hello(name, len) : (^byte, u32) -> () {
    }

    export hello, answer, Node;
}

module Y {
    import X123 as X;

    fun greet() : () -> () {
        X.hello("Johnny");
    }
}
```

- Symbols not in `export` are private to the module.
- `import M as alias` makes symbols available as `alias.symbol` or unqualified if unambiguous.

---

## FFI / C Interop

Import external C symbols with platform metadata:

```
extern [dll_import="kernel32.dll", entry_point="GetLastError"]
fun get_last_error() : () -> u32;

extern [dll_import="kernel32.dll", entry_point=5]
fun some_ordinal() : () -> u32;
```

Expose functions as C-ABI symbols:

```
export fun my_add(a, b) : (i32, i32) -> i32 {
    return a + b;
}
```

Struct layout is C-compatible by default.

---

## Builtins

| Builtin | Description |
|---|---|
| `sizeof(T)` | Size of type or variable in bytes |
| `length(x)` | Length of outermost array dimension |
| `cast(T, x)` | Explicit type cast (pointer ↔ pointer, pointer ↔ integer) |
| `divmod(a, b)` | `(quotient, remainder)` — integer division and remainder in one op |
| `mul(a, b)` | `(high, low)` — full-width multiply (e.g. 32×32→64) |
| `to_i8(x)` … `to_u32(x)` | Narrowing integer cast → `(value, overflow)` |

---

## Compilation Targets

**Native** — via cranelift (phase 1), LLVM (phase 2). Produces standalone binary or linkable `.o`.

```
pintc build main.pnt
```

---

## Deliberately Omitted

| Feature | Rationale |
|---|---|
| Generics | Complexity, compile time cost |
| Exceptions | Unpredictable control flow, hidden overhead |
| Garbage collector | Unacceptable for OS / embedded targets |
| Inheritance | Complexity without payoff |
| Operator overloading | Readability — operations should be explicit |
| Implicit conversions | Source of subtle bugs |
| Macros | Language should be readable as-is |
| Concurrency primitives | Out of scope — use OS threads or platform APIs |
| Forward declarations | Unnecessary with a collection pass; adds surface area |
| Bounds checking | Overhead unacceptable for systems targets |
| Fat pointers / slices | Length is caller's responsibility — no hidden state in pointers |

