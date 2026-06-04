# Output_&_Input

> **Where this fits:** You know how to exit a program using a syscall. Now you'll use syscalls to actually *do* things — print text to the screen, read input, and open files. Everything here uses the same syscall pattern from `Assembly_Language_Intro.md`, just with different syscall numbers and more registers filled in.

---

## Table of Contents

- [File Descriptors — the "where"](#file-descriptors--the-where)
- [Writing Output — syscall 1](#writing-output--syscall-1)
- [Writing Multiple Bytes](#writing-multiple-bytes)
- [Chaining Syscalls — write then exit](#chaining-syscalls--write-then-exit)
- [Reading Input — syscall 0](#reading-input--syscall-0)
- [Opening Files — syscall 2](#opening-files--syscall-2)
- [Hardcoding a Filename onto the Stack](#hardcoding-a-filename-onto-the-stack)
- [Full Example — open, read, write](#full-example--open-read-write)
- [Debugging Tips](#debugging-tips)
- [Security Context](#-security-context--why-this-matters-for-hacking)
- [Practice Checklist](#practice-checklist)

---

## File Descriptors — the "where"

Before writing or reading anything, you need to say *where* — to the screen? from the keyboard? from a file?

Linux answers this with **File Descriptors (FDs)** — small integers that represent an open channel.

Every process starts with three already open:

| FD | Name | What it is |
|---|---|---|
| `0` | **stdin** | Input — keyboard, or piped-in data |
| `1` | **stdout** | Normal output — what you see in the terminal |
| `2` | **stderr** | Error output — error messages |

When you `open()` a file, Linux gives you back a new FD (usually starting at 3). You use that number the same way as 0, 1, or 2 — pass it to `read`/`write` as the first argument.

> **Connecting back:** FDs are just integers — you store them in registers and pass them as syscall arguments, exactly like exit codes. The concept is the same; just a different number with a different meaning.

---

## Writing Output — syscall 1

The `write` syscall prints data to a file descriptor. Its signature:

```
write(file_descriptor, memory_address, number_of_bytes)
```

Three parameters → three registers (the Linux syscall calling convention):

| Register | Parameter | What to put in it |
|---|---|---|
| `rax` | syscall number | `1` (write) |
| `rdi` | file descriptor | `1` = stdout, `2` = stderr |
| `rsi` | memory address | where your data starts in memory |
| `rdx` | byte count | how many bytes to write |

> **rdi vs rdx confusion:** These look almost identical. Remember: **rdi** = the **i**nitial (first) parameter. **rdx** = the e**x**tra (third) parameter. You'll mix these up — everyone does at first. Just slow down and double-check.

### Example — write 1 byte from the stack to stdout

```asm
.intel_syntax noprefix
.global _start
_start:
    mov rax, 1          ; syscall 1 = write
    mov rdi, 1          ; fd 1 = stdout
    mov rsi, [rsp+16]   ; address of data (argv[1] pointer from the stack)
    mov rdx, 1          ; write 1 byte
    syscall
```

> **Why not pass data in a register?** Registers only hold 8 bytes at most. A syscall that took data via register could only print 8 characters at a time — terrible for anything real. Instead, `write` takes a *pointer* to memory where your data lives, plus a length. The kernel reads directly from your memory. This is why `rsi` holds an **address**, not the actual text.

---

## Writing Multiple Bytes

To write more than one character, just increase `rdx`:

```asm
mov rax, 1
mov rdi, 1
mov rsi, [rsp+16]   ; address of data
mov rdx, 13         ; write 13 bytes
syscall
```

The kernel reads `rdx` bytes starting from the address in `rsi`. That's it — same call, bigger count.

> **Performance note:** Every syscall is expensive — the CPU has to stop your program, switch into kernel mode, do the work, then switch back. This is why `write` takes a length instead of making you call it once per character. Fewer syscalls = faster program.

---

## Chaining Syscalls — write then exit

After writing, you still need to exit cleanly. Just put the exit syscall after:

```asm
.intel_syntax noprefix
.global _start
_start:
    ; --- write ---
    mov rax, 1
    mov rdi, 1
    mov rsi, [rsp+16]
    mov rdx, 1
    syscall

    ; --- exit ---
    mov rax, 60
    mov rdi, 0          ; exit code 0 = success
    syscall
```

> **Order matters.** The CPU runs instructions top to bottom. If you put exit before write, the program ends before printing anything. Always think: what has to happen first?

---

## Reading Input — syscall 0

`read` is the mirror of `write`. Same structure, syscall number 0:

```
read(file_descriptor, memory_address, max_bytes_to_read)
```

| Register | Value | Meaning |
|---|---|---|
| `rax` | `0` | syscall 0 = read |
| `rdi` | `0` | fd 0 = stdin |
| `rsi` | address | where in memory to store the input |
| `rdx` | `64` | read up to 64 bytes |

```asm
mov rax, 0
mov rdi, 0          ; read from stdin
mov rsi, rsp        ; store data at the top of the stack
mov rdx, 64         ; read up to 64 bytes
syscall
```

After this syscall, the bytes the user typed are now sitting in memory starting at `rsp`. You can then pass that same address to `write` to echo them back.

### What the memory looks like after reading "HELLO"

```
Address       │ Value (hex) │ ASCII
──────────────┼─────────────┼──────
rsp + 0       │ 0x48        │ H
rsp + 1       │ 0x45        │ E
rsp + 2       │ 0x4c        │ L
rsp + 3       │ 0x4c        │ L
rsp + 4       │ 0x4f        │ O
```

Characters are stored as their ASCII values — just numbers. `'H'` = `0x48` = `72` in decimal. Same byte, three different ways to write it.

> **Confusion clarified:** When reading, `rdi` = `0` (stdin), not `1`. It's easy to write `mov rdi, 1` by habit from write. Always check: am I reading (fd 0) or writing (fd 1)?

---

## Opening Files — syscall 2

stdin, stdout, stderr are always open. But to read a file from disk, you need to open it first. That's `open`:

```
open(filename_pointer, flags)
```

| Register | Value | Meaning |
|---|---|---|
| `rax` | `2` | syscall 2 = open |
| `rdi` | address | pointer to the filename string in memory |
| `rsi` | `0` | flags: `0` = read-only |

```asm
mov rax, 2
mov rdi, [rsp+16]   ; pointer to filename (e.g. argv[1] = "/flag")
mov rsi, 0          ; read-only
syscall
; rax now holds the new file descriptor (e.g. 3)
```

**Critical:** after `open` returns, `rax` holds the new FD. You must save it before you overwrite `rax` with the next syscall number:

```asm
syscall             ; open() — rax = new fd (e.g. 3)
mov rdi, rax        ; save the fd into rdi NOW, before you lose it
```

If you forget this and immediately do `mov rax, 0` for read, you've lost the fd and can't read the file.

> **Connecting back:** The fd returned by `open` is just an integer — same as the hardcoded `0`, `1`, `2` you already know. The only difference is that `open` creates it dynamically and hands it to you in `rax`.

---

## Hardcoding a Filename onto the Stack

Sometimes the filename isn't passed as an argument — you need to write it directly into memory yourself. The "hacker" way: build the string byte by byte on the stack.

```asm
mov BYTE PTR [rsp],   '/'
mov BYTE PTR [rsp+1], 'f'
mov BYTE PTR [rsp+2], 'l'
mov BYTE PTR [rsp+3], 'a'
mov BYTE PTR [rsp+4], 'g'
mov BYTE PTR [rsp+5], 0     ; null terminator — marks end of string
```

Three things to understand here:

**`BYTE PTR`** — a size directive. Tells the assembler: write exactly 1 byte here. Without it, the assembler doesn't know if you mean 1, 2, 4, or 8 bytes.

**Single quotes** — `'f'` is just a readable way to write the ASCII value of `f`, which is `0x66`. The assembler converts it. Same thing.

**The null byte (`0`)** — Linux reads a string by starting at the pointer and stopping at the first `0` byte. This is called a **null-terminated string**. Without it, `open` would keep reading past "flag" into whatever garbage is next on the stack and try to open a file with a nonsense name.

After writing the bytes, pass `rsp` (the start of your string) to `open` via `rdi`:

```asm
mov rdi, rsp        ; rdi = pointer to "/flag\0" we just wrote
mov rax, 2
mov rsi, 0
syscall
```

---

## Full Example — open, read, write

This is the complete pattern for reading a file and printing its contents. The order of syscalls matters — each one feeds into the next.

```asm
.intel_syntax noprefix
.global _start

_start:
    ; Step 1: write "/flag\0" onto the stack
    mov BYTE PTR [rsp],   '/'
    mov BYTE PTR [rsp+1], 'f'
    mov BYTE PTR [rsp+2], 'l'
    mov BYTE PTR [rsp+3], 'a'
    mov BYTE PTR [rsp+4], 'g'
    mov BYTE PTR [rsp+5], 0

    ; Step 2: open("/flag", 0) → fd in rax
    mov rdi, rsp        ; pointer to our "/flag" string
    mov rax, 2
    mov rsi, 0
    syscall

    ; Step 3: save the fd, then read from the file
    mov rdi, rax        ; save fd (e.g. 3) before overwriting rax
    mov rax, 0          ; syscall 0 = read
    mov rsi, rsp        ; store file contents at rsp (overwrites the filename — that's fine)
    mov rdx, 64         ; read up to 64 bytes
    syscall

    ; Step 4: write the contents to stdout
    mov rax, 1
    mov rdi, 1          ; stdout
    mov rsi, rsp        ; same address — contents are here now
    mov rdx, 64
    syscall

    ; Step 5: exit cleanly
    mov rax, 60
    mov rdi, 0
    syscall
```

### The data flow, step by step

```
Stack memory (rsp)
  → we write "/flag\0" here           (Step 1)
  → open() reads it, returns fd=3     (Step 2)
  → read() fills rsp with file data   (Step 3) ← overwrites "/flag", that's fine
  → write() prints from rsp           (Step 4)
  → exit                              (Step 5)
```

> **Personal note:** Make sure your terminal is connected to the right machine when running these. It happened to me that I was connected to a different device and the challenge wasn't working — the machine running the code was somewhere else entirely.

---

## Debugging Tips

**`strace` first.** If something's wrong, run `strace ./your_binary` and read what each syscall got as arguments and returned. If `open` returns `-1`, your string pointer or encoding is wrong.

```bash
strace ./p
```

**Check your string in GDB.** If `open` is failing, pause at that instruction and inspect what string is actually at `rsp`:

```bash
gdb ./p
(gdb) starti
(gdb) stepi     # step through until after you've written the filename bytes
(gdb) x/s $rsp  # print the string at rsp — does it show "/flag"?
```

**Common mistakes:**

| Mistake | Symptom | Fix |
|---|---|---|
| Forgot null terminator | `open` returns -1, strange filename | Add `mov BYTE PTR [rsp+5], 0` |
| Used `rdi=1` for read | Reading from stdout instead of stdin | Read uses `rdi=0` |
| Overwrote `rax` before saving fd | `read` uses wrong fd | `mov rdi, rax` immediately after `open` syscall |
| Wrong terminal/machine | Challenge never works | Check which SSH session you're in |

---

## 🔐 Security Context — Why This Matters for Hacking

**This is the exact pattern used in shellcode.** Real shellcode for privilege escalation often does exactly this sequence: open `/etc/shadow` or `/flag`, read its contents into a buffer on the stack, write them to stdout (or over a socket). You just built that from scratch.

**File descriptor hijacking** is a real attack. If a privileged process opens a sensitive file and you can manipulate which fd number it uses — or if it inherits your open fd — you can read data you shouldn't have access to. Understanding what fds are and how `open` assigns them is the foundation.

**Understanding syscall arguments is how you read exploits.** When you see a shellcode payload in a CTF writeup or CVE, it's often a sequence of `mov` instructions setting up `rax`, `rdi`, `rsi`, `rdx` for exactly these syscalls. You can now read those directly.

**Null termination bugs are a real vulnerability class.** If a program builds a string in memory and forgets the null byte, `open`/`read`/`write` will read past the intended boundary into adjacent memory — potentially leaking secrets or crashing. You just felt firsthand why the null byte matters.

> **Connecting the dots:** Registers + syscalls (`Assembly_Language_Intro.md`) → memory addressing to pass pointers (`Memory.md`) → stack as scratch space to store strings (`The_Stack.md`) → introspection to debug when it breaks (`Software_Introspection.md`) → **this file**: putting it all together to read files, print output, and chain syscalls. This is the first complete, useful program you've built.

---

## Practice Checklist

- [ ] Write 1 byte to stdout using syscall 1
- [ ] Write multiple bytes to stdout (change `rdx`)
- [ ] Chain write + exit without crashing
- [ ] Read from stdin into the stack, then write it back out (echo program)
- [ ] Open a file passed as `argv[1]`, read it, print contents
- [ ] Hardcode `/flag` byte by byte onto the stack with null terminator
- [ ] Use `strace` to verify your syscall arguments are correct
- [ ] Use `x/s $rsp` in GDB to verify your string is correctly built
