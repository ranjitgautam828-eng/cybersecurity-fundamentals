# Nibbling on Numbers

> **Where this fits:** This is the math layer under everything else. Every value in a register, every memory address, every ASCII character — it's all just bits. This section is about reading those bits four different ways: binary, unsigned decimal, signed decimal, and hex. Once this clicks, reading raw memory dumps and GDB output makes way more sense.

---

## Table of Contents

- [The Four Ways to Read One Byte](#the-four-ways-to-read-one-byte)
- [Binary to Unsigned Decimal](#binary-to-unsigned-decimal)
- [Signed Numbers — Two's Complement](#signed-numbers--twos-complement)
- [The Quick Formula](#the-quick-formula)
- [Hex Encoding](#hex-encoding)
- [Mixed Conversions — the full picture](#mixed-conversions--the-full-picture)
- [How to Run the Challenges](#how-to-run-the-challenges)
- [Connecting Back](#connecting-back)
- [Practice Checklist](#practice-checklist)

---

## The Four Ways to Read One Byte

One byte = 8 bits. That same byte can be written four ways and they all mean the same underlying value:

| Format | Example | Range |
|---|---|---|
| Binary | `11000101` | 8 digits of 0s and 1s |
| Unsigned decimal | `197` | 0 to 255 |
| Signed decimal | `-59` | -128 to 127 |
| Hex | `C5` | 00 to FF |

All four of those are the same byte. Just different ways to write it.

> **Why does this matter?** GDB shows memory in hex. Disassembly shows immediates in hex. ASCII characters are stored as decimal numbers. Syscall return values can be negative (errors). You need to move between these four representations fast.

---

## Binary to Unsigned Decimal

Each bit position has a value. From right to left: 1, 2, 4, 8, 16, 32, 64, 128.

```
Bit position:  7    6    5    4    3    2    1    0
Value:        128   64   32   16    8    4    2    1

11000011:
  1×128 + 1×64 + 0×32 + 0×16 + 0×8 + 0×4 + 1×2 + 1×1
= 128 + 64 + 2 + 1
= 195
```

If the top bit (bit 7) is `0`, the number is positive and both unsigned and signed readings agree.
If the top bit is `1`, the number could be negative — depends on whether you're reading it as signed.

---

## Signed Numbers — Two's Complement

Same bits, different interpretation. When the top bit is `1`, the number is negative in signed reading.

**Unsigned reading:** just add up the bits normally (ignore the sign).

**Signed reading:** `unsigned value - 256`

```
11000011  →  unsigned = 195
             signed   = 195 - 256 = -61
```

```
11101110  →  unsigned = 238
             signed   = 238 - 256 = -18
```

If the top bit is `0`, skip the signed step — it's just the same positive number either way.

```
01110111  →  top bit = 0, positive
             both unsigned and signed = 119
```

> **The formula only works for 8-bit (1 byte).** For 16-bit subtract 65536. For 32-bit subtract 4294967296. The pattern is always `unsigned - 2^(number of bits)`. But for pwn.college the challenges are 8-bit so `unsigned - 256` is all you need.

---

## The Quick Formula

```
Top bit = 0  →  positive, one answer: just add the bits
Top bit = 1  →  negative, two answers:
                  unsigned = add the bits normally
                  signed   = unsigned - 256
```

### Worked examples from the challenges

```
11000011  →  top bit 1 → unsigned: 128+64+2+1 = 195 → signed: 195-256 = -61  ✓
11101110  →  top bit 1 → unsigned: 128+64+32+8+4+2 = 238 → signed: 238-256 = -18  ✓
10110000  →  top bit 1 → unsigned: 128+32+16 = 176 → signed: 176-256 = -80  ✓
01110111  →  top bit 0 → positive → 64+32+16+4+2+1 = 119  ✓
00111001  →  top bit 0 → positive → 32+16+8+1 = 57  ✓
```

> **Personal note:** I got caught out a few times: when the top bit is 1 and they ask for unsigned, just add the bits — don't subtract 256. Unsigned always just adds. The signed formula is only for the signed reading.

---

## Hex Encoding

One hex digit = 4 bits. Two hex digits = one byte.

Split the 8 bits into two groups of 4, convert each half:

```
11110001
→ 1111  0001
→   F     1
→ F1
```

Hex digit reference:

| Binary | Hex | Decimal |
|---|---|---|
| 0000 | 0 | 0 |
| 0001 | 1 | 1 |
| 0010 | 2 | 2 |
| 0011 | 3 | 3 |
| 0100 | 4 | 4 |
| 0101 | 5 | 5 |
| 0110 | 6 | 6 |
| 0111 | 7 | 7 |
| 1000 | 8 | 8 |
| 1001 | 9 | 9 |
| 1010 | A | 10 |
| 1011 | B | 11 |
| 1100 | C | 12 |
| 1101 | D | 13 |
| 1110 | E | 14 |
| 1111 | F | 15 |

You already used this table in `Software_Introspection.md`. Same table, now you're using it to go both directions.

---

## Mixed Conversions — the full picture

The final challenge gives you one byte in one format and asks for all three others. This is the real test — going in any direction between all four representations.

### Example from the challenge

```
Input: c5  [hex]
  → signed decimal: -59   ✓
  → binary:         11000101   ✓
  → unsigned:       197   ✓
```

Working it out:

```
c5 hex
→ C = 1100, 5 = 0101
→ binary: 11000101

→ unsigned: 128+64+4+1 = 197

→ top bit is 1, so signed: 197 - 256 = -59
```

### Another example

```
Input: -111  [signed decimal]
  → unsigned: 145   ✓
  → binary:   10010001   ✓
  → hex:      91   ✓
```

Working it out:

```
signed -111
→ unsigned: -111 + 256 = 145  (reverse of the signed formula)

→ 145 = 128+16+1 = 10010001 in binary

→ 1001 = 9, 0001 = 1 → hex: 91
```

> Going from signed to unsigned is just `signed + 256`. The formula works both ways.

---

## How to Run the Challenges

Most challenges in this section tell you what to do. But for the mixed conversions one there's no prompt — you just run it:

```bash
cd /challenge
ls
/challenge/mixed
```

That's it. Just run it directly.

---

## Connecting Back

This section feels like pure math but it shows up constantly:

**In `Software_Introspection.md`** — GDB shows register values in hex. `0x1bf6` → you convert to decimal to get the actual number. Same hex table.

**In `Memory.md`** — memory dumps show bytes as hex pairs. `0x48 0x45 0x4c 0x4c 0x4f` → you convert each to ASCII to read "HELLO".

**In `Assembly_Language_Intro.md`** — immediates in assembly are often hex (`mov rdi, 0x2a`). `0x2a` = 42 decimal.

**In `Control_Flow.md`** — `objdump` shows comparison values in hex. `cmp rdi, 0x1bf6` → you convert `0x1bf6` to decimal to know what the program is checking.

**In `File_Descriptors_IO.md`** — ASCII characters stored in memory are just their decimal/hex values. `'H'` = `0x48` = `72`.

> **This is the conversion layer that makes everything else readable.** You don't need to be fast at it yet — just accurate. Speed comes with repetition.

---

## Practice Checklist

- [ ] Convert 8-bit binary to unsigned decimal (add the bits)
- [ ] Convert 8-bit binary to signed decimal (unsigned - 256 if top bit is 1)
- [ ] Convert decimal to binary (reverse: subtract powers of 2)
- [ ] Convert binary to hex (split into 4-bit groups)
- [ ] Convert hex to binary (expand each digit to 4 bits)
- [ ] Go from signed decimal to unsigned (add 256)
- [ ] Complete the mixed conversions challenge: `/challenge/mixed`
- [ ] In GDB, read a hex register value and convert it to decimal mentally
