# Language Specification v0.1

## Philosophy

> Explicit over implicit. Predictable over convenient. The programmer is in control.

No hidden allocations. No runtime. No magic. Every byte accounted for. Designed for OS kernels, embedded firmware, and WASM modules where you need to know exactly what the machine is doing.

---

## Profile Summary

| Decision | Choice |
|---|---|
| **Primary goal** | Systems / OS / embedded |
| **Memory model** | Manual |
| **Type system** | Strong static, explicit |
| **Syntax** | C-family curly braces |
| **Error handling** | Multiple return values |
| **Concurrency** | None built-in |
| **Abstraction** | Structs only |
| **Metaprogramming** | None |
| **Generics** | None — monomorphic |
| **Targets** | Native binary + WASM |
| **C interop** | FFI layer |

---

## 1. Primitive Types

| Type | Size | Description |
|---|---|---|
| `i8`, `i16`, `i32`, `i64` | 1–8 bytes | Signed integers |
| `u8`, `u16`, `u32`, `u64` | 1–8 bytes | Unsigned integers |
| `f32`, `f64` | 4–8 bytes | IEEE 754 floats |
| `bool` | 1 byte | `true` / `false` |
| `byte` | 1 byte | Alias for `u8` |
| `usize` | platform | Pointer-sized unsigned int |
| `isize` | platform | Pointer-sized signed int |
| `void` | — | No value |

No implicit numeric conversions. All casts are explicit.

```c
let x: i32 = 10;
let y: i64 = x as i64;   // explicit cast required
```

---

## 2. Pointers

Two pointer types, both explicit:

```c
*T        // raw pointer — nullable, no safety guarantees
[*]T      // pointer to array — carries no length
```

Pointer arithmetic is allowed and explicit:

```c
let p: *u8 = buf;
let next: *u8 = p + 1;
```

No references, no smart pointers. You own the pointer, you manage the lifetime.

---

## 3. Arrays and Slices

```c
[N]T      // fixed-size array, N known at compile time
[]T       // slice — fat pointer (ptr + length), no ownership
```

```c
let buf: [256]u8 = zero;
let view: []u8 = buf[0..128];
```

Slices are the primary safe view into memory — bounds-checked by default, with an opt-out pragma for hot paths.

---

## 4. Structs

The only compound type. No inheritance, no vtables, no interfaces.

```c
struct Point {
    x: f64,
    y: f64,
}

struct Rect {
    origin: Point,
    width:  f64,
    height: f64,
}
```

Methods are free functions that take a struct pointer — no method syntax sugar, keeping the language surface small:

```c
fn rect_area(r: *Rect) f64 {
    return r.width * r.height;
}
```

---

## 5. Functions and Multiple Return Values

Functions return a parenthesized tuple for multiple values. This is the primary error handling mechanism.

```c
fn open_file(path: []byte) (*File, Error) {
    if !fs_exists(path) {
        return nil, Error.NotFound;
    }
    let f: *File = alloc(File);
    // ...
    return f, Error.None;
}
```

Call site unpacking is explicit:

```c
let file, err = open_file("config.txt");
if err != Error.None {
    // handle
}
```

The `_` discard is available for ignored return values:

```c
let file, _ = open_file("config.txt");
```

---

## 6. Error Handling Pattern

No exceptions. No `Result` type. Errors are values — plain enums returned as the last value:

```c
enum Error {
    None,
    NotFound,
    PermissionDenied,
    OutOfMemory,
    InvalidInput,
}
```

Each module defines its own error enum. Errors are checked explicitly at every call site — no propagation magic. Verbose by design: if you don't handle it, you made a deliberate choice.

---

## 7. Control Flow

Minimal and familiar:

```c
// conditionals
if x > 0 {
    // ...
} else if x < 0 {
    // ...
} else {
    // ...
}

// loops
for i: usize = 0; i < len; i += 1 {
    // ...
}

// while-style
loop {
    if done { break; }
}

// range over slice
for item in items {
    // ...
}
```

No `switch` — use if/else chains. No ternary operator. No exceptions.

---

## 8. Memory Management

No allocator built into the language. You bring your own:

```c
// built-in low-level primitives
alloc(T) *T                  // allocate one T
alloc_n(T, n: usize) [*]T   // allocate n T's
free(ptr: *T) void           // free pointer
```

The idiomatic pattern is passing an allocator struct into functions that need to allocate:

```c
struct Allocator {
    alloc:   fn(usize) *void,
    free:    fn(*void) void,
}

fn make_buffer(a: *Allocator, n: usize) ([]byte, Error) {
    let ptr: *void = a.alloc(n);
    if ptr == nil {
        return nil_slice, Error.OutOfMemory;
    }
    return slice(byte, ptr, n), Error.None;
}
```

No garbage collector. No destructor calls. No RAII. You free what you allocate.

---

## 9. Enums

Flat, C-style enums. No attached data (no algebraic types):

```c
enum Direction {
    North,
    South,
    East,
    West,
}

let d: Direction = Direction.North;
```

---

## 10. Module System

One file = one module. Explicit imports, no circular dependencies:

```c
import "std/io"
import "std/mem"
import "../mylib/parser"

// use with module prefix
io.write(fd, buf);
```

Public symbols are exported by default. Private symbols prefixed with `_`:

```c
fn compute() i32 { ... }     // public
fn _helper() i32 { ... }     // private to module
```

---

## 11. FFI / C Interop

Declare external C symbols explicitly. No header parsing — declarations are manual:

```c
extern "C" {
    fn malloc(size: usize) *void;
    fn free(ptr: *void) void;
    fn printf(fmt: *byte, ...) i32;
}
```

Struct layout is C-compatible by default — no hidden padding reordering. Calling C libraries requires no wrapper layer for simple cases.

Exposing functions to C:

```c
export fn my_add(a: i32, b: i32) i32 {
    return a + b;
}
```

The `export` keyword emits a C-ABI-compatible symbol usable from any C caller or WASM host.

---

## 12. Compilation Targets

**Native binary** — compiles to machine code via cranelift (phase 1) or LLVM (phase 2). Produces a standalone executable or linkable `.o` object file.

**WASM** — first-class target, not an afterthought. `export` functions become WASM exports. Memory is explicit (`alloc`/`free` map to WASM linear memory operations). No WASM-specific syntax needed — the same source compiles to both.

Target is specified at compile time:

```bash
myc build --target native main.my
myc build --target wasm32 main.my
```

---

## 13. What the Language Deliberately Omits

| Feature | Rationale |
|---|---|
| Generics | Complexity, compile time cost — use monomorphic functions |
| Exceptions | Unpredictable control flow, hidden overhead |
| Garbage collector | Unacceptable for OS / embedded targets |
| Inheritance | Complexity without sufficient payoff |
| Operator overloading | Readability — operations should be explicit |
| Implicit conversions | Source of subtle bugs |
| Macros | Complexity — the language should be readable as-is |
| Concurrency primitives | Out of scope — use OS threads or platform APIs |
