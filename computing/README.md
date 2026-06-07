# Computing Fundamentals

> Notes built hands-on while doing pwn.college — Computer Architecture & Assembly → Computing 101.  
> Written while actually doing the challenges, not before.

---

## What This Section Covers

Low-level computing: how a CPU executes instructions, how memory is addressed, how the stack is laid out, how numbers are encoded, and how to read and trace compiled binaries. This is the foundation that every higher-level security concept — buffer overflows, shellcode, ROP, heap exploitation — is built on.

---

## Topics

Read in order. Each file builds directly on the one before it.

| # | File | What it covers | Security relevance |
|---|---|---|---|
| 1 | [Assembly Language Intro](./Assembly_Language_Intro.md) | CPU registers, Intel syntax, syscalls, build pipeline (`as` → `ld`) | Shellcode, reading exploits, understanding CVEs |
| 2 | [Memory](./Memory.md) | Memory addressing, pointer chains, LEA, endianness, dereferencing | Buffer overflows, heap exploitation, ASLR bypass |
| 3 | [The Stack](./The_Stack.md) | Stack layout, `rsp`, `argc`/`argv`, `pop`, stack offsets | Return address overwrite, ROP chains |
| 4 | [Nibbling on Numbers](./Nibbling_on_Numbers.md) | Binary, unsigned/signed decimal, two's complement, hex conversion | Reading GDB output, memory dumps, objdump values |
| 5 | [Software Introspection](./Software_Introspection.md) | `objdump`, `strace`, GDB (`starti`, `stepi`, `x/`, `print`) | Reverse engineering, malware analysis, CTF pwn |
| 6 | [File Descriptors & I/O](./File_Descriptors_IO.md) | `write`, `read`, `open` syscalls, FDs, null-terminated strings | Shellcode, fd hijacking, file read exploits |
| 7 | [Control Flow](./Control_Flow.md) | `cmp`, flags, `jne`, labels, jump tables, loops | Password bypasses, timing attacks, ROP, control flow hijacking |
| 8 | [Assembly Assortment](./Assembly_Assortment.md) | Reverse calculation (add, sub, XOR), `.rodata` string extraction | Hardcoded password cracking, XOR obfuscation, binary reversing |

---

## The Learning Path

```
Assembly Language Intro    ← vocabulary: registers, syscalls, build pipeline
        ↓
      Memory               ← reach into RAM: addressing, pointers, dereferencing
        ↓
    The Stack              ← program launch layout: argc, argv, rsp offsets
        ↓
Nibbling on Numbers        ← read raw values: binary, hex, signed/unsigned
        ↓
Software Introspection     ← read any binary: objdump, strace, GDB
        ↓
  File Descriptors & I/O   ← do real things: write, read, open files
        ↓
    Control Flow           ← make decisions: cmp, jumps, loops
        ↓
    (coming next)
```

Each topic uses the vocabulary of every topic above it. By Control Flow, all seven are in play at once.

---

## What Each File Contains

Every file follows the same structure so they're easy to use for revision:

- **"Where this fits"** — how it connects to the other files
- **Worked examples** — real assembly and bash you can run
- **"Confusion clarified"** callouts — things that actually tripped me up
- **Security Context** — why this matters for real hacking
- **Practice Checklist** — pwn.college challenges for that topic

---

## Skills Built

| Skill | Files |
|---|---|
| x86-64 assembly, Intel syntax, GAS | 1, 2, 3, 6, 7 |
| Linux syscall ABI (numbers, register convention) | 1, 6 |
| Memory addressing (direct, indirect, offset, pointer chains) | 2, 3 |
| Stack mechanics (`push`/`pop`, `rsp`, program entry layout) | 3 |
| Number systems (binary, hex, signed/unsigned, two's complement) | 4 |
| Binary analysis tools (`objdump`, `strace`, GDB) | 5 |
| File I/O in assembly (write, read, open, FDs) | 6 |
| Control flow (compare, conditional jumps, loops, jump tables) | 7 |

---

## Coming Next

As I progress through pwn.college, this section grows:

- [ ] Processes & File Descriptors (deeper)
- [ ] Shellcode
- [ ] Sandboxing (seccomp)
- [ ] Memory Errors (buffer overflows)
- [ ] ROP (Return-Oriented Programming)

---

## Source

Challenges from [pwn.college](https://pwn.college) — Computer Architecture & Assembly + Computing 101 modules.  
Personal lab: Kali Linux.  
See the [root README](../README.md) for the full repo overview.
