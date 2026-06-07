# pintc Command-Line Interface

## Synopsis

```
pintc [options] <file>...
```

One or more `.pnt` source files. All files are compiled together as a single
program; a module declared across multiple files is valid.

## Options

| Option | Description |
|---|---|
| `-o <file>` | Output file path. Default: stem of the first input file plus `.exe` or `.dll`. |
| `--exe` | Produce an EXE. Default; provided for use in scripts and makefiles. |
| `--dll` | Produce a DLL instead of an EXE. |
| `--version` | Print version and exit. |
| `--help` | Print usage and exit. |

## Output

Produces a single PE32 binary. The binary type is determined by `--exe` / `--dll`:

| Flag | Output | Entry point |
|---|---|---|
| *(absent)* or `--exe` | EXE (`IMAGE_SUBSYSTEM_WINDOWS_GUI`) | Function marked `[win32_entry]` |
| `--dll` | DLL | Functions marked `[dll_export]` |

Passing both `--exe` and `--dll` is a usage error.

A `[win32_entry]` function in a DLL build is a compiler error. A `[dll_export]`
function in an EXE build is silently ignored.

## Diagnostics

Errors and warnings are written to stderr:

```
path/to/file.pnt:10:5: error: message text
path/to/file.pnt:14:1: warning: message text
```

Line and column numbers are 1-based.

## Exit codes

| Code | Meaning |
|---|---|
| `0` | Success |
| `1` | Compilation error |
| `2` | Bad arguments |

## Examples

Compile a single-file program:
```
pintc main.pnt
```

Specify output name:
```
pintc -o hello.exe main.pnt
```

Multi-file build:
```
pintc -o app.exe main.pnt vec.pnt console.pnt
```

Build a DLL:
```
pintc --dll -o mylib.dll lib.pnt
```
