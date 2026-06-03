# Software Introspection: Data Types, Disassembly & GDB

> Notes on number systems, disassembling binaries, tracing syscalls, and debugging with GDB — for x86-64 Linux assembly.

---

## Table of Contents

- [Number Systems](#number-systems)
- [Disassembling a Program](#disassembling-a-program)
- [Tracing Syscalls with strace](#tracing-syscalls-with-strace)
- [GDB — The GNU Debugger](#gdb--the-gnu-debugger)
  - [Starting GDB](#starting-gdb)
  - [Disassembling in GDB](#disassembling-in-gdb)
  - [Stepping Through Instructions](#stepping-through-instructions)
  - [Reading Register Values](#reading-register-values)
  - [Popping Stack Values](#popping-stack-values)
  - [Examining Memory](#examining-memory)
  - [Examining Stack Pointers](#examining-stack-pointers)
  - [Cooperative Debugging with int3](#cooperative-debugging-with-int3)
  - [Running with Arguments](#running-with-arguments)

---

## Number Systems

Three number systems show up constantly in assembly: **decimal**, **binary**, and **hexadecimal**.

| Hex | Binary | Decimal |
|-----|--------|---------|
| `0` | `0000` | 0  |
| `1` | `0001` | 1  |
| `2` | `0010` | 2  |
| `3` | `0011` | 3  |
| `4` | `0100` | 4  |
| `5` | `0101` | 5  |
| `6` | `0110` | 6  |
| `7` | `0111` | 7  |
| `8` | `1000` | 8  |
| `9` | `1001` | 9  |
| `a` | `1010` | 10 |
| `b` | `1011` | 11 |
| `c` | `1100` | 12 |
| `d` | `1101` | 13 |
| `e` | `1110` | 14 |
| `f` | `1111` | 15 |

> **Key idea:** One hex digit = exactly 4 binary bits. Two hex digits = one byte (8 bits). This is why memory addresses and raw instruction bytes are shown in hex.

---

## Disassembling a Program

**Disassembly** is the process of converting binary machine code back into human-readable assembly instructions.

The most common tool is **`objdump`**. The `-d` flag disassembles executable sections, and `-M intel` forces Intel syntax (always use this):

```bash
objdump -d -M intel /challenge/disassemble-me
```

### Example output

```
/challenge/disassemble-me:     file format elf64-x86-64

Disassembly of section .text:

0000000000401000 <__bss_start-0x1000>:
  401000:  48 c7 c7 f6 1b 00 00    mov    rdi,0x1bf6   ← main value required
  401007:  48 c7 c7 00 00 00 00    mov    rdi,0x0
  40100e:  48 c7 c0 3c 00 00 00    mov    rax,0x3c
  401015:  0f 05                   syscall
```

### Three things to notice

1. **Always use `-M intel`** — without it, objdump defaults to AT&T syntax, which is confusing and harder to read.
2. **Raw bytes are shown alongside instructions** — e.g. `0f 05` is the `syscall` instruction in hex.
3. **Register values are in hexadecimal** — e.g. `0x1bf6` is the decimal value `7158`.

---

## Tracing Syscalls with strace

**`strace`** uses Linux OS functionality to record every system call a program makes, along with its parameters and return values.

Output format: `syscall_name(param, param, ...)` — borrowed from C syntax.

```bash
strace /challenge/trace-me
```

### Example output

```
execve("/challenge/trace-me", ["/challenge/trace-me"], 0x7ffc23db5940 /* 15 vars */) = 0
alarm(13123)                            = 0
exit(0)                                 = ?
+++ exited with 0 +++
```

### Reading the output

| Syscall   | What it does |
|-----------|--------------|
| `execve`  | Launches the program (yin to `exit`'s yang — we'll learn more later) |
| `alarm`   | Sets a timer — here with value `13123` |
| `exit(0)` | Program terminates with exit code `0` |

> strace is useful for understanding what a program is *doing* at the OS level without reading source code.

---

## GDB — The GNU Debugger

**GDB** (GNU Debugger) is the most common debugger on Linux. It lets you monitor and introspect a running process — pause execution, inspect registers, read memory, and step through instructions one at a time.

> **GNU** = a massive, completely free collection of software. Combined with Linux it forms the "GNU/Linux" operating system.

---

### Starting GDB

```bash
gdb /path/to/binary
# e.g.
gdb /challenge/debug-me
```

To quit GDB:

```gdb
(gdb) quit
# or shorthand:
(gdb) q
```

---

### Disassembling in GDB

Typical flow:

```
gdb → starti → disassemble → quit
```

- `starti` — launches the program and immediately pauses at the first instruction
- `disassemble` — shows the assembly of the current function

---

### Stepping Through Instructions

When a value is `CENSORED` in the challenge output, step through the instruction to reveal it:

```
gdb → starti → stepi → quit
```

- `stepi` (or `si`) — executes exactly one instruction and pauses again

---

### Reading Register Values

To read the value in a register after stepping:

```
(gdb) starti
(gdb) stepi
(gdb) print $rdi
```

> **Confusion clarified:** Always run `stepi` **after** `starti` before reading registers. If you skip `stepi`, you'll read the register value from *after* the program ends — not the mid-execution value you actually want.

---

### Popping Stack Values

Same flow as reading registers — the pop instruction moves the top stack value into a register, so you step through it and then read:

```
(gdb) starti
(gdb) stepi
(gdb) print $rdi
```

---

### Examining Memory

To inspect the value at the top of the stack (`rsp = argc` at program entry):

```gdb
(gdb) starti
(gdb) stepi
(gdb) x/d $rsp
```

- `x` = examine memory
- `/d` = display as **decimal** (without `/d` it shows hex by default)
- `$rsp` = the stack pointer register

---

### Examining Stack Pointers

To read a pointer from the stack and then follow it to see the string it points to:

```gdb
# Step 1: read the pointer value at rsp+16 (argv[1])
(gdb) x/a $rsp+16

# Output example:
# 0x7ffc001c4750

# Step 2: follow the pointer to see the string
(gdb) x/s 0x7ffc001c4750
```

| Format flag | Meaning |
|-------------|---------|
| `/a`        | Display as a memory **address** (hex) |
| `/s`        | Display as a **string** |
| `/d`        | Display as **decimal** |

Full flow:

```
starti → stepi → x/a $rsp+16 → x/s <address>
```

---

### Cooperative Debugging with `int3`

Instead of us stopping the program with `starti`, the **program itself** can signal the debugger to stop using the `int3` instruction. This is called a **breakpoint**.

- On x86-64, `int3` tells a connected debugger: *"pause here"*
- In GDB, use `run` (not `starti`) — the program runs until it hits `int3` and stops automatically

### Example: write a program with a breakpoint

```asm
# p.s
.intel_syntax noprefix
.global _start
_start:
    mov rdi, 1337
    int3          # signal debugger to pause here
    syscall
```

```bash
echo -e ".intel_syntax noprefix\n.global _start\n_start:\n mov rdi, 1337\n int3\n syscall" > p.s
as -o p.o p.s && ld -o p p.o && /challenge/check ./p
```

> No need to manually run the program in GDB — `/challenge/check` handles it. The `int3` hands control to the debugger automatically.

---

### Running with Arguments

Outside GDB you pass arguments normally:

```bash
./program argument
```

Inside GDB, use `run` followed by the arguments:

```gdb
(gdb) run argument
```

Whatever follows `run` becomes `argv[1]`, `argv[2]`, etc. GDB also accepts the shorthand `r`:

```gdb
(gdb) r argument
```

| Context      | Command            | Result              |
|--------------|--------------------|---------------------|
| Shell        | `./program foo`    | `argv[1]` = `"foo"` |
| GDB          | `run foo`          | `argv[1]` = `"foo"` |
| GDB shorthand| `r foo`            | `argv[1]` = `"foo"` |

---

## Quick Reference

| Tool / Command       | What it does                                              |
|----------------------|-----------------------------------------------------------|
| `objdump -d -M intel`| Disassemble binary in Intel syntax                       |
| `strace`             | Trace all syscalls a program makes                        |
| `gdb /binary`        | Launch GDB with a binary                                  |
| `starti`             | Start program, pause at first instruction                 |
| `run` / `r`          | Run program (until `int3` or end)                        |
| `stepi` / `si`       | Execute one instruction                                   |
| `print $rdi`         | Print current value of register `rdi`                    |
| `x/d $rsp`           | Examine memory at `rsp` as decimal                       |
| `x/a $rsp+16`        | Examine memory at `rsp+16` as address (hex)              |
| `x/s <addr>`         | Examine memory at address as string                      |
| `quit` / `q`         | Exit GDB                                                  |
| `int3`               | Breakpoint instruction — pauses execution if debugger attached |
