# CPU Architecture & Assembly Language

> **Where this fits:** Every program you've ever run — Python, Chrome, a game — eventually becomes assembly. This is the bottom of the software stack. Understanding it means you can read, inspect, and manipulate programs at their lowest level.

---

## Table of Contents

- [What is a CPU?](#what-is-a-cpu)
- [CPU Architectures](#cpu-architectures)
- [Assembly Language Basics](#assembly-language-basics)
- [Registers](#registers)
- [Moving Data Between Registers](#moving-data-between-registers)
- [Syscalls](#syscalls-system-calls)
- [Exit Codes](#exit-codes)
- [Building an Executable](#building-an-executable)
- [Full Worked Example](#full-worked-example--exit-with-code-42)
- [Security Context](#-security-context--why-this-matters-for-hacking)
- [Practice Checklist](#practice-checklist)

---

## What is a CPU?

The CPU is the brain of the computer — it runs instructions one at a time, very fast.

You give it instructions using **assembly language**. Everything else (Python, C++, Java) eventually gets translated into assembly before the CPU touches it.

```
Your Python code
      ↓
   Compiled/interpreted
      ↓
  Assembly instructions
      ↓
   CPU executes
```

---

## CPU Architectures

Two dominate the world:

| Architecture | Made by | Where you'll see it |
|---|---|---|
| **x86-64** | Intel / AMD | Laptops, desktops, servers — most CTF/pwn challenges |
| **ARM** | ARM Holdings | Phones, Raspberry Pi, Apple Silicon Macs |

> **For pwn.college and most CTFs:** we're working in x86-64. That's what these notes cover.

---

## Assembly Language Basics

Assembly has two parts:

- **Operations** — the action (`mov`, `add`, `sub`, `syscall`)
- **Operands** — what the action works on (registers, values, memory addresses)

```asm
mov  rdi, 42     ; operation = mov, operands = rdi and 42
add  rax, rbx    ; operation = add, operands = rax and rbx
```

Assembly is a human-readable version of the raw binary (machine code) the CPU actually runs. The assembler (`as`) converts it to binary. You write assembly; the CPU runs binary.

---

## Registers

Registers are tiny, ultra-fast storage slots **inside the CPU itself** — not RAM.

- Only **~16 general-purpose registers** in x86-64
- Each holds **8 bytes (64 bits)** of data
- When the CPU needs to do anything, the data has to be in a register first

### The registers you'll use most

| Register | Role |
|---|---|
| `rax` | General purpose + **syscall number** |
| `rdi` | General purpose + **1st syscall argument** / exit code |
| `rsi` | General purpose + **2nd syscall argument** |
| `rdx` | General purpose + **3rd syscall argument** |
| `rsp` | **Stack pointer** — top of the stack (see `The_Stack.md`) |
| `rip` | **Instruction pointer** — address of the next instruction to run |
| `rbx`, `rcx`, `r8`–`r15` | General purpose scratch registers |

> **Pattern to notice:** The CPU calling convention (how arguments are passed) maps directly to registers. `rdi` is always argument #1. This is why you always put the exit code in `rdi` — exit is a syscall that takes one argument.

---

## Moving Data Between Registers

`mov` is the most common instruction. It copies a value — like copy-paste, not cut-paste. The source doesn't change.

```asm
mov rax, 42        ; rax = 42  (load a number)
mov rdi, rax       ; rdi = rax (copy from register to register)
mov rdi, rsi       ; rdi = rsi (rsi unchanged, rdi now has rsi's value)
```

### Practical example — forward rsi into rdi

If `rsi` holds a value you need as your exit code:

```asm
mov rdi, rsi       ; copy rsi's value into rdi
mov rax, 60        ; syscall 60 = exit
syscall
```

Try it:

```bash
echo -e ".intel_syntax noprefix\n.global _start\n_start:\n mov rdi, rsi\n mov rax, 60\n syscall" > p.s
as -o p.o p.s      # change the program to binary not readable
ld -o p p.o        # link to make it readable as we studied it before
./p
echo $?            # whatever value rsi held at launch
```

> The exit code depends on what the OS put in `rsi` at program start — which varies. That's intentional in pwn.college challenges.

---

## Syscalls (System Calls)

A syscall is how your program asks the **operating system** to do something — write to the screen, open a file, exit, etc.

```
Your program  →  syscall  →  Linux kernel  →  does the thing
```

How to make a syscall in x86-64:

1. Put the **syscall number** in `rax`
2. Put **arguments** in `rdi`, `rsi`, `rdx` (in that order)
3. Execute the `syscall` instruction

```asm
mov rax, 42    ; which syscall (42 = sendfile, just as an example)
mov rdi, 1     ; first argument
syscall        ; go
```

Linux has ~330 syscalls. The ones you'll use constantly:

| Number | Name | What it does |
|---|---|---|
| `0` | `read` | Read from a file/stdin |
| `1` | `write` | Write to a file/stdout |
| `60` | `exit` | End the program |
| `59` | `execve` | Execute another program |

> **Full list:** `/usr/include/asm/unistd_64.h` on any Linux system, or search "linux syscall table x86_64".

---

## Exit Codes

Exit codes are how a program reports its status back to the shell (or whatever ran it).

- `0` = success
- anything else = some kind of result or error

```asm
mov rax, 60    ; syscall: exit
mov rdi, 0     ; exit code: 0 (success)
syscall
```

Check the exit code in bash:

```bash
./program
echo $?        ; prints the exit code of the last command
```

> **In pwn.college:** many challenges want you to exit with a specific secret value. That's why we will be spending so much time getting the right value into `rdi` before calling exit.

---

## Building an Executable

Three steps every time:

```
Write (.s)  →  Assemble (.o)  →  Link (executable)
```

### Step 1 — Write your assembly file

Save as `program.s`. Always start with these two lines:

```asm
.intel_syntax noprefix    ; use Intel syntax (not AT&T)
.global _start            ; tell the linker where the program starts
_start:
    ; your code here
```

### Step 2 — Assemble (source → object file)

```bash
as -o program.o program.s
```

`as` (GNU Assembler) converts your text into binary machine code. The `.o` file is not yet runnable — it's an intermediate.

### Step 3 — Link (object file → executable)

```bash
ld -o program program.o
```

`ld` (Linker) packages the object file into a real executable. In bigger projects it combines multiple `.o` files and resolves references between them.

### Run it

```bash
./program
echo $?
```

---

## Full Worked Example — Exit with Code 42

**Goal:** write a program that exits with the value `42`.

```asm
.intel_syntax noprefix   ; (1) tell assembler: Intel syntax
.global _start           ; (2) expose _start to the linker
_start:                  ; (3) entry point label
    mov rdi, 42          ; (4) exit code = 42
    mov rax, 60          ; (5) syscall number = exit
    syscall              ; (6) trigger the syscall
```

Line by line:

| Line | What it does |
|---|---|
| `.intel_syntax noprefix` | Directive to `as` — use Intel syntax. Without this you'd need AT&T syntax (`movq $42, %rdi`) which is harder to read. |
| `.global _start` | Makes `_start` visible to `ld`. The linker needs this to know the entry point. |
| `_start:` | Label — a named address. The linker sets this as the program's start. |
| `mov rdi, 42` | Load 42 into `rdi`. For the exit syscall, `rdi` = the exit code. |
| `mov rax, 60` | Load 60 into `rax`. `rax` always holds the syscall number. 60 = exit on Linux x86-64. |
| `syscall` | Trigger. CPU looks at `rax` (which syscall), `rdi` (first arg), and hands control to the kernel. |

Build and run:

```bash
as -o program.o program.s
ld -o program program.o
./program
echo $?     # → 42
```

### The full flow at a glance

```
program.s  →(as)→  program.o  →(ld)→  program  →(./program)→  exits 42
 source text      object file       executable
```

---

## 🔐 Security Context — Why This Matters for Hacking

Understanding assembly isn't just academic. Here's how it connects to real attacks:

**Shellcode** is hand-written assembly that gets injected into a running process and executed. Classic buffer overflow exploits end with shellcode running as the target program. You can't write shellcode without knowing registers, syscalls, and the build pipeline.

**Reading exploits and CVEs** — when researchers publish proof-of-concept exploits, they often include raw assembly or disassembled code. Fluency here means you can actually understand what an exploit does rather than just running it blind.

**Reverse engineering** — malware analysts and CTF players constantly read disassembled assembly to understand programs that have no source code. The `objdump`/GDB skills in `Software_Introspection.md` are the direct tools; knowing what `mov rdi, rax` means is the prerequisite.

**The syscall interface is the attack surface** — every interaction a program has with the OS goes through syscalls. Understanding what `execve`, `read`, `write`, and `mmap` do in assembly is foundational for understanding privilege escalation, sandbox escapes, and ROP chains.

> **Connecting the dots:** Assembly → Memory (pointers, stack) → Software Introspection (read binaries) → exploitation techniques (buffer overflows, shellcode, ROP). We're building from the bottom up.

---

## Practice Checklist

- [ ] Exit with a hardcoded value (e.g. `42`)
- [ ] Exit with a value from `rsi` (forwarding registers)
- [ ] Use each of the three build steps manually (`as`, `ld`, `./`)
- [ ] Verify exit codes with `echo $?`
- [ ] Write a program that uses a different syscall (e.g. syscall 1 = write)
- [ ] Read the Linux syscall table and find 5 syscalls you recognize
