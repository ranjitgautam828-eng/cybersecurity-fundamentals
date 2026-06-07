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
