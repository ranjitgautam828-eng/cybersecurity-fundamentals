# Register Challenges

> **Where this fits:** This is practical application — moving from theory into actual challenges. We already know the instructions. This section is about applying them correctly and the gotchas we hit along the way.

---

## Table of Contents

- [1. Set a Single Register](#1-set-a-single-register)
- [2. Set Multiple Registers](#2-set-multiple-registers)
- [3. Add to a Register](#3-add-to-a-register)
- [4. Linear Equation](#4-linear-equation)
- [5. Integer Division](#5-integer-division)
- [Security Context](#-security-context)
- [Practice Checklist](#practice-checklist)

---

## 1. Set a Single Register

**Goal:** set `rdi = 0x1337`

```asm
.intel_syntax noprefix
.global _start
_start:
    mov rdi, 0x1337
    mov rax, 60
    syscall
```

> **Gotcha I hit:** `0x1337` is hex. Writing `1337` without `0x` is decimal — a completely different value and the challenge fails. Always check what format the challenge is asking for.

---

## 2. Set Multiple Registers

**Goal:** set `rax = 0x1337`, `r12 = 0xCAFED00D1337BEEF`, `rsp = 0x31337`

The problem here: `rax` is also used for the syscall number (`60` for exit). If we set `rax = 0x1337` and then do the exit syscall, we'd overwrite it with `60` — wrong value for the challenge.

I went too far ahead thinking about the syscall. The fix is simple: just don't use a syscall at all. The checker reads register state directly — it doesn't need a clean exit.

```asm
.intel_syntax noprefix
.global _start
_start:
    mov rax, 0x1337
    mov r12, 0xCAFED00D1337BEEF
    mov rsp, 0x31337
```

> No `syscall` here — checker inspects the registers directly.

---

## 3. Add to a Register

**Goal:** add `0x1337` to whatever value `rax` already holds

Same syntax as `mov`, just a different instruction:

```asm
.intel_syntax noprefix
.global _start
_start:
    add rax, 0x1337
```

Result stays in `rax`.

---

## 4. Linear Equation

**Goal:** compute `f(x) = m*x + b` where `m = rdi`, `x = rsi`, `b = rdx`, put result in `rax`

```asm
.intel_syntax noprefix
.global _start
_start:
    imul rdi, rsi       ; rdi = rdi * rsi  (m * x)
    add rdi, rdx        ; rdi = rdi + rdx  (+ b)
    mov rax, rdi        ; rax = result
```

`imul` = integer multiply. Multiplies `rdi` by `rsi`, result goes back into `rdi`. Then we add `rdx` and move the final value to `rax`.

> Multiply first, then add — same order as normal math.

---

## 5. Integer Division

**Goal:** divide using `div`

`div` uses implicit registers — we don't choose them:

- Dividend goes in `rax`
- Divisor passed as the operand
- After `div`: quotient → `rax`, remainder → `rdx`

We need to clear `rdx` before calling `div` — otherwise it's treated as the high 64 bits of the dividend and we get a wrong result or a crash:

```asm
.intel_syntax noprefix
.global _start
_start:
    xor rdx, rdx        ; clear rdx before div
    div rdi             ; rax = rax / rdi, rdx = rax % rdi
```

> `xor rdx, rdx` = fastest way to zero a register. Same as `mov rdx, 0` but shorter. We'll see this pattern a lot.

---

## 🔐 Security Context

**This is exactly what ROP gadgets do.** Every ROP gadget is just moving specific values into specific registers before a syscall or function call — the same precision we practiced here. Getting hex vs decimal wrong in a real exploit payload means a crash or wrong address.

**`imul` overflow is a real bug class.** When multiplication produces a value too large for the register, it wraps around silently. In C programs this can cause buffer size miscalculations that lead to heap overflows. Understanding `imul` at the assembly level is the foundation for spotting integer overflow bugs.

**`div` by zero crashes the process.** Forgetting to clear `rdx` before `div` produces unexpected behavior that can look like a different kind of crash — knowing the mechanics helps us debug fast.

> **Connecting the dots:** Register knowledge from `Assembly_Language_Intro.md` + number formats from `Nibbling_on_Numbers.md` → applied here as real challenges. This is where theory becomes muscle memory.

---

## Practice Checklist

- [ ] Set a single register to a hex value
- [ ] Set multiple registers without using syscall exit
- [ ] Add an immediate to a register with `add`
- [ ] Implement `f(x) = m*x + b` using `imul` and `add`
- [ ] Use `div` correctly: clear `rdx` first, read quotient from `rax` and remainder from `rdx`
- [ ] Deliberately write decimal where hex is required — observe the wrong result
