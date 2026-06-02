# Memory & Addressing Notes

## Human Memory vs Computer Memory

Think of RAM like human short-term memory.

* We store most knowledge in long-term memory (books, journals, Wikipedia, experiences).
* When solving a problem, we load only the needed information into short-term memory.
* Short-term memory is limited (about 5–9 items).
* After using the information, new knowledge is stored back into long-term memory.

Computer memory works similarly:

```text
Storage (SSD/HDD)
      ↓
     RAM
      ↓
  Registers
```

* Registers = what the CPU is actively thinking about right now.
* RAM = information available for quick access.
* Storage = long-term memory.

Smaller memory = faster access.
Larger memory = slower access.

---

## Stack

The stack is a special area of memory used for temporary data.

`rsp` always points to the top of the stack.

```asm
push rcx
```

really means:

```asm
sub rsp, 8
mov [rsp], rcx
```

Idea:

* Move stack pointer down.
* Put value there.

---

## Memory Access

Registers hold data directly.

Memory holds data at addresses.

To access memory:

```asm
mov rbx, [rax]
```

Read:

* Go to address stored in `rax`.
* Copy value into `rbx`.

```asm
mov [rax], rbx
```

Write:

* Go to address stored in `rax`.
* Store `rbx` there.

Think:

```text
Register = value
[Register] = value at address
```

---

## Address Calculation

Memory locations are often calculated rather than hardcoded.

```asm
[rsp + rax*8]
```

means:

```text
Base address
+ offset
+ scaling
```

Useful for:

* Arrays
* Stack frames
* Data structures

Common formula:

```text
base + index*scale + offset
```

---

## LEA

`lea` calculates an address without reading memory.

```asm
lea rbx, [rsp + rax*8]
```

Memory is NOT accessed.

Think:

```text
lea = give me the address
mov = give me the value
```

---

## RIP Addressing

`rip` points to the next instruction.

```asm
lea rax, [rip]
```

Gets the address near the current code.

Useful because:

* Program doesn't need fixed memory locations.
* Code can run anywhere in memory.
* Helps modern security features.

---

## Endianness

Memory stores bytes one at a time.

x86 uses Little Endian.

```text
0x12345678
```

stored as:

```text
78 56 34 12
```

Rule:

```text
Lowest byte first.
```

---

## Quick Mental Models

```text
Registers = CPU's current thoughts

RAM = working memory

Storage = long-term memory
```

```text
mov = get/store value

lea = get address
```

```text
rax      -> value

[rax]    -> memory at address rax
```

```text
rsp = top of stack

rip = current code location
```

in here we will practise acessing dat stored in memory:
we normally do 
mov rex, 3112
for moving value but if you want to see that for adddess:
mov rex, [3112] now 3112 is not value but the address which content value
