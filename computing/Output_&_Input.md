# Output_&_Input

> **Where this fits:** We know syscalls now — exit was the first one. This section is about actually *doing stuff* with syscalls: printing to screen, reading input, opening files. Same pattern, just more registers to fill and more syscalls chained together.

---

## Table of Contents

- [File Descriptors — what even is that](#file-descriptors--what-even-is-that)
- [Writing to the Screen — syscall 1](#writing-to-the-screen--syscall-1)
- [Writing More Than One Character](#writing-more-than-one-character)
- [Chaining — write then exit](#chaining--write-then-exit)
- [Reading Input — syscall 0](#reading-input--syscall-0)
- [Opening a File — syscall 2](#opening-a-file--syscall-2)
- [Hardcoding the Filename on the Stack](#hardcoding-the-filename-on-the-stack)
- [Full Program — open, read, write, exit](#full-program--open-read-write-exit)
- [Debugging](#debugging)
- [Security Context](#-security-context)
- [Practice Checklist](#practice-checklist)

---

## File Descriptors — what even is that

Every process starts with three "channels" already open. Linux calls them **File Descriptors (FDs)** — just small integers that say *where* to read from or write to.

| FD | Name | What it is |
|---|---|---|
| `0` | stdin | input — keyboard, or piped data |
| `1` | stdout | normal output — what prints to your terminal |
| `2` | stderr | error output |

When we open a file later, Linux gives us back a new number (3, 4, 5...). we use it exactly the same way.

> **Connecting back:** FDs are just integers. We pass them in registers like any other number. Nothing special about them — just a convention Linux uses.

---

## Writing to the Screen — syscall 1

`write` is syscall number 1. It needs three things:

```
write(file_descriptor, memory_address, number_of_bytes)
```

Which maps to registers like this:

| Register | What to put |
|---|---|
| `rax` | `1` — write syscall number |
| `rdi` | file descriptor (`1` = stdout, `2` = stderr) |
| `rsi` | address in memory where your data starts |
| `rdx` | how many bytes to write |

```asm
mov rax, 1          ; syscall 1 = write
mov rdi, 1          ; fd 1 = stdout
mov rsi, [rsp+16]   ; address of data to write
mov rdx, 1          ; write 1 byte
syscall
```

> **Why not just put the text in a register?** Registers only hold 8 bytes. We can't fit a real string in one register. So instead, `rsi` holds a *pointer* — an address to where the data lives in memory — and `rdx` says how many bytes to read from there. The kernel goes and fetches it.

> **Confusion clarified — rdi vs rdx:** These two look almost the same and it trips everyone up. **rdi** = first param (file descriptor). **rdx** = third param (byte count). Just remember: **i** = initial, **x** = extra. Or honestly just slow down and double check every time you write them.

---

## Writing More Than One Character

Same call, just change `rdx`:

```asm
mov rax, 1
mov rdi, 1
mov rsi, [rsp+16]
mov rdx, 13         ; write 13 bytes instead of 1
syscall
```

That's it. The kernel reads `rdx` bytes starting from `rsi` and writes them all at once.

> **Why write multiple at once?** Every syscall costs the CPU — it has to stop our program, jump into the kernel, do the work, come back. That overhead adds up. Writing 100 bytes in one call is way faster than calling write 100 times with 1 byte each.

---

## Chaining — write then exit

Goal: write something, then exit cleanly without crashing.

```asm
.intel_syntax noprefix
.global _start
_start:
    ; write
    mov rax, 1
    mov rdi, 1
    mov rsi, [rsp+16]
    mov rdx, 1
    syscall

    ; exit
    mov rax, 60
    mov rdi, 0
    syscall
```

> **Order matters.** CPU runs top to bottom. Put exit before write and the program ends before printing anything. Always think: what needs to happen first?

---

## Reading Input — syscall 0

`read` is syscall 0. Mirror of write — same structure, different direction.

```
read(file_descriptor, memory_address, max_bytes)
```

```asm
mov rax, 0          ; syscall 0 = read
mov rdi, 0          ; fd 0 = stdin
mov rsi, rsp        ; store the input here (top of stack)
mov rdx, 64         ; read up to 64 bytes
syscall
```

After this, whatever you typed is sitting in memory at `rsp`. We can then pass that same address to `write` to print it back out.

### What memory looks like after reading "HELLO"

```
rsp + 0  →  0x48  ('H')
rsp + 1  →  0x45  ('E')
rsp + 2  →  0x4c  ('L')
rsp + 3  →  0x4c  ('L')
rsp + 4  →  0x4f  ('O')
```

Characters are stored as their ASCII values — just numbers. `'H'` = `0x48`. Same byte, different ways to write it.

> **Confusion clarified:** When reading, `rdi` = `0` (stdin), not `1`. Super easy to write `mov rdi, 1` out of habit from write. I did this. Just check: reading from stdin = fd 0, writing to stdout = fd 1.

---

## for reading exactly 

To see what kinfd of byte are coming and you are working with:
Some problem i faced in this problem:
From my perspective, the problem was two-fold:

1. Linking mistake – I used ld -shared which produces a .so shared library, not an executable. The checker explicitly said it expects an ELF executable, so switched to ld -o s s.o (no -shared). That fixed the file type.

2. Syscall setup – The checker also complained that I didn’t invoke read (set rax to 0). I had used xor eax, eax to zero rax, but maybe the checker scans for the exact immediate value 0 in the instruction. When I changed it to mov eax, 0, the checker accepted it. So the insight is: sometimes the checker is literal about the instruction pattern, not just the effect.

After those two fixes, the code worked and passed the check.
code:

```
.intel_syntax noprefix
.global _start
_start:
    sub rsp, 128
    mov rsi, rsp
    xor edi, edi
    mov edx, 128
    mov eax, 0
    syscall
    mov rdx, rax
    mov edi, 1
    mov eax, 1
    syscall
    mov edi, 42
    mov eax, 60
    syscall
```
---

## Opening a File — syscall 2

stdin/stdout/stderr are always open. To read a file from disk, we have to open it first.

```
open(filename_pointer, flags)
```

| Register | Value | Meaning |
|---|---|---|
| `rax` | `2` | syscall 2 = open |
| `rdi` | address | pointer to the filename string in memory |
| `rsi` | `0` | 0 = read-only |

```asm
mov rax, 2
mov rdi, [rsp+16]   ; pointer to filename string (e.g. argv[1])
mov rsi, 0          ; read-only
syscall
; rax now = the new file descriptor (e.g. 3)
```

**The important part:** after `open` returns, `rax` holds the new fd. We have to save it *before* we put the next syscall number in `rax`, otherwise you lose it:

```asm
syscall             ; open — rax = new fd
mov rdi, rax        ; save it NOW before overwriting rax
```

Forget this and your read call uses a garbage fd and nothing works.

> **Connecting back:** The fd open gives you is just an integer — same as the 0, 1, 2 you already know. open just creates a new one dynamically and hands it back in rax.

---

## Hardcoding the Filename on the Stack

Sometimes the filename isn't passed as an argument — we need to write it into memory yourself, byte by byte. This is the "hacker" way from pwn.college, designed for exploitation not normal software dev.

```asm
mov BYTE PTR [rsp],   '/'
mov BYTE PTR [rsp+1], 'f'
mov BYTE PTR [rsp+2], 'l'
mov BYTE PTR [rsp+3], 'a'
mov BYTE PTR [rsp+4], 'g'
mov BYTE PTR [rsp+5], 0     ; null terminator
```

Three things to know:

**`BYTE PTR`** — size directive. Tells the assembler: write exactly 1 byte here. Without it the assembler doesn't know if you mean 1, 2, 4 or 8 bytes.

**Single quotes** — `'f'` is just a readable way to write the ASCII value of f, which is `0x66`. Same thing, just easier to read.

**The null byte (0)** — Linux reads a string starting at your pointer and stops when it hits a `0` byte. Without it, `open` keeps reading past "flag" into whatever else is on the stack and tries to open a file with a garbage name. Always end your string with `0`.

Then pass `rsp` to `open` as the filename pointer:

```asm
mov rdi, rsp        ; rdi = pointer to "/flag" we just wrote
mov rax, 2
mov rsi, 0
syscall
```

---

## Full Program — open, read, write, exit

This is the complete flow. Each syscall feeds into the next — the fd from open goes into read, the memory from read goes into write.

```asm
.intel_syntax noprefix
.global _start

_start:
    ; Step 1 — build "/flag\0" on the stack
    mov BYTE PTR [rsp],   '/'
    mov BYTE PTR [rsp+1], 'f'
    mov BYTE PTR [rsp+2], 'l'
    mov BYTE PTR [rsp+3], 'a'
    mov BYTE PTR [rsp+4], 'g'
    mov BYTE PTR [rsp+5], 0

    ; Step 2 — open("/flag", 0) → fd comes back in rax
    mov rdi, rsp
    mov rax, 2
    mov rsi, 0
    syscall

    ; Step 3 — save the fd, then read from the file into rsp
    mov rdi, rax        ; save fd before overwriting rax
    mov rax, 0
    mov rsi, rsp        ; store file contents here (overwrites "/flag" — fine)
    mov rdx, 64
    syscall

    ; Step 4 — write contents to stdout
    mov rax, 1
    mov rdi, 1
    mov rsi, rsp
    mov rdx, 64
    syscall

    ; Step 5 — exit
    mov rax, 60
    mov rdi, 0
    syscall
```

### What's happening to memory at rsp

```
After Step 1:   rsp → "/flag\0"
After Step 3:   rsp → file contents (overwrites the filename — that's fine, we're done with it)
Step 4 reads:   rsp → same address, now prints the file contents
```

> **Personal note:** Double check which terminal is connected to which machine when running these. I had a case where I was solving the challenge but connected to a different device — the program was running somewhere else and nothing made sense. If it's not working, check your SSH session first.

---

## Debugging

**strace first.** Run `strace ./your_binary` and look at what arguments each syscall got and what it returned. If `open` returns `-1`, our string pointer or null terminator is wrong.

```bash
strace ./p
```

**Check your string in GDB.** Step through until after we've written the filename bytes, then inspect what's actually at rsp:

```bash
gdb ./p
(gdb) starti
(gdb) stepi   # repeat until past the BYTE PTR writes
(gdb) x/s $rsp
# should show: "/flag"
```

**Common mistakes I hit:**

| Mistake | What goes wrong | Fix |
|---|---|---|
| Forgot null terminator | `open` returns -1, garbage filename | `mov BYTE PTR [rsp+5], 0` |
| `rdi=1` for read | reading from stdout instead of stdin | read uses `rdi=0` |
| Didn't save fd before next `mov rax` | read uses garbage fd | `mov rdi, rax` right after open syscall |
| Connected to wrong terminal | nothing works, no idea why | check your SSH session |

---

## 🔐 Security Context

**This is literally shellcode.** The open→read→write pattern is exactly what shellcode does to read `/etc/shadow`, `/flag`, or any sensitive file. You just built it from scratch in assembly. That's not an exercise — that's the real thing.

**Null termination bugs are a vulnerability class.** Programs that build strings in memory without the null byte can leak data past the string boundary. You felt firsthand why the null byte matters — now you understand the bug when you see it in a CTF.

**Every syscall argument is inspectable.** `strace` shows you what any binary passes to any syscall — fd numbers, memory addresses, byte counts. Malware has to make syscalls to do anything real. `strace` catches all of it.

> **Connecting the dots:** All four previous files come together here for the first time — registers and syscalls (`Assembly_Language_Intro.md`) + memory addressing for pointers (`Memory.md`) + stack as scratch space (`The_Stack.md`) + strace/GDB to debug it (`Software_Introspection.md`). This is the first program that actually *does something*.

---

## Practice Checklist

- [ ] Write 1 byte to stdout
- [ ] Write multiple bytes (change rdx)
- [ ] Chain write + exit cleanly
- [ ] Read from stdin into stack, then echo it back with write
- [ ] Open a file from argv[1], read contents, print them
- [ ] Hardcode `/flag` byte by byte onto the stack
- [ ] Verify your string with `x/s $rsp` in GDB
- [ ] Use strace to confirm each syscall got the right arguments
