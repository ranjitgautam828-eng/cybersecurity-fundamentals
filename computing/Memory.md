# Memory & Memory Hierarchy

## Human Memory Analogy

* **Short-Term Memory (STM)**

  * Used for current thinking and tasks.
  * Holds about **5–9 items**.
* **Long-Term Memory (LTM)**

  * Stores knowledge permanently.
  * Examples: books, journals, Wikipedia.

### Memory Flow

```text
Long-Term Memory
      ↓
Short-Term Memory (work on Assembly Language Intro)
      ↓
Store results back to Long-Term Memory
```

---

# Computer Memory Hierarchy

```text
Registers → RAM → Storage
```

* **Registers**

  * Smallest
  * Fastest
  * Inside CPU

* **RAM**

  * Larger than registers
  * Slower than registers
  * Main memory

---

# Stack Basics

* `rsp` = Stack Pointer
* CPU knows stack location through `rsp`

```asm
push rcx

; equivalent to

sub rsp, 8
mov [rsp], rcx
```

---

# Memory Access

## Read Memory

```asm
mov rax, 0x12345
mov rbx, [rax]
```

* Load value at address `0x12345` into `rbx`

## Write Memory

```asm
mov rax, 0x133337
mov [rax], rbx
```

* Store `rbx` into address `0x133337`

---

# Address Calculation

General form:

```text
base + index*scale + offset
```

Example:

```asm
mov rbx, [rsp + rax*8]
```

Common scales:

```text
1, 2, 4, 8
```

---

# LEA (Load Effective Address)

Calculates address without reading memory.

```asm
lea rbx, [rsp + rax*8 + 5]
```

* Address stored in `rbx`
* No memory access

---

# RIP-Relative Addressing

* `rip` = Instruction Pointer
* Points to next instruction

Get address:

```asm
lea rax, [rip]
lea rax, [rip+8]
```

Read near current code:

```asm
mov rax, [rip]
```

Write near current code:

```asm
mov [rip], rax
```

Uses:

* Position-independent code
* Shared libraries
* ASLR/security features

---

# Endianness

x86/x64 = **Little Endian**

```text
Value: 0x12345678

Memory:
78 56 34 12
```

* Lowest byte stored first

---

# Quick Revision

| Register | Purpose             |
| -------- | ------------------- |
| rsp      | Stack Pointer       |
| rip      | Instruction Pointer |
| rax      | General Purpose     |
| rbx      | General Purpose     |
| rcx      | General Purpose     |

### Common Syntax

```asm
[rax]          ; memory at rax
[rsp+8]        ; offset from stack
[rsp+rax*8]    ; indexed access
lea rax,[rsp]  ; calculate address
mov rax,[rsp]  ; read value
```
