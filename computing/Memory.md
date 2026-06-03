# Memory Addressing in Assembly

> **Where this fits:** The CPU can only work on data that's in registers — but registers are tiny (16 slots). Everything else lives in memory (RAM). This file is about how we reach into memory to read and write values, and how addresses, pointers, and pointer chains work.

---

## Table of Contents

- [The Memory Hierarchy](#the-memory-hierarchy)
- [Reading and Writing Memory](#reading-and-writing-memory)
- [Address Calculation Formula](#address-calculation-formula)
- [LEA — Load Effective Address](#lea--load-effective-address)
- [The Stack Pointer (rsp)](#the-stack-pointer-rsp)
- [RIP-Relative Addressing](#rip-relative-addressing)
- [Endianness](#endianness-x86x64--little-endian)
- [Dereferencing — Step by Step](#dereferencing--step-by-step)
- [Quick Syntax Reference](#quick-syntax-reference)
- [Security Context](#-security-context--why-this-matters-for-hacking)
- [Practice Checklist](#practice-checklist)

---

## The Memory Hierarchy

```
┌─────────────────────────────────────────────────┐
│  Registers   │  ~16 slots │ fastest │ inside CPU │
├─────────────────────────────────────────────────┤
│  RAM         │  GBs       │ slower  │ main memory│
├─────────────────────────────────────────────────┤
│  Storage     │  TBs       │ slowest │ disk/SSD   │
└─────────────────────────────────────────────────┘
```

**Analogy:** Registers = what's in our hand right now. RAM = our desk. Storage = a filing cabinet across the room.

The CPU can only do math on register values. If the data we need is in RAM, we have to **load** it into a register first. If we've computed something and need to **save** it, you write it back to RAM.

---

## Reading and Writing Memory

Memory is one giant array of bytes. Every byte has an **address** — a number that tells us where it is.

### Reading from memory (load)

```asm
mov rax, 0x12345      ; rax = the address 0x12345 (just a number)
mov rbx, [rax]        ; rbx = the VALUE stored at that address
```

The `[ ]` brackets mean **"go to this address and grab what's there"** — like following a link or opening a box.

### Writing to memory (store)

```asm
mov rax, 0x133337     ; rax = target address
mov [rax], rbx        ; write rbx's value INTO that address
```

### The simple rule

| Syntax | Meaning |
|---|---|
| `mov rax, rbx` | rax = the value of rbx |
| `mov rax, [rbx]` | rax = the value **at the address** rbx holds |
| `mov [rax], rbx` | write rbx's value **to the address** rax holds |

> **`[ ]` = dereference.** Follow the address, don't just use the number.

---

## Address Calculation Formula

We don't have to use a plain register as an address. we can compute addresses inline:

```
[base + index * scale + offset]
```

| Part | Example | Meaning |
|---|---|---|
| `base` | `rsp` | starting address (usually a register) |
| `index` | `rax` | which element you want |
| `scale` | `8` | size of each element — must be 1, 2, 4, or 8 |
| `offset` | `+5` | fixed extra shift |

**Example — reading element `rax` from an array at `rsp`:**

```asm
mov rbx, [rsp + rax*8]    ; go to rsp, jump (rax × 8) bytes, read value
```

Why `* 8`? Because each element is 8 bytes (64-bit values). If we want element 0, jump 0. Element 1, jump 8. Element 2, jump 16. Etc.

---

## LEA — Load Effective Address

`lea` **calculates** an address without actually reading from memory.

```asm
lea rbx, [rsp + rax*8 + 5]   ; rbx = the address itself (rsp + rax*8 + 5)
mov rbx, [rsp + rax*8 + 5]   ; rbx = the VALUE stored at that address
```

**Analogy:**
- `lea` gives us the house number written on paper
- `mov [...]` actually opens the door and walks inside

**Why use `lea`?** Sometimes we need the address itself (to pass as a pointer, or to do math on it), not the value at that address.

---

## The Stack Pointer (rsp)

`rsp` is a register that always points to the **top of the stack** — a pre-allocated region of memory our program gets for free.

```asm
push rcx

; push does this manually:
sub rsp, 8        ; grow the stack downward (stack grows down in x86-64)
mov [rsp], rcx    ; store rcx at the new top
```

See `The_Stack.md` for a full breakdown of how the stack works and how program arguments are laid out on it.

---

## RIP-Relative Addressing

`rip` is the **Instruction Pointer** — it always holds the address of the *next* instruction to be executed.

```asm
lea rax, [rip]      ; rax = address of the next instruction
mov rax, [rip]      ; rax = the bytes AT the next instruction's address
```

**Why does this matter?**
- Used in **position-independent code (PIC)** — code that works regardless of where it's loaded in memory
- Required for shared libraries (`.so` files)
- Relevant to **ASLR** (Address Space Layout Randomization) — a security feature that randomizes where code is loaded. RIP-relative addressing is what makes code survive ASLR.

---

## Endianness (x86/x64 = Little Endian)

When we store a multi-byte value like `0x12345678` in memory, the bytes go in **lowest byte first**:

```
Address:  [0]   [1]   [2]   [3]
Stored:   0x78  0x56  0x34  0x12
          ↑ least significant byte first
```

This is **little-endian** — x86 and x64 always work this way.

**When does this matter?**
- Reading raw memory dumps (in GDB, Wireshark, hex editors)
- Writing shellcode that embeds addresses as bytes
- Network protocols (most use big-endian, so you have to flip)

> It doesn't affect normal assembly (`mov rax, 0x12345678` just works). We only notice endianness when we inspect raw bytes.

---

## Dereferencing — Step by Step

These six examples build on each other. Each one is a real pattern from pwn.college's Computer Memory module.

---

### 1. Load from a hardcoded address

```bash
echo -e ".intel_syntax noprefix\n.global _start\n_start:\n mov rdi, qword ptr [133700]\n mov rax, 60\n syscall" > p.s
as -o p.o p.s
ld -o p p.o
```

What happens: loads the value stored at address `133700` into `rdi`, then exits with it.

> **`qword ptr`** — tells GAS (GNU Assembler) this is a 64-bit (8-byte) read. NASM figures this out automatically; GAS needs the hint when using a raw address.

---

### 2. Use a register as the address

```asm
mov rdi, qword ptr [rax]   ; follow the address stored in rax
```

pwn.college already put the right address in `rax` — you just follow it.

Same as example 1, except the address is in a register instead of hardcoded.

---

### 3. Register pointing to itself

```asm
mov rdi, qword ptr [rdi]   ; rdi holds an address → follow it → load new value back into rdi
```

`rdi` contains an address. You dereference it and put the result back into `rdi`. Register pointing to itself — totally valid.

---

### 4. Dereference with offset

If one address holds multiple values in a row:

```
address 133700 → value: 50
address 133708 → value: 42   (8 bytes later)
address 133716 → value: 99   (16 bytes later)
```

```asm
mov rax, [rdi]        ; gets 50  (at rdi + 0)
mov rax, [rdi + 8]    ; gets 42  (at rdi + 8)
mov rax, [rdi + 16]   ; gets 99  (at rdi + 16)
```

> Offsets are multiples of 8 for 64-bit values because each one takes 8 bytes of space.

---

### 5. Follow a pointer chain (2 hops)

```bash
echo -e ".intel_syntax noprefix\n.global _start\n_start:\n mov rdi, [567800]\n mov rdi, [rdi]\n mov rax, 60\n syscall" > p.s
```

What happens:
- Load the value at `567800` into `rdi` — that value is itself another address
- Follow that second address to get the actual secret value

**Analogy:** You open a bag and find a key. The key opens the real locker. Two steps to get what you want.

```
567800 → [address X]
address X → [secret value]
```

---

### 6. Double dereference — pointer to a pointer

```bash
echo -e ".intel_syntax noprefix\n.global _start\n_start:\n mov rdi, qword ptr [rax]\n mov rdi, qword ptr [rdi]\n mov rax, 60\n syscall" > p.s
```

What happens:
1. `rax` contains an address — follow it → get `address2`
2. Follow `address2` → get the actual secret value
3. Exit with that value in `rdi`

```
rax → [address2]
address2 → [secret]
```

This is a **pointer-to-a-pointer** — one of the most common patterns in C and in memory exploitation.

---

## Quick Syntax Reference

| Syntax | Meaning |
|---|---|
| `mov rax, 5` | rax = 5 |
| `mov rax, [5]` | rax = value stored at address 5 |
| `mov [rax], rbx` | store rbx's value at the address rax holds |
| `lea rax, [rsp+8]` | rax = rsp+8 (the address, not the value there) |
| `[rsp + rax*8]` | address = rsp + (rax × 8) |
| `qword ptr [addr]` | explicit 64-bit read (needed in GAS with raw addresses) |

---

## 🔐 Security Context — Why This Matters for Hacking

**Buffer overflows** happen when a program writes past the end of a buffer into adjacent memory. To exploit one, you need to know exactly what's at each memory address, how offsets work, and how to overwrite specific values — exactly what this file covers.

**Pointer chains are everywhere in exploitation.** Real heap exploits (use-after-free, double-free) involve manipulating linked lists of pointers. When we free a chunk of memory and reallocate it, you're redirecting a pointer chain. The double-dereference pattern (example 6) is not a toy — it's literally how heap metadata works.

**Understanding `[rsp + offset]` is how you read/overwrite return addresses.** The return address (where execution jumps after a function finishes) lives on the stack at a predictable offset from `rsp`. Knowing how to address memory with offsets is the mechanical skill behind stack smashing.

**ASLR and PIE** (Position Independent Executables) randomize where code and data are loaded. Attackers bypass ASLR by leaking an address at runtime and computing offsets from it — which is exactly what `[rip + offset]` and pointer chain walking are for.

> **Connecting the dots:** Registers → Memory addressing (this file) → Stack layout (`The_Stack.md`) → Reading binaries (`Software_Introspection.md`) → Buffer overflows and heap exploitation.

---

## Practice Checklist

- [ ] Load from a hardcoded address
- [ ] Load using a register as the address (`[rax]`)
- [ ] Load where register points to itself (`[rdi]` → back into `rdi`)
- [ ] Load with byte offset (`[rdi + 8]`, `[rdi + 16]`)
- [ ] Follow a 2-hop pointer chain
- [ ] Double dereference (`rax` → address → address → secret)
- [ ] Use `lea` to get an address without dereferencing
