# The Stack — Part 2

> **Where this fits:** This comes after you've finished sections 1–7. You already know rsp, argc, argv, pop, and push. Part 2 goes deeper — how the stack looks across function calls, how to read data that belongs to other functions, and how environment variables live on the stack and affect memory addresses.

---

## Table of Contents

- [Stack Layout Across Function Calls](#stack-layout-across-function-calls)
- [Reaching into the Caller's Frame](#reaching-into-the-callers-frame)
- [Environment Variables on the Stack](#environment-variables-on-the-stack)
- [Stack Alignment via Environment](#stack-alignment-via-environment)
- [Aligning the Stack through GDB](#aligning-the-stack-through-gdb)
- [Security Context](#-security-context)
- [Practice Checklist](#practice-checklist)

---

## Stack Layout Across Function Calls

You know the stack grows downward. Here's what it looks like when one function calls another:

```
High addresses (older data)
┌──────────────────────────┐
│  caller's local variables │  ← caller's frame
│  caller's saved registers │
│  return address           │  ← pushed by call
├──────────────────────────┤  ← rsp when solve() starts
│  solve's frame            │  ← grows down as you push
└──────────────────────────┘
Low addresses
```

When your `solve` starts:
- `[rsp]` = return address (8 bytes, pushed by `call`)
- `[rsp + 8]` and above = the caller's data, still sitting there

The caller's frame doesn't disappear. It's still in memory at higher addresses — you can read it.

---

## Reaching into the Caller's Frame

**The key insight:** there is no protection stopping your code from reading anywhere on the stack. Local variables from the caller, return addresses, anything — all just memory at predictable offsets from `rsp`.

In the challenge, the caller stores the flag in its own local variables before calling `solve`. From `solve`'s view, it's at `[rsp + 0x40]`.

```
[rsp]        = return address
[rsp + 0x08] = start of caller's frame
...
[rsp + 0x40] = where caller stashed the flag
```

### Solution

```asm
.intel_syntax noprefix
.global solve
solve:
    mov rsi, rsp        ; start at rsp
    add rsi, 0x40       ; advance to where the flag is
    mov rax, 1          ; write syscall
    mov rdi, 1          ; stdout
    mov rdx, 0x40       ; 128 bytes
    syscall
    ret
```

> `add rsi, 0x40` gives you the **address** of the flag — which is what `write` needs as a pointer. Don't dereference it.

### Build and run

```bash
as -o solve.o solve.s
ld -shared -o solve.so solve.o
/challenge/check solve.so
```

**My mistakes:**
- Passed `solve.s` to checker — that's source text, not a compiled library
- Passed `solve` with no extension — file not found
- Fix: always build first, pass the `.so`

---

## Environment Variables on the Stack

At program start, the kernel puts three things on the stack:

```
[rsp]      = argc
[rsp+8]    = argv[0] pointer
[rsp+16]   = argv[1] pointer (if exists)
[rsp+24]   = envp[0] pointer  ← environment variables start here
```

`envp[0]` points to the first environment variable string — in the challenge this is `"FLAG=pwn.college{...}"`.

### Solution — print envp[0]

```asm
.intel_syntax noprefix
.global _start
_start:
    mov rsi, [rsp+24]   ; envp[0] → pointer to FLAG string
    mov rax, 1          ; sys_write
    mov rdi, 1          ; stdout
    mov rdx, 100        ; bytes to print
    syscall

    mov rax, 60         ; sys_exit
    xor rdi, rdi
    syscall
```

### Mistakes I hit on this one

| Mistake | What happened | Lesson |
|---|---|---|
| Used `eax`, `edi`, `edx` (32-bit) | Checker complained | Always use `rax`, `rdi`, `rsi`, `rdx` for syscalls |
| Used `ld -shared` instead of `ld -o` | Got `.so` when checker wanted ELF binary | `.so` ≠ executable — know which one checker wants |
| Didn't rebuild clean | Checker tested old binary | Always `rm solve solve.o` before rebuilding |
| Wrong instruction count | Checker failed | In pwn.college, instruction count is a real constraint |
| Tried `/challenge/run` | Didn't exist | Always `ls /challenge` first |

**Clean rebuild habit:**

```bash
rm -f solve solve.o
as -o solve.o solve.s
ld -o solve solve.o
```

---

## Stack Alignment via Environment

### The idea

Linux builds the entire stack at launch — env strings, argv strings, pointer tables, argc — all in one contiguous region. More environment bytes → everything shifts up.

**Changing `FOO=xxxxx` literally moves `argv[0]`'s address.**

### The challenge

Make `argv[0]` land at a specific target address the program tells you.

### Workflow

```bash
# 1. run normally, see current vs target address
/challenge/program

# 2. compute shift needed
shift = target - current

# 3. add padding via environment variable
env -i FOO=$(python3 -c 'print("x"*N)') /challenge/program

# 4. adjust N until addresses match
```

> `env -i` starts with a clean environment — no system variables shifting things unpredictably.

> Only the **length** of the variable matters, not the name or value. You're just adding bytes.

**Pitfall:** stack layout changes slightly between runs — always recompute from fresh output, not old addresses.

---

## Aligning the Stack through GDB

GDB adds its own environment variables, so addresses inside GDB differ from a normal shell run. The challenge is to make `argv[0]` land at the same address in both.

### Workflow

```bash
# Step 1: get target address from GDB
gdb /challenge/program
(gdb) run
# note the argv[0] address

# Step 2: get normal execution address
env -i VAR=value /challenge/program
# program shows current address and difference from target

# Step 3: tune the padding
env -i VAR=$(python3 -c 'print("x"*N)') /challenge/program
# adjust N until difference = 0
```

### Mental model

```
Stack alignment problem = string length tuning problem

More chars in VAR  →  stack shifts  →  argv[0] moves
Fewer chars        →  stack shifts the other way
```

> **Pitfall:** don't change the variable name — that shifts things differently. Only change the value length.

---

## 🔐 Security Context

**Reading the caller's frame is what code execution gives you.** When an attacker gets code execution inside a process, the whole stack is readable — passwords, keys, tokens stored as local variables in other functions are all accessible with simple `[rsp + offset]` reads.

**Stack canaries exist because of this.** A canary is a random value placed between local variables and the return address. Before `ret`, the program checks it's unchanged. It doesn't stop reading — only detects overwrites.

**Environment variable padding is a real technique.** In older exploits (before ASLR), attackers used environment variable size to position shellcode at predictable addresses. Same mechanic you just learned.

**GDB vs real execution address differences matter in CTFs.** When an exploit works in GDB but fails outside, the stack shifted. Fix: account for the environment difference — exactly what this section teaches.

> **Connecting the dots:** Stack basics (`The_Stack.md`) + `call`/`ret` mechanics (`Control_Flow.md`) → caller frame reading + stack alignment. The stack goes from a data structure to an attack surface.

---

## Quick Reference

| Concept | Detail |
|---|---|
| Caller's frame | Above `rsp` — higher addresses than your function |
| `[rsp]` at `solve` entry | Return address (8 bytes) |
| `[rsp + 0x40]` | Caller's local variable — readable directly |
| `envp[0]` on stack | `[rsp+24]` at program entry |
| Stack shift control | Change env variable length → all addresses shift |
| `env -i` | Clean environment — removes unpredictable shifts |

---

## Practice Checklist

- [ ] Read flag from `[rsp + 0x40]` in the caller's frame challenge
- [ ] Print `envp[0]` using `[rsp+24]`
- [ ] Use clean rebuild: `rm` → `as` → `ld`
- [ ] Use `env -i` to control the stack environment
- [ ] Align `argv[0]` to a target address by tuning env variable length
- [ ] Explain why GDB and shell execution have different stack addresses
- [ ] Draw the full stack: env strings → argv strings → envp[] → argv[] → argc
