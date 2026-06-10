# Assembly Assortment — Reverse Calculation

> **Where this fits:** We know how to read disassembly with `objdump` and step through code in GDB. This section is about reversing what a binary does to our input — figuring out what to pass as an argument by working backwards from the comparison in the disassembly.

---

## Table of Contents

- [The Pattern](#the-pattern)
- [Add — reverse by subtracting](#add--reverse-by-subtracting)
- [Sub — reverse by adding](#sub--reverse-by-adding)
- [XOR — reverse by XORing again](#xor--reverse-by-xoring-again)
- [Loops on Data — string in .rodata](#loops-on-data--string-in-rodata)
- [Security Context](#-security-context)
- [Practice Checklist](#practice-checklist)

---

## The Pattern

Every reverse challenge follows the same structure in disassembly:

```
operation  BYTE PTR [rax], <key>    ; transform our input
cmp        BYTE PTR [rax], <target> ; compare result to expected value
```

Our job: work backwards from `<target>` using the inverse of `<operation>` to find what input produces it.

---

## Add — reverse by subtracting

**Disassembly:**
```asm
add BYTE PTR [rax], 0x2a
cmp BYTE PTR [rax], 0x96
```

**What it does:** adds `0x2a` (42) to our input byte, then checks if the result equals `0x96` (150).

**Reverse:**
```
input + 42 = 150
input = 150 - 42 = 108 = 0x6c = 'l'
```

```bash
/challenge/reverse-me l
```

> **Personal note:** I ran this three times and couldn't get the flag — turned out I was in privilege mode and couldn't see the output. If something's not working, check which mode you're in before assuming your math is wrong.

---

## Sub — reverse by adding

**Disassembly:**
```asm
sub BYTE PTR [rax], 0x1a
cmp BYTE PTR [rax], 0x38
```

**What it does:** subtracts `0x1a` (26) from input, checks if result equals `0x38` (56).

**Reverse:**
```
input - 26 = 56
input = 56 + 26 = 82 = 0x52 = 'R'
```

```bash
/challenge/reverse-me R
```

---

## XOR — reverse by XORing again

XOR is different from add/sub. It works bit by bit: result is `1` if exactly one of the two bits is `1`, otherwise `0`.

The key property: **XOR is its own inverse.** XOR a value twice with the same key and get the original back.

```
A XOR K = B
B XOR K = A      ← XOR again with same key to reverse it
```

**Disassembly:**
```asm
xor BYTE PTR [rax], 0x93
cmp BYTE PTR [rax], 0xf1
```

**What it does:** XORs your input with `0x93`, checks if result equals `0xf1`.

**Reverse:**
```
input XOR 0x93 = 0xf1
input = 0xf1 XOR 0x93 = 0x62 = 'b'
```

```bash
/challenge/reverse-me b
```

### XOR bit-by-bit example

```
0xf1 = 1111 0001
0x93 = 1001 0011
XOR  = 0110 0010 = 0x62 = 'b'
```

Each bit: 1 if different, 0 if same.

---
Even or odd
What we did: We checked whether a number is even or odd, but backwards.

The trick:

The lowest bit (last digit in binary) tells you: 1 = odd, 0 = even

But the challenge wanted: return 1 for even, 0 for odd

So we:

Isolated the lowest bit (got 1 for odd, 0 for even)

Flipped it (1 became 0, 0 became 1)

Example:

Number 4 (even) → lowest bit = 0 → flip → return 1 ✓

Number 7 (odd) → lowest bit = 1 → flip → return 0 ✓

That's all. Just a bitwise AND followed by a NOT (XOR with 1).

code:
.intel_syntax noprefix
.text
.global solve

solve:
    mov rax, rdi
    and rax, 1
    xor rax, 1
    ret

---

Masking Bits:
We wrote a function that extracts only the lowest 8 bits (the least significant byte) from a 64-bit input value. We did this by moving the input from rdi into rax, then using a bitwise AND with the mask 0xFF (which is 255 in decimal, or 11111111 in binary). This mask has 1's in the lowest 8 positions and 0's everywhere else, so when we AND it with the original value, only the lowest byte survives—all higher bits get forced to zero. The result in rax is the original number's lowest byte, isolated and ready to return.

code:
.intel_syntax noprefix
.text
.global LOBYTE

LOBYTE:
    mov rax, rdi
    and rax, 0xFF
    ret

save as lobyte and run it after as and ld

---
lowercase a string
code:
.intel_syntax noprefix
.text
.global chr_lower

chr_lower:
    mov rax, rdi
    or rax, 0x20
    ret

We wrote a function that converts an uppercase ASCII letter to its lowercase equivalent by setting bit 5 (the 6th bit) of the character. In ASCII, uppercase letters (A-Z, hex 0x41-0x5A) differ from their lowercase counterparts (a-z, hex 0x61-0x7A) by exactly one bit—bit 5 (value 0x20 or 32 in decimal). We moved the input character from rdi into rax, then used a bitwise OR with the mask 0x20. This forces bit 5 to 1 while leaving all other bits unchanged, turning, for example, 0x41 ('A') into 0x61 ('a'), 0x48 ('H') into 0x68 ('h'), and so on. The result in rax is the lowercase letter, ready to return.

---

Uppercase a String
We wrote a function that converts a lowercase string to uppercase by modifying it directly in memory. The function takes a pointer to the string in rdi, then loops through each byte until it hits the NUL terminator (value 0). For each character, we load the byte, use AND with 0xDF (which is 11011111 in binary) to clear bit 5 (the same bit we set earlier for lowercasing), store the uppercased character back, and move to the next byte. The AND operation preserves all bits except the case bit—clearing it changes 'a' (0x61) to 'A' (0x41), 'b' to 'B', and so on. The function returns nothing (void) because the string is uppercased in place.
code:
.intel_syntax noprefix
.text
.global str_upper

str_upper:
    mov rcx, rdi          ; rcx points to the current character
    
loop:
    mov al, [rcx]         ; load byte from memory
    test al, al           ; check if it's NUL (0)
    jz done               ; if NUL, we're done
    
    and al, 0xDF          ; clear bit 5 (0x20) to uppercase
    mov [rcx], al         ; store back to memory
    
    inc rcx               ; move to next character
    jmp loop              ; repeat
    
done:
    ret

---

Swap case
.intel_syntax noprefix
.text
.global str_swapcase

str_swapcase:
    mov rcx, rdi

loop:
    mov al, [rcx]
    test al, al
    jz done
    xor al, 0x20
    mov [rcx], al
    inc rcx
    jmp loop

done:
    ret

We wrote a function that flips the case of every letter in a string—uppercase becomes lowercase and lowercase becomes uppercase—by XORing each character with 0x20. The function takes a pointer to the string in rdi, then loops through each byte until it hits the NUL terminator (value 0). For each character, we load the byte into al, XOR it with 0x20 (which toggles bit 5—the case bit), store the flipped character back to memory, and advance to the next byte. XOR works as a toggle: if the bit is 1, XOR with 1 makes it 0; if it's 0, XOR with 1 makes it 1. Since the input strings contain only letters (A-Z and a-z), flipping bit 5 safely converts 'A' (0x41) to 'a' (0x61), 'z' (0x7A) to 'Z' (0x5A), and everything in between. The function returns nothing because the string is modified directly in place.

---

shifting Left
code:
.intel_syntax noprefix
.text
.global solve

solve:
    mov rax, rdi
    shl rax, 4
    ret

We wrote a function that takes a 64-bit value and shifts it left by 4 bits, which is the same as multiplying it by 16. The input comes in rdi, we move it to rax, then use shl rax, 4 to shift every bit 4 positions to the left. Each left shift doubles the value, so 4 shifts multiplies by 2⁴ = 16. For example, 3 becomes 48 (3 × 16), and 5 becomes 80 (5 × 16). Zeros fill in from the right, and any bits shifted off the top are lost. The result is returned in rax.

---

Shifting Right
.intel_syntax noprefix
.text
.global solve

solve:
    mov rax, rdi
    shr rax, 8
    and rax, 0xFF
    ret

We wrote a function that extracts the second byte (bits 8 through 15) from a 64-bit value and returns it as a number between 0 and 255. The input comes in rdi, we move it to rax, then shift it right by 8 bits using shr rax, 8. This moves the second byte down to the lowest 8 bits (bits 0-7) where it's easy to isolate. Then we use and rax, 0xFF to mask away everything except those lowest 8 bits, discarding any higher bytes that shifted down. The result in rax is just the original second byte. For example, if the input is 0x1234567890ABCDEF, shifting right by 8 brings 0xCD (the second byte) to the bottom, masking removes everything else, and we return 0xCD.

---
## Loops on Data — string in .rodata

Some challenges compare our input against a **hardcoded string** stored in the binary's `.rodata` section (read-only data) — not as an immediate value in the instructions. We won't see the characters directly in `cmp` — the binary loops through memory.

Three ways to find the string:

### 1. strings command (fastest)

```bash
strings /challenge/reverse-me
```

Lists all printable strings in the binary. Lots of noise — look for something short that looks like a password.

### 2. Dump .rodata section

```bash
objdump -s -j .rodata /challenge/reverse-me
```

Shows raw hex + ASCII of the read-only data section. The password will be readable in the ASCII column.

### 3. GDB live inspection

```bash
gdb /challenge/reverse-me
(gdb) starti
(gdb) stepi    # step until we are at the comparison
(gdb) x/s $rax # print the string the register points to
```

Then run with what we can found:

```bash
/challenge/reverse-me <found_string>
```

Quote it if it has spaces or special characters.

---

## 🔐 Security Context

**This is literally how you crack hardcoded passwords.** Any program that checks a password by transforming our input and comparing — add, sub, XOR, whatever — can be reversed with `objdump` and basic math. No brute force needed.

**XOR is everywhere in simple obfuscation.** Malware often XORs strings with a single-byte key to hide readable content from `strings`. You reverse it exactly like above — XOR the obfuscated bytes with the key to recover the original. Same operation, same inverse property.

**`.rodata` is a real hiding spot.** Strings in `.rodata` don't appear in the instruction stream so they're slightly less obvious than immediates — but `strings` and `objdump -s` find them instantly. "Security through obscurity" doesn't work.

> **Connecting the dots:** `objdump` and GDB from `Software_Introspection.md` → reading `cmp` instructions → number conversion from `Nibbling_on_Numbers.md` (hex → decimal → ASCII) → working backwards through the operation. All three previous skills in one challenge.

---

## Practice Checklist

- [ ] Reverse an add challenge: find `cmp` target, subtract the key, convert to ASCII
- [ ] Reverse a sub challenge: find `cmp` target, add the key back, convert to ASCII
- [ ] Reverse an XOR challenge: XOR the target with the key, convert result to ASCII
- [ ] Do the XOR manually bit-by-bit at least once to understand it
- [ ] Find a `.rodata` password using `strings`
- [ ] Find a `.rodata` password using `objdump -s -j .rodata`
- [ ] Find a `.rodata` password by stepping in GDB and using `x/s`
