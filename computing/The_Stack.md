# The Stack

> **Where this fits:** The stack is a region of memory our program gets for free — already set up when `_start` runs. The OS fills it with information about how we program was launched (argument count, argument values). Understanding the stack is how we read program arguments in assembly, and it's foundational for understanding function calls and buffer overflows.

---

## Table of Contents

- [What is the Stack?](#what-is-the-stack)
- [Stack Offsets](#stack-offsets)
- [Program Arguments on the Stack](#program-arguments-on-the-stack)
- [Popping from the Stack](#popping-from-the-stack)
- [Security Context](#-security-context--why-this-matters-for-hacking)
- [Practice Checklist](#practice-checklist)

---

## What is the Stack?

The stack is a **pre-allocated region of memory** — no `malloc`, no setup. It's just there when our program starts.

- Grows **downward** (toward lower addresses) as data is added
- The `rsp` register (**Stack Pointer**) always points to the **top** (the current lowest address in use)
- The OS fills the bottom of the initial stack with program launch data before your code runs

```
High addresses
┌──────────────────┐
│  environment vars │
├──────────────────┤
│  argv strings     │  ← actual text of arguments ("./program", "hello", etc.)
├──────────────────┤
│  NULL             │  ← end of argv array
├──────────────────┤
│  argv[n]          │
│  ...              │
│  argv[1]          │  ← pointers to argument strings
│  argv[0]          │
├──────────────────┤
│  argc             │  ← rsp points here at _start
└──────────────────┘
Low addresses (rsp)
```

### Read `argc` and exit with it

```asm
.intel_syntax noprefix
.global _start
_start:
    mov rdi, qword ptr [rsp]   ; load argc from top of stack
    mov rax, 60                ; syscall: exit
    syscall
```

```bash
echo -e ".intel_syntax noprefix\n.global _start\n_start:\n mov rdi, qword ptr [rsp]\n mov rax, 60\n syscall" > p.s
as -o p.o p.s && ld -o p p.o && ./p
echo $?    # number of arguments including program name
```

> **`qword ptr`** explicitly denotes a 64-bit (8-byte) memory read. Optional in most cases but good practice — it makes our intent clear.

---

## Stack Offsets

The stack at program entry is laid out in 8-byte slots. To access a slot that isn't the top, add an offset to `rsp`.

```
[rsp]      → slot 0  (first thing)
[rsp + 8]  → slot 1  (8 bytes in)
[rsp + 16] → slot 2  (16 bytes in)
[rsp + 24] → slot 3  (24 bytes in)
```

Why 8? Because each pointer/value is 8 bytes on x86-64 (64 bits = 8 bytes).

### Read the value at `[rsp+8]`

```asm
.intel_syntax noprefix
.global _start
_start:
    mov rdi, qword ptr [rsp+8]  ; load the value 8 bytes into the stack
    mov rax, 60
    syscall
```

```bash
echo -e ".intel_syntax noprefix\n.global _start\n_start:\n mov rdi, qword ptr [rsp+8]\n mov rax, 60\n syscall" > p.s
as -o p.o p.s && ld -o p p.o && /challenge/check p
```

---

## Program Arguments on the Stack

At `_start` on x86-64 Linux, the stack layout is always:

| Offset | Contents | What it is |
|---|---|---|
| `[rsp]` | `argc` | number of arguments (always ≥ 1) |
| `[rsp+8]` | `argv[0]` | **pointer** to the program name string |
| `[rsp+16]` | `argv[1]` | **pointer** to the first argument string |
| `[rsp+24]` | `argv[2]` | **pointer** to the second argument string |
| `[rsp + 8*argc + 8]` | `NULL` | marks the end of the argv array |

**Important:** `argv[0]`, `argv[1]`, etc. are **pointers** — they hold an address, not the actual string data. To get the string (or integer value) we have to dereference the pointer.

```
[rsp+16] → address 0x7ffc001c4750
0x7ffc001c4750 → "hello"   ← the actual string
```

### Exit with the integer value of `argv[1]`

```asm
.intel_syntax noprefix
.global _start
_start:
    mov rdi, [rsp+16]   ; load the pointer to argv[1]
    mov rdi, [rdi]      ; dereference: follow the pointer to get the actual value
    mov rax, 60
    syscall
```

```bash
echo -e ".intel_syntax noprefix\n.global _start\n_start:\n mov rdi, [rsp+16]\n mov rdi, [rdi]\n mov rax,60\n syscall" > p.s
as -o p.o p.s && ld -o p p.o && /challenge/check p
```

> **Two-step pattern:**
> 1. `mov rdi, [rsp+16]` → get the address (pointer)
> 2. `mov rdi, [rdi]` → follow the address to get the value
>
> This is exactly the same double-dereference from `Memory.md` — just applied to the stack.

---

## Popping from the Stack

`pop` is a single instruction that does two things atomically:

1. Reads the value at `[rsp]` into the destination register
2. Advances `rsp` by 8 (moves the stack pointer to the next slot)

It's shorthand for:
```asm
mov rdi, [rsp]
add rsp, 8
```

### Memory diagram

**Before `pop rdi`:**

```
Address   │ Value
──────────┼──────────────────────────
1337000   │  3          ← rsp points here (argc = 3)
1337008   │  ???
1337016   │  ???

Register  │ Value
──────────┼──────────────
rsp       │ 1337000
rdi       │ 0
```

**After `pop rdi`:**

```
Address   │ Value
──────────┼──────────────────────────
1337000   │  3          (still there, stack doesn't erase)
1337008   │  ???        ← rsp now points here
1337016   │  ???

Register  │ Value
──────────┼──────────────
rsp       │ 1337008     (advanced by 8)
rdi       │ 3           (received the popped value)
```

> The data at `1337000` isn't erased — `rsp` just moved past it. It will be overwritten eventually, but it's still there until something writes over it. This matters in exploitation.

### Example: pop `argc` and exit with it

```asm
.intel_syntax noprefix
.global _start
_start:
    pop rdi          ; pop argc into rdi; rsp advances by 8
    mov rax, 60
    syscall
```

```bash
echo -e ".intel_syntax noprefix\n.global _start\n_start:\n pop rdi\n mov rax,60\n syscall" > p.s
as -o p.o p.s && ld -o p p.o && /challenge/check p
```

---

## Stack Layout Summary

```
Before any pops at _start:

rsp → [argc]          [rsp]
      [argv[0] ptr]   [rsp + 8]    → "/path/to/program"
      [argv[1] ptr]   [rsp + 16]   → "first_arg"
      [argv[2] ptr]   [rsp + 24]   → "second_arg"
      [NULL]          [rsp + 8*argc + 8]

After pop rdi:

      [argc]          (rsp moved past this — value now in rdi)
rsp → [argv[0] ptr]   [rsp]        ← rsp advanced
      [argv[1] ptr]   [rsp + 8]
      ...
```

---

## 🔐 Security Context — Why This Matters for Hacking

**Stack-based buffer overflows** are the classic vulnerability class. When a function puts a local variable on the stack and you write more data than it can hold, we overflow into adjacent stack memory — including the **return address**, which controls where execution goes when the function returns. Knowing the stack layout is mandatory for exploiting this.

**Return addresses live on the stack.** When you call a function (`call foo`), the CPU pushes the return address (where to come back to) onto the stack. When the function returns (`ret`), it pops that address and jumps to it. Overwriting that address = redirecting execution anywhere we want.

**`pop` chains are at the heart of ROP (Return-Oriented Programming).** When we can't inject shellcode (because of NX/no-execute protections), attackers chain together small existing code snippets that end in `ret`. Each snippet is called a "gadget." Most useful gadgets look like `pop rdi ; ret` — they set a register from the stack, then return to the next gadget. Understanding `pop` and `rsp` behavior is the mechanical foundation of ROP.

**The `argv` pointer chain is a real attack surface.** Programs that pass `argv[1]` to `strcpy()` or `sprintf()` without length checks are vulnerable to command-line buffer overflows. We now know exactly how those arguments are laid out in memory.

> **Connecting the dots:** Memory addressing → Stack layout (this file) → Function call mechanics → Return address overwrite → ROP chains. Each file builds directly on the last.

---

## Quick Reference

| Instruction / Syntax | Meaning |
|---|---|
| `rsp` | Stack pointer — always points to top of stack |
| `[rsp]` | Value at the top of the stack |
| `[rsp+8]` | Value 8 bytes below the top |
| `[rsp+16]` | Value 16 bytes below the top |
| `qword ptr [rsp]` | Explicit 64-bit read |
| `pop rdi` | `rdi = [rsp]`, then `rsp += 8` |
| `push rax` | `rsp -= 8`, then `[rsp] = rax` |
| `argc` | Always at `[rsp]` on program entry |
| `argv[0]` | Pointer at `[rsp+8]` (program name) |
| `argv[1]` | Pointer at `[rsp+16]` (first argument) |

---

## Practice Checklist

- [ ] Read `argc` directly with `mov rdi, [rsp]`
- [ ] Read `argc` using `pop rdi`
- [ ] Read `argv[0]` (the program name pointer) at `[rsp+8]`
- [ ] Read `argv[1]` at `[rsp+16]` and dereference it
- [ ] Run with different numbers of arguments, verify argc changes
- [ ] Trace `rsp` value before and after a `pop` in GDB


Part 2 compe after we are done with other other section from 1- 7. As it contain advance topics:
we now know the basic so lets delve onto advance topicsHow does it actually get laid out? Why does it "grow downwards," and what does that mean for the data you can reach? And how does the way your program was launched shape the addresses of everything on it?

Rwaching into the callers Frame- mostly about accessoing lower stack memory in the cjallenge
In addition to storing scratch data and return addresses, the stack stores the local variables of functions: data they use for functionality that's not necessarily needed by other functions of a program. In security situations where a hacker gets ``code execution'' inside a process, these variables are an open book: there is nothing preventing code in a process from reading data from all over the stack!
Imagine the stack as a pile of sticky notes. When one function calls another, it writes a "return address" on a new note and puts it on top of the pile. Then the called function can add its own notes (local variables) on top.

But in this challenge, your solve function has no arguments—it's just called. The caller function, before calling you, stored the secret flag in its own local variables (like its own sticky notes). Those notes are still on the stack, just below where your function starts (at higher memory addresses).

Because the stack is just one big block of memory, and your code can read anywhere in that block (there's no protection stopping you), you can simply look down from where you are (positive offsets from rsp) and find the flag. You don't need to be passed anything—you just reach into the caller's "frame" and grab the bytes.

Key points to remember:

The stack grows downward (toward smaller addresses) when you push. So a "higher" address means older data.

When solve starts, rsp points to the return address (8 bytes). The caller's locals start right after that.

By reading memory at [rsp + 0x40] (for example), you can directly read the flag that the caller left behind.

That's the vulnerability: any code running in a process can read the entire stack—nothing hides local variables from other functions on the same stack.
Chqallenge:
ubuntu@the-stack-revisited~reaching-into-the-callers-frame:~$ nano solve.s
ubuntu@the-stack-revisited~reaching-into-the-callers-frame:~$ as -o solve.o sol
ve.s
ubuntu@the-stack-revisited~reaching-into-the-callers-frame:~$ ld -shared -o sol
ve.so solve.o
ubuntu@the-stack-revisited~reaching-into-the-callers-frame:~$ /challenge/checksolve| solve.s
bash: /challenge/checksolve: No such file or directory
bash: solve.s: command not found
ubuntu@the-stack-revisited~reaching-into-the-callers-frame:~$ /challenge/check solve.s
Check failed:

Expected an ELF 64-bit shared object, but solve.s is:
  ASCII text

Build your library like this:
    as --64 your-solve.s -o your-solve.o
    ld -shared your-solve.o -o your-solve.so

ubuntu@the-stack-revisited~reaching-into-the-callers-frame:~$ /challenge/check solve
Check failed:

File solve not found.
ubuntu@the-stack-revisited~reaching-into-the-callers-frame:~$ /challenge/check solve.so

Let's see if your solve reads the flag out of the caller's frame and writes it 
to stdout...

hacker@the-stack-revisited~reaching-into-the-callers-frame:/home/hacker$ /challenge/harness /tmp/your-program.so

[harness] loading shared library /tmp/your-program.so ...
[harness] resolving `solve` symbol ...
[harness] found solve at 0x74c847b22000
[harness] reading 128 bytes of flag from stdin ...
[harness] calling caller(solve, flag_buf) --- caller stashes the flag in its local frame at [rsp+0x40] (from solve's view) and then calls into your solve
[harness] your `solve` should `write` 128 bytes from [rsp+0x40] to stdout:
pwn.college{wofDtHcFWzOwF1y2wUQi01PrUi4.0FO1kDNywyM2cTM3EzW}====[harness] caller() returned. (If your write was correct, the flag printed above.)

If your solve is right, the flag is printed above!
ubuntu@the-stack-revisited~reaching-into-the-callers-frame:~$ cat solve.s
.intel_syntax noprefix
.global solve
solve:
 mov rsi, rsp
 add rsi, 0x40
 mov rax, 1
 mov rdi, 1
 mov rdx, 0x40
 syscall
 ret
