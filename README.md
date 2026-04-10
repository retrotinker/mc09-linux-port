# Dunfield Micro-C 6809 Toolchain — Linux Port

A Linux port of Dave Dunfield's **Micro-C** cross-compiler and **asm09**
assembler for the Motorola 6809.

Original source: [Dunfield Development Services](https://dunfield.themindfactory.com)
— released as freeware.  See `COPY.TXT` for licence terms.

---

## What's here

| File | Description |
|------|-------------|
| `mcc09` | Micro-C 6809 cross-compiler (K&R C subset → 6809 asm) |
| `asm09` | Dunfield 6809 cross-assembler (→ Motorola / Intel HEX) |
| `mc09pp` | Post-processor: converts mcc09 output for standalone assembly |
| `include/` | Dunfield 6809 runtime headers (`6809io.h`, `stddef.h`, …) |

---

## Build

```sh
make
```

Requires only `gcc` and `make`.  The sources are K&R C; GCC is invoked with
`-std=gnu89` and a handful of `-Wno-*` flags to silence the expected
implicit-declaration warnings endemic to the Dunfield codebase.

---

## Smoke test

```sh
make test
```

Compiles `hello.c`, post-processes the output, assembles it, and prints the
Motorola S-record HEX file to stdout.

---

## Toolchain pipeline

```
  hello.c
     │
     ▼  mcc09 -I./include hello.c hello.asm
  hello.asm          (Dunfield source-linked asm format)
     │
     ▼  mc09pp 0C000H hello.asm > hello_pp.asm
  hello_pp.asm       (standalone, $EX: resolved to stubs)
     │
     ▼  asm09 hello_pp.asm l=hello.lst
  hello.HEX          (Motorola S-records)
  hello.lst          (annotated listing)
```

In Dunfield's original DOS toolchain the middle step is **slink** (source
linker), which resolves `$EX:` external declarations against a pre-built
runtime library (`lib09/`).  `mc09pp` is a minimal substitute that inserts
`EQU $DEAD` stubs — useful for inspecting generated code before you have a
working runtime.

To use a real runtime, assemble the `lib09/` sources with asm09 and pass all
the resulting `.HEX` files to a hex-merge tool, or port slink next.

---

## Compiler options (`mcc09`)

| Flag | Effect |
|------|--------|
| `-I<path>` | Add include search path (or set `MCINCLUDE` env var) |
| `-q` | Quiet (suppress banner) |
| `-s` | Emit symbolic debug comments |
| `-c` | Include source lines as comments in output |

## Assembler options (`asm09`)

| Flag | Effect |
|------|--------|
| `-s` | Include symbol table in listing |
| `-f` | Full listing (include non-code lines) |
| `-i` | Intel HEX output (default: Motorola) |
| `-q` | Quiet |
| `l=<file>` | Write listing to file (default: stdout) |
| `c=<file>` | Write HEX to file (default: `<input>.HEX`) |

---

## Linux porting notes

Six bugs fixed to make the DOS K&R sources build and run correctly on
64-bit Linux:

1. **`abort(msg)`** — Dunfield's signature takes a message string; ANSI
   `abort()` takes none.  Routed via a function-like macro in `portab.h`.

2. **CRLF stripping** — DOS line endings stripped in `MC_fgets` (assembler)
   and `get_lin` (compiler).

3. **`#CPU` stringize** — K&R token-pasting `"foo"#MACRO"bar"` is not valid
   in modern cpp.  Replaced with a proper `STRINGIFY()` double-macro.

4. **`skip_comment()` 16-bit window** (`compile.c`) — The comment scanner uses
   a two-character sliding window stored in `unsigned x`.  On DOS, `unsigned`
   is 16 bits, so the 8-bit left-shift stays within the word.  On LP64,
   `unsigned` is 32 bits and the pattern never matches.  Fixed: `unsigned short x`.

5. **Block comments spanning `#include` boundaries** (`compile.c`) — When
   `skip_comment()` reaches EOL inside an included file, `read_char()` crosses
   the file boundary and starts consuming the parent source.  Fixed: track
   `in_comment` depth; inject a `*/` sentinel on include pop-back so the
   scanner closes at the file boundary (standard C behaviour).

6. **`optr`/`itype`/`otype`/`post` types** (`asm09.c`) — `optr` was `char`,
   used as an array index into `operand[200]`; values > 127 wrap negative.
   `itype`/`otype` hold opcode class codes (`0x81`–`0x8a`), which are > 127
   and read as negative in signed char.  Fixed: `optr` → `int`;
   `itype`/`otype`/`post` → `unsigned char`.

7. **`isterm()` missing `\n`** (`asm09.c`) — `fgets()` includes the trailing
   newline in the buffer.  `eval()` saw `\n` as an unrecognised operator and
   fired "invalid expression syntax" after every successfully-assembled line.
   Fixed: added `\n`, `\r`, and `;` to `isterm()`.
