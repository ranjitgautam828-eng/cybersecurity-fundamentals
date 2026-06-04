# Computing Fundamentals

> Notes built from pwn.college's Computer Architecture & Assembly module -> Computing 101.  
> Everything here is hands-on — written while actually doing the challenges, not before.

---

## What This Section Covers

Low-level computing: how a CPU executes instructions, how memory is addressed, how the stack is laid out, and how to read and trace compiled binaries. This is the foundation that every higher-level security concept (buffer overflows, shellcode, ROP, heap exploitation) is built on.

---

## Topics

| File | What it covers | Security relevance |
|---|---|---|
| [Assembly Language Intro](./Assembly_Language_Intro.md) | CPU registers, Intel syntax, syscalls, build pipeline (`as` → `ld`) | Shellcode, reading exploits, understanding CVEs |
| [Memory](./Memory.md) | Memory addressing, pointer chains, LEA, endianness, dereferencing | Buffer overflows, heap exploitation, ASLR bypass |
| [The Stack](./The_Stack.md) | Stack layout, `rsp`, `argc`/`argv`, `pop`, stack offsets | Return address overwrite, ROP chains |
| [Software Introspection](./Software_Introspection.md) | `objdump`, `strace`, GDB (starti, stepi, x/, print) | Reverse engineering, malware analysis, CTF pwn |
| [Output_&_Input](./Output_&_Input.md) | write, read, open syscalls, FDs, null-terminated strings | Shellcode, fd hijacking, file read exploits |
---

## How to Read These Notes

They're designed to be read **in order** — each file builds on the last:

```
Assembly Language Intro
        ↓
      Memory
        ↓
    The Stack
        ↓
Software Introspection
        ↓
    (coming next)
```

Each file has:
- A **"Where this fits"** header — how it connects to the others
- **Worked examples** with real bash/assembly code you can run
- **"Confusion clarified"** callouts for things that tripped me up
- A **Security Context** section — why this matters for actual hacking
- A **Practice Checklist** — pwn.college challenges for that topic

---

## Skills Demonstrated

Working through this module builds:

- x86-64 assembly (Intel syntax, GAS assembler)
- Linux syscall ABI (syscall numbers, register convention)
- Memory addressing modes (direct, register-indirect, offset, pointer chains)
- Stack mechanics (layout at program entry, `push`/`pop`, `rsp` offsets)
- Binary analysis tools: `objdump`, `strace`, GDB

---

## Coming Next

As I progress through pwn.college, this section will grow. Planned additions:

- [ ] coming soon

---

## Source

All challenges from [pwn.college](https://pwn.college) — Computer Architecture module.  
Personal lab: Kali Linux.  
See the [root README](../README.md) for the full repo overview.
