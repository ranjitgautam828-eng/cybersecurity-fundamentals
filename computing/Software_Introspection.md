# Software Introspection: Disassembly, Syscall Tracing & GDB

> **Where this fits:** We now know how to *write* assembly. Introspection is the other direction — looking *inside* a compiled binary we didn't write (or can't read the source of). These are the tools that let you understand what a program is actually doing at the CPU level.

---

## Table of Contents

- [Number Systems](#number-systems)
- [Disassembling a Binary — objdump](#disassembling-a-binary--objdump)
- [Tracing Syscalls — strace](#tracing-syscalls--strace)
- [GDB — The GNU Debugger](#gdb--the-gnu-debugger)
  - [Starting GDB](#starting-gdb)
  - [Disassembling in GDB](#disassembling-in-gdb)
  - [Stepping Through Instructions](#stepping-through-instructions)
  - [Reading Register Values](#reading-register-values)
  - [Examining Memory](#examining-memory)
  - [Examining Stack Pointers](#examining-stack-pointers)
  - [Cooperative Debugging with int3](#cooperative-debugging-with-int3)
  - [Running with Arguments](#running-with-arguments)
- [Typical GDB Workflows](#typical-gdb-workflows)
- [Security Context](#-security-context--why-this-matters-for-hacking)
- [Quick Reference](#quick-reference)

---

## Number Systems

Three number systems appear constantly in assembly and debugging: **decimal**, **binary**, and **hexadecimal**.

| Hex | Binary | Decimal |
|---|---|---|
| `0` | `0000` | 0 |
| `1` | `0001` | 1 |
| `2` | `0010` | 2 |
| `3` | `0011` | 3 |
| `4` | `0100` | 4 |
| `5` | `0101` | 5 |
| `6` | `0110` | 6 |
| `7` | `0111` | 7 |
| `8` | `1000` | 8 |
| `9` | `1001` | 9 |
| `a` | `1010` | 10 |
| `b` | `1011` | 11 |
| `c` | `1100` | 12 |
| `d` | `1101` | 13 |
| `e` | `1110` | 14 |
| `f` | `1111` | 15 |

**The key relationship:** one hex digit = exactly 4 binary bits. Two hex digits = one byte (8 bits).

```
0x4f  =  0100 1111
  4   =    0100
  f   =         1111
```

**Why hex everywhere?**
- Memory addresses: `0x7ffc001c4750` is shorter than the decimal equivalent
- Raw instruction bytes: `0f 05` is the `syscall` instruction
- Register values in GDB output are always hex by default

---

## Disassembling a Binary — objdump

**Disassembly** converts binary machine code back into human-readable assembly. We use this when we have a compiled program but no source code.

```bash
objdump -d -M intel /path/to/binary
```

| Flag | Meaning |
|---|---|
| `-d` | Disassemble executable sections |
| `-M intel` | Use Intel syntax (always add this — AT&T syntax is confusing) |

### Example output

```
/challenge/disassemble-me:     file format elf64-x86-64

Disassembly of section .text:

0000000000401000 <_start>:
  401000:  48 c7 c7 f6 1b 00 00    mov    rdi, 0x1bf6
  401007:  48 c7 c0 3c 00 00 00    mov    rax, 0x3c
  40100e:  0f 05                   syscall
```

### Reading the output

```
401000:  48 c7 c7 f6 1b 00 00    mov    rdi, 0x1bf6
│         │                       │
│         │                       └── the instruction in Intel syntax
│         └── raw machine code bytes (hex)
└── memory address of this instruction
```

- `0x1bf6` in decimal = **7158** — that would be the exit code here
- `0x3c` in decimal = **60** — that's the exit syscall number
- `0f 05` — that's the 2-byte machine code for `syscall`

> **Always use `-M intel`** — without it, objdump defaults to AT&T syntax where the same instruction looks like `movq $0x1bf6,%rdi`. Both mean the same thing; Intel is just easier to read.

---

## Tracing Syscalls — strace

`strace` intercepts and records every syscall a program makes as it runs. We don't need source code — it works on any binary.

```bash
strace /path/to/binary
strace /path/to/binary argument1 argument2
```

Output format: `syscall_name(arguments...) = return_value`

### Example output

```
execve("/challenge/trace-me", ["/challenge/trace-me"], 0x7ffc23db5940) = 0
alarm(13123)                            = 0
exit(0)                                 = ?
+++ exited with 0 +++
```

### Reading the output

| Line | What it means |
|---|---|
| `execve(...)` | OS launched the program. Always appears first. |
| `alarm(13123)` | Program set a timer for 13123 (this value might be what you need) |
| `exit(0)` | Program called exit with code 0 |

> **`strace` is incredibly useful for CTFs and reverse engineering.** Instead of reading every instruction, we can see exactly which syscalls are called and with what arguments — often that's enough to solve a challenge or understand what a suspicious binary is doing.

### Useful strace flags

```bash
strace -e write ./binary     # only show write() syscalls
strace -o output.txt ./binary  # save output to file (strace prints to stderr by default)
strace -f ./binary           # follow child processes too
```

---

## GDB — The GNU Debugger

GDB lets you run a program in slow motion — pause it, inspect registers, read memory, and step through one instruction at a time.

> **GNU** = a massive collection of free software tools (gcc, gdb, ld, as, objdump are all GNU tools). Together with the Linux kernel they form the OS we're using.

### Starting GDB

```bash
gdb /path/to/binary
```

Quit GDB:
```gdb
(gdb) quit
# or shorthand:
(gdb) q
```

---

### Disassembling in GDB

```gdb
(gdb) starti                ; launch and pause at the very first instruction
(gdb) disassemble           ; show assembly of the current function
(gdb) disassemble _start    ; show assembly of a specific function
```

---

### Stepping Through Instructions

```gdb
(gdb) starti       ; pause at instruction 1
(gdb) stepi        ; execute exactly one instruction, then pause
(gdb) stepi        ; another one
```

`stepi` (shorthand: `si`) — one instruction at a time. Use this when we need to see what a specific instruction does to registers.

> **Confusion clarified:** Always run `stepi` *after* `starti` before reading registers. `starti` pauses *before* the first instruction runs. If we read a register before `stepi`, we see the pre-execution value — not the result of the instruction you care about.

---

### Reading Register Values

```gdb
(gdb) starti
(gdb) stepi
(gdb) print $rdi       ; print current value of rdi (in decimal)
(gdb) info registers   ; print ALL registers at once
```

`$` prefix means "register" in GDB. `print $rdi`, `print $rax`, etc.

---

### Examining Memory

The `x` command examines memory. Format: `x/[count][format] address`

```gdb
(gdb) x/d $rsp         ; read value at rsp, show as decimal
(gdb) x/x $rsp         ; read value at rsp, show as hex
(gdb) x/a $rsp+16      ; read value at rsp+16, show as address
(gdb) x/s 0x7ffc001c   ; read string starting at this address
```

| Format flag | Meaning |
|---|---|
| `/d` | decimal |
| `/x` | hexadecimal |
| `/a` | address (hex, formatted nicely) |
| `/s` | string (reads until null terminator) |
| `/i` | instruction (disassemble at that address) |

---

### Examining Stack Pointers

To find what `argv[1]` points to:

```gdb
# Step 1: find the pointer at rsp+16 (argv[1])
(gdb) x/a $rsp+16
# Output: 0x7ffc001c4750

# Step 2: follow the pointer to see the string
(gdb) x/s 0x7ffc001c4750
# Output: "hello"
```

Full flow:
```
starti → stepi → x/a $rsp+16 → x/s <address from previous output>
```

---

### Cooperative Debugging with int3

Instead of stopping a program externally with `starti`, the **program itself** can signal the debugger to stop using the `int3` instruction. This is a **software breakpoint**.

- `int3` tells a connected debugger: "pause here"
- In GDB, use `run` instead of `starti` — the program runs until it hits `int3`

```asm
.intel_syntax noprefix
.global _start
_start:
    mov rdi, 1337
    int3             ; pause here if a debugger is attached
    mov rax, 60
    syscall
```

```bash
echo -e ".intel_syntax noprefix\n.global _start\n_start:\n mov rdi, 1337\n int3\n syscall" > p.s
as -o p.o p.s && ld -o p p.o
gdb ./p
(gdb) run          ; runs until int3, then pauses
(gdb) print $rdi   ; → 1337
```

> `int3` is exactly what GDB sets when you use `break` — it patches a single byte (`0xcc`) into the binary at that location. Now you know what's happening under the hood.

---

### Running with Arguments

Outside GDB:
```bash
./program hello
```

Inside GDB:
```gdb
(gdb) run hello
# or shorthand:
(gdb) r hello
```

`run` + arguments → they become `argv[1]`, `argv[2]`, etc., exactly as they would outside GDB.

---

## Typical GDB Workflows

These are the exact sequences you'll use for pwn.college challenges:

### Workflow A: Read a censored register value

```
gdb ./binary → starti → stepi → print $rdi → quit
```

### Workflow B: Find a value in memory

```
gdb ./binary → starti → stepi → x/d $rsp → quit
```

### Workflow C: Follow a string pointer

```
gdb ./binary → starti → stepi → x/a $rsp+16 → x/s <address> → quit
```

### Workflow D: Program stops itself (int3)

```
gdb ./binary → run → (pauses at int3) → print $rdi → quit
```

### Workflow E: Run with arguments

```
gdb ./binary → run argument → (pauses at int3 or end) → examine state → quit
```

---

## 🔐 Security Context — Why This Matters for Hacking

**Reverse engineering** is the skill of understanding a binary without source code. `objdump` and GDB are the entry-level tools. Professional reverse engineers use Ghidra (free, by the NSA) or IDA Pro, but they're all doing the same thing — disassembling and tracing execution.

**Every CTF pwn challenge starts here.** We get a binary, no source. we `objdump` it to understand the structure. We `strace` it to see what syscalls it makes. We run it in GDB to inspect register/memory values at specific points. These are not optional — they're the workflow.

**`strace` catches programs trying to hide.** Malware that exfiltrates data has to make syscalls to send it — `strace` captures those. It's a first-pass triage tool in malware analysis.

**GDB's `x/s` is how you find strings in memory.** Passwords, keys, flags stored in memory can be read directly if you can pause execution at the right moment. This is a real technique — not just a CTF trick.

**`int3` / `0xcc` is the foundation of dynamic analysis.** When antivirus software, debugger-detection code, or anti-tamper systems look for debuggers, they often check for `0xcc` patches in memory. Understanding what `int3` is at the byte level is how we start thinking about anti-debugging bypass.

> **Connecting the dots:** You write assembly (`Assembly_Language_Intro.md`) → we understand memory (`Memory.md`) → we understand the stack (`The_Stack.md`) → now we can *read and trace* binaries we didn't write (this file) → next: coming soon.

---

## Quick Reference

| Tool / Command | What it does |
|---|---|
| `objdump -d -M intel ./binary` | Disassemble in Intel syntax |
| `strace ./binary` | Trace all syscalls |
| `strace -e write ./binary` | Only show specific syscalls |
| `gdb ./binary` | Launch GDB |
| `starti` | Start, pause before first instruction |
| `run` / `r` | Run until `int3`, breakpoint, or end |
| `stepi` / `si` | Execute one instruction |
| `print $rdi` | Print register value |
| `info registers` | Print all registers |
| `x/d $rsp` | Examine memory as decimal |
| `x/x $rsp` | Examine memory as hex |
| `x/a $rsp+16` | Examine memory as address |
| `x/s <addr>` | Examine memory as string |
| `x/i <addr>` | Examine memory as instruction |
| `quit` / `q` | Exit GDB |
| `int3` | Breakpoint instruction in assembly |
