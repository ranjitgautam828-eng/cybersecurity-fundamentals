# Endian Escapades

After learning about control flow, I moved into understanding **endianness**, memory layouts, and how programs store data.

## Sign Extention
### The Challenge:
The tricky part wasn’t just loading a byte—it was handling negative numbers correctly. I realized that if I just loaded 0xff (which is -1 as a signed byte) and padded it with zeroes, I’d incorrectly get 255 instead of -1. The real challenge was making sure the sign bit (the leftmost bit) gets copied into all the new high bits when expanding to 64 bits.

### The Problem:
Given a pointer to a single byte in rdi, I needed to read that byte, treat it as a signed 8‑bit value, and return its fully sign‑extended 64‑bit equivalent in rax.

### How We Solved It:
We skipped the manual bit-twiddling and used the CPU’s built‑in movsx (Move with Sign‑eXtend) instruction. It takes one byte from [rdi], looks at its sign bit, and automatically fills the rest of rax with either all 0s (for positive) or all 1s (for negative). One instruction did exactly what we needed. We just wrapped it in a function with .global solve so the grader could call it, and returned with ret.

code:
```
.intel_syntax noprefix
.global solve
.section .text

solve:
    movsx rax, BYTE PTR [rdi]
    ret
```

--- 

## Little Endian

Most modern CPUs use **Little Endian (LE)** format.

In Little Endian systems, the **least significant byte** is stored first in memory.

Example:

| Value        | Big Endian    | Little Endian |
| ------------ | ------------- | ------------- |
| `0x1234`     | `12 34`       | `34 12`       |
| `0x44434241` | `44 43 42 41` | `41 42 43 44` |

Modern x86 and x86-64 systems use Little Endian.

### Why Little Endian?

* Arithmetic naturally starts from the smallest digits.
* Reading smaller pieces of a larger value is easier.
* The address of a value is the address of its lowest byte.

---

## Common Data Sizes

| Name                | Bits | Bytes |
| ------------------- | ---- | ----- |
| Byte                | 8    | 1     |
| Word                | 16   | 2     |
| Dword (Double Word) | 32   | 4     |
| Qword (Quad Word)   | 64   | 8     |

Assembly examples:

```asm
mov al, [rdi]     ; byte
mov ax, [rdi]     ; word
mov eax, [rdi]    ; dword
mov rax, [rdi]    ; qword
```

Memory write examples:

```asm
mov BYTE PTR [rdi], 0x11
mov WORD PTR [rdi], 0x1122
mov DWORD PTR [rdi], 0x11223344
mov QWORD PTR [rdi], 0x1122334455667788
```

---

## Word Size Confusion

One confusing thing is the term **word**.

In x86 assembly:

* Word = 16 bits

But in computer architecture:

* Machine word = natural register size of the CPU

Examples:

* 32-bit CPU → machine word is 32 bits
* 64-bit CPU → machine word is 64 bits

To avoid confusion, people often use:

* Byte = 8 bits
* Short = 16 bits
* Dword = 32 bits
* Qword = 64 bits

---

# Challenges

## Little Endian Bytes

### Goal

Find the value loaded into a register and convert it into readable text.

### Process

1. Use:

```bash
objdump -d -M intel /challenge/challenge
```

2. Find the `movabs rbx` value.

Example:

```asm
movabs rbx, 0x6f6c6c6548
```

3. Reverse the bytes because the system is Little Endian.

4. Convert the bytes into ASCII characters.

5. Submit the resulting string.

### Key Idea

Little Endian stores bytes in reverse order, so decoding requires reversing them first.

---

## Qword by Qword

### Goal

Decode two Qword values and combine them.

### Process

1. Find both Qword values.
2. Reverse each value from Little Endian format.
3. Convert them to ASCII.
4. Join the results together.
5. Submit the final string.

### Important

The challenge generates new values after a restart.

If the machine restarts:

* Re-run the challenge.
* Extract the new values.
* Decode everything again.

---

## Dword by Dword / Word by Word / Byte by Byte

These challenges follow the same idea.

### Process

1. Extract the values.
2. Reverse according to Little Endian.
3. Convert to ASCII.
4. Join the pieces together.
5. Submit the final result.

The only difference is the size of the chunks being used.

---

## Cracking a Struct

Real programs rarely read memory using one single size.

Instead, they use **structures (structs)**.

A struct contains multiple fields of different sizes stored one after another.

Example:

```c
struct data {
    char a;
    short b;
    int c;
};
```

### Process

1. Identify each field size.
2. Decode each field individually.
3. Apply Little Endian rules when needed.
4. Convert values to characters.
5. Combine all pieces together.

### Key Idea

Treat every field separately before joining them into the final answer.

---

## Scrambled Struct

This challenge is similar to Cracking a Struct but the values are not stored in order.

### What to Look For

Memory accesses:

```asm
mov eax, [rdi+4]
mov bx, [rdi+2]
mov cl, [rdi]
```

The offsets (`+4`, `+2`, etc.) show where each piece belongs.

### Important Observation

I originally thought `cmp` was only used for comparisons.

However, the value being compared often reveals useful constant data that belongs to the final answer.

Example:

```asm
cmp eax, 0x41424344
```

The constant itself may be part of the solution.

### Process

1. Identify all offsets.
2. Determine the size of each field.
3. Decode each value.
4. Place each piece in the correct position.
5. Combine everything together.

### Key Idea

The challenge is not only decoding values but also rebuilding the correct order of the structure.

---

# Main Lessons Learned

* Modern systems use Little Endian.
* Always pay attention to byte order.
* Byte, Word, Dword, and Qword represent different data sizes.
* Structs store multiple fields of different sizes together.
* Memory offsets are extremely important.
* `objdump` is often enough to recover hidden values.
* Many reversing challenges are simply about understanding how data is stored in memory and reconstructing it correctly.
