# Control Flow & Functions

> **Where this fits:** So far every program ran straight through. Control flow adds decisions and loops — like if/else and for loops but in assembly. Functions adds the ability to write code that gets *called by* other programs. Both topics use the same registers and stack you already know.

---

## Table of Contents

- [Comparing Values](#comparing-values)
- [The Zero Flag](#the-zero-flag)
- [setz — capture the result](#setz--capture-the-result)
- [Register Sizes — rdi vs dil](#register-sizes--rdi-vs-dil)
- [Comparing Characters](#comparing-characters)
- [Conditional Jumps — jne](#conditional-jumps--jne)
- [Labels](#labels)
- [Two-path program](#two-path-program)
- [Comparing Strings](#comparing-strings)
- [Reverse Engineering a Password](#reverse-engineering-a-password)
- [Jump Tables](#jump-tables)
- [Loops](#loops)
- [Shared Libraries](#shared-libraries)
- [call and ret](#call-and-ret)
- [Building a Shared Library](#building-a-shared-library)
- [Calling Convention](#calling-convention)
- [Calling a Function Pointer](#calling-a-function-pointer)
- [Calling a Function Pointer with Arguments](#calling-a-function-pointer-with-arguments)
- [Caller-Saved vs Callee-Saved](#caller-saved-vs-callee-saved)
- [Saving Caller-Saved Registers](#saving-caller-saved-registers)
- [Saving Callee-Saved Registers](#saving-callee-saved-registers)
- [Security Context](#-security-context)
- [Practice Checklist](#practice-checklist)

---

## Comparing Values

Programs need to make decisions — if this, do that. It starts with `cmp`.

`cmp` subtracts the second value from the first. It does **not** store the result — it just updates CPU flags based on what happened.

```asm
cmp rdi, 42     ; internally: rdi - 42, result thrown away, flags updated
```

> **rdi is not changed.** Only the flags change.

---

## The Zero Flag

The main flag: **Zero Flag (ZF)**

- Result was `0` (values equal) → ZF = 1
- Result was anything else → ZF = 0

```
cmp rdi, 42

rdi = 42  →  42 - 42 = 0  →  ZF = 1
rdi = 99  →  99 - 42 ≠ 0  →  ZF = 0
```

---

## setz — capture the result

`setz` reads the Zero Flag and writes `1` or `0` into a register:

```asm
cmp rdi, 42
setz dil        ; dil = 1 if equal, 0 if not
```

`setnz` does the opposite (1 if not equal). Less common.

### Challenge example — exit with 1 if argc equals 42

```asm
.intel_syntax noprefix
.global _start
_start:
    mov rdi, [rsp]
    cmp rdi, 42
    setz dil
    mov rax, 60
    syscall
```

---

## Register Sizes — rdi vs dil

`setz` writes only 1 byte. That's why we use `dil`, not `rdi`.

Every 64-bit register has a smaller 8-bit (1 byte) part you can access:

| 64-bit | Low 8 bits |
|---|---|
| `rdi` | `dil` |
| `rax` | `al` |
| `rsi` | `sil` |
| `rdx` | `dl` |

Write to `dil` and only that bottom byte of `rdi` changes.

---

## Comparing Characters

`argv[1]` is at `[rsp+16]` — a pointer to a string. To check the first character:

```asm
mov rax, [rsp+16]       ; rax = pointer to argv[1]
cmp BYTE PTR [rax], 'p' ; compare first character with 'p'
```

`BYTE PTR` = read exactly 1 byte. `'p'` = ASCII value `0x70`.

> **cmp rule:** can't compare two memory locations directly. At least one side must be a register. Load into a register first if needed.

### Challenge example

```asm
.intel_syntax noprefix
.global _start
_start:
    mov rdi, [rsp+16]
    cmp BYTE PTR [rdi], 'p'
    setz dil
    mov rax, 60
    syscall
```

---

## Conditional Jumps — jne

`setz` gives you a 0 or 1. But what if you want to actually *do something different* based on the result? That's what conditional jumps are for.

**`jne`** = Jump if Not Equal

```asm
cmp BYTE PTR [rax], 'p'
jne fail        ; if NOT equal → jump to fail
                ; if equal → keep going (fall through)
```

Other useful jumps:

| Instruction | Meaning |
|---|---|
| `jne` | jump if not equal (ZF = 0) |
| `je` | jump if equal (ZF = 1) |
| `jmp` | always jump (unconditional) |

---

## Labels

`fail` is a **label** — a name for a location in your code. The assembler converts it to an address. Labels don't generate machine code, they just mark a spot.

```asm
fail:
    mov rdi, 1
    mov rax, 60
    syscall
```

Name them whatever makes sense: `fail`, `success`, `done`, `loop`.

---

## Two-path program

```asm
.intel_syntax noprefix
.global _start
_start:
    mov rdi, [rsp+16]
    cmp BYTE PTR [rdi], 'p'
    jne fail

success:
    mov rdi, 0          ; exit code 0 = success
    mov rax, 60
    syscall

fail:
    mov rdi, 1          ; exit code 1 = failure
    mov rax, 60
    syscall
```

- Equal → falls through to success → exits `0`
- Not equal → jumps to fail → exits `1`

> Linux convention: `0` = success, non-zero = failure. Same idea as how you've been setting exit codes all along.

---

## Comparing Strings

Chain `cmp`/`jne` pairs — one per character, all jumping to the same `fail`:

```asm
mov rax, [rsp+16]

cmp BYTE PTR [rax],   'Y'
jne fail

cmp BYTE PTR [rax+1], 'E'
jne fail

cmp BYTE PTR [rax+2], 'S'
jne fail

; all matched — fall through
mov rdi, 0
mov rax, 60
syscall

fail:
    mov rdi, 1
    mov rax, 60
    syscall
```

First wrong character → `jne` bails immediately. All pass → falls through to success.

---

## Reverse Engineering a Password

In pwn.college some challenges have a hidden password. You find it by reading the binary:

```bash
objdump -d -M intel /challenge/reverse-me
```

Look for `cmp` instructions — they show what each character is compared against (in hex). Convert to ASCII:

```bash
printf "\x6f\x62\x6f\x6a"
```

That prints the actual password string. You just reverse-engineered a program with no source code.

> Use GDB to step through it live if `objdump` alone isn't enough.

---

## Jump Tables

Chaining `cmp/jne` for many cases gets slow. A **jump table** is faster — use the input as an index into an array of addresses, then jump directly.

In disassembly it looks like:

```asm
xor    eax, eax                 ; zero rax
mov    al, BYTE PTR [rcx]       ; load character into al (low byte of rax)
mov    rax, [rax*8+0x1234000]   ; look up address in jump table
jmp    rax                      ; jump there
```

- `al` = low byte of `rax` (same idea as `dil` for `rdi`)
- `rax*8` = each address is 8 bytes, so multiply index by 8
- The table at `0x1234000` is just an array of addresses in memory

> This is what a C `switch` statement compiles to. Brain-frying the first time — do the pwn.college challenge and step through it in GDB to watch it happen.

---

## Loops

A loop jumps *backward* to repeat. `jmp` is unconditional — it always jumps.

```asm
loop:
    cmp BYTE PTR [rax], 0   ; null terminator = end of string
    je success

    cmp BYTE PTR [rax], 'x'
    jne fail

    add rax, 1              ; advance to next character
    jmp loop                ; go back to top

success:
    mov rdi, 0
    mov rax, 60
    syscall

fail:
    mov rdi, 1
    mov rax, 60
    syscall
```

- `jmp loop` → unconditional, always loops back
- `je success` → exits loop when null terminator found
- `add rax, 1` → moves pointer to next byte

> This is the pattern behind every `for` and `while` loop in compiled code. Do the pwn.college loop challenge — read the disassembly, figure out the password, verify in GDB.

---

## Shared Libraries

Everything so far has been a standalone executable starting at `_start`. A **shared library** (`.so` file) is different — it's code that another program loads and calls into. You write a named function, not `_start`.

The grader in pwn.college calls your `solve` function. You don't start the program — you just receive the call and do your work.

Examples in the real world: `libpng` parses PNGs, `libc` handles memory and files. Same syscalls underneath, just wrapped.

---

## call and ret

**`call <target>`** — how a caller invokes a function:
1. Pushes return address (next instruction after `call`) onto the stack
2. Jumps to `<target>`

**`ret`** — how a function returns:
1. Pops the return address from the stack
2. Jumps to it

```asm
; grader does this:
call solve          ; pushes return address, jumps into your code

; you write this:
solve:
    ; do stuff
    ret             ; pop return address, jump back to grader
```

> **Connecting back:** `call` uses the stack from `The_Stack.md`. The return address is just another value on the stack — which is exactly why stack overflows can hijack execution.

---

## Building a Shared Library

```bash
as -o solve.o solve.s
ld -shared -o solve.so solve.o
/challenge/check solve.so
```

Your function must be global:

```asm
.intel_syntax noprefix
.global solve
solve:
    ; your code
    ret
```

> **My mistakes:**
> - Wrong filename (`your-solve.s` instead of `solve.s`)
> - Forgot `.intel_syntax noprefix` — assembler rejected the code
> - Tried `sudo` — disabled in the environment
>
> Fix: correct filename, add the directive, no sudo, link with `-shared`.

---

## Calling Convention

When the grader calls your function, arguments arrive in registers. This is the **Linux x86-64 calling convention**:

| Register | Role |
|---|---|
| `rdi` | 1st argument |
| `rsi` | 2nd argument |
| `rdx` | 3rd argument |
| `rax` | **return value** |

Same registers as syscalls — same order. Functions and syscalls follow the same convention.

### Return the first argument

```asm
.intel_syntax noprefix
.global solve
solve:
    mov rax, rdi    ; return value = first argument
    ret
```

### Write a buffer to stdout (pointer in rdi, length in rsi)

```asm
.intel_syntax noprefix
.global solve
solve:
    mov rdx, rsi    ; length → rdx (3rd syscall arg)
    mov rsi, rdi    ; address → rsi (2nd syscall arg)
    mov rdi, 1      ; stdout
    mov rax, 1      ; write syscall
    syscall

    mov rdi, 0
    mov rax, 60
    syscall
```

---

## Calling a Function Pointer

Sometimes an argument is itself a function pointer. Use `call rdi` to call whatever address is in `rdi`:

```asm
.intel_syntax noprefix
.global solve
solve:
    call rdi
    ret
```

---

## Calling a Function Pointer with Arguments

Problem: the pointer is in `rdi` but `rdi` is also the first argument slot. You can't put both there.

Fix: move the pointer to another register first, then set `rdi` freely:

```asm
.intel_syntax noprefix
.global solve
solve:
    mov rax, rdi    ; save pointer to rax
    mov rdi, 1337   ; set first argument
    call rax        ; call the function
    ret
```

> Registers are shared by everything. When two things need the same register, shuffle one out first.

---

## Caller-Saved vs Callee-Saved

Only 16 registers, shared by every function. The calling convention defines who's responsible for saving them:

**Caller-saved** — the function you call can overwrite these freely. You save them before calling, restore after.
```
rax, rcx, rdx, rsi, rdi, r8, r9, r10, r11
```

**Callee-saved** — your function must leave these unchanged. If you use them, push on entry, pop before `ret`.
```
rbx, rbp, r12, r13, r14, r15
```

Think of callee-saved as borrowed — return them exactly as found.

---

## Saving Caller-Saved Registers

Push before the call, pop after in reverse order:

```asm
.intel_syntax noprefix
.global solve
solve:
    push rax
    push rcx
    push rdx
    push rsi
    push rdi
    push r8
    push r9
    push r10
    push r11

    call rdi            ; call clobber_function

    pop r11
    pop r10
    pop r9
    pop r8
    pop rdi
    pop rsi
    pop rdx
    pop rcx
    pop rax

    call rsi            ; call flag_function (pointer restored)
    ret
```

> Always pop in reverse order — stack is LIFO.

---

## Saving Callee-Saved Registers

Push on entry, pop before `ret`:

```asm
.intel_syntax noprefix
.global solve
solve:
    push rbx
    push r12
    push r13
    push r14
    push r15

    mov rbx, 0x1337
    mov r12, 0x1337
    mov r13, 0x1337
    mov r14, 0x1337
    mov r15, 0x1337

    call rdi

    pop r15
    pop r14
    pop r13
    pop r12
    pop rbx

    ret
```

---

## 🔐 Security Context

**Return address overwrite.** `call` pushes the return address onto the stack. Overflow a buffer inside that function → overwrite the return address → `ret` jumps wherever you want. That's the classic stack exploit.

**ROP (Return-Oriented Programming).** Instead of injecting shellcode, chain together existing code snippets that end in `ret`. Each `ret` pops the next address from a stack you control. `call`/`ret` mechanics are the foundation.

**Caller/callee-saved matters for exploits.** Knowing which registers survive across a `call` tells you what values stay set when building an exploit payload or ROP chain.

**Timing attacks from `jne fail`.** When you bail on the first wrong character, the program exits faster for early mismatches. An attacker can measure that — guess the password one character at a time. Real attack, comes directly from how `jne` works.

> **Connecting the dots:** Stack layout → `call` pushes return address → `ret` pops and jumps → overflow buffer → overwrite return address → control execution. All your previous notes feed directly into this.

---

## Practice Checklist

**Control Flow**
- [ ] Use `cmp` + `setz` to exit with 1 if argc equals a value
- [ ] Use `cmp` + `jne` to check first character of argv[1]
- [ ] Write a two-path program: success exits 0, fail exits 1
- [ ] Chain `cmp`/`jne` to check a multi-character string
- [ ] Use `objdump` to reverse engineer a password, convert hex with `printf "\xNN"`
- [ ] Step through a jump table in GDB
- [ ] Write a loop that checks every character until null terminator

**Functions**
- [ ] Write a `.so` with a `solve` function that returns a value in `rax`
- [ ] Write `solve` that calls `write` syscall using arguments it receives
- [ ] Call a function pointer in `rdi`
- [ ] Call a function pointer with a separate argument (save pointer to `rax` first)
- [ ] Save and restore all caller-saved registers around a `call`
- [ ] Save and restore callee-saved registers inside your function
- [ ] Build with `ld -shared`, verify with `/challenge/check`
