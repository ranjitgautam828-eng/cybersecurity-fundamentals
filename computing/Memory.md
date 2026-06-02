# 🧠 Memory Addressing in Assembly 

---

## The Big Picture: Memory Hierarchy

Think of it like this:

```
Registers  →  fastest, tiny (inside CPU)
RAM        →  slower, bigger (main memory)
Storage    →  slowest, huge (disk)
```

> **Analogy:** Registers are like what's in your hand right now. RAM is your desk. Storage is a filing cabinet across the room.

---

## Accessing Memory (Read & Write)

### Reading from memory
```asm
mov rax, 0x12345      ; rax = the address 0x12345
mov rbx, [rax]        ; rbx = the VALUE stored at that address
```
The `[ ]` brackets mean **"go to that address and grab the value"** — like following a link.

### Writing to memory
```asm
mov rax, 0x133337     ; rax = the address
mov [rax], rbx        ; put rbx's value INTO that address
```

> **Simple rule:** `[something]` = dereference it (go there and grab/put the value).

---

## Address Calculation Formula

```
[base + index * scale + offset]
```

| Part   | Example   | Meaning                    |
|--------|-----------|----------------------------|
| base   | `rsp`     | starting address           |
| index  | `rax`     | which element              |
| scale  | `8`       | size of each element (1/2/4/8) |
| offset | `+5`      | extra shift                |

```asm
mov rbx, [rsp + rax*8]    ; go to rsp, jump rax*8 bytes ahead, read value
```

---

## LEA — Load Effective Address

`lea` **calculates** an address but does **NOT** read from memory.

```asm
lea rbx, [rsp + rax*8 + 5]   ; rbx = the calculated address itself
mov rbx, [rsp + rax*8 + 5]   ; rbx = the VALUE at that address
```

> **Think of it like:** `lea` gives you the house number. `mov [...]` actually opens the door and goes inside.

---

## The Stack

- `rsp` = **Stack Pointer** — always points to the top of the stack.

```asm
push rcx

; same as doing this manually:
sub rsp, 8        ; make room (stack grows downward)
mov [rsp], rcx    ; store rcx there
```

---

## RIP-Relative Addressing

- `rip` = **Instruction Pointer** — points to the *next* instruction being executed.

```asm
lea rax, [rip]      ; rax = address of the next instruction
mov rax, [rip]      ; read the value AT the next instruction's address
```

**Why does this matter?**
- Used in position-independent code (the program can be loaded anywhere in memory)
- Important for shared libraries and security features like ASLR

---

## Endianness (x86/x64 = Little Endian)

When a value like `0x12345678` is stored in memory, it goes in **backwards by byte**:

```
Memory address:  [0]   [1]   [2]   [3]
Stored bytes:    0x78  0x56  0x34  0x12
```

> The **least significant byte** goes first. Just something to remember when reading raw memory dumps.

---

## Dereferencing — Step by Step {MAIN}

### 1. Simple dereference — load from hardcoded address

**Your code:**
```bash
echo -e ".intel_syntax noprefix\n.global _start\n_start:\n mov rdi, qword ptr [133700]\n mov rax, 60\n syscall" > p.s
as -o p.o p.s
ld -o p p.o
```
What it does: loads the value sitting at address `133700` directly into `rdi`, then exits.

> **Note:** In GAS Intel syntax you need `qword ptr` to specify the size. NASM figures it out automatically — that's the only difference.

We already talked about `as` and `ld` in the **Assembly Language Intro**, but here's a brief reminder:

- **`as`** (GNU Assembler): Converts assembly code into binary machine code (not human-readable)
- **`ld`** (Linker): Links object files together to create an executable file we can run

---

### 2. Using a register as pointer — `rax` holds the address

**Your code:**
```bash
echo -e ".intel_syntax noprefix\n.global _start\n_start:\n mov rdi, qword ptr [rax]\n mov rax, 60\n syscall" > p.s
```
What it does: `rax` already contains the address (set up by pwn.college), so `[rax]` follows it and loads the value into `rdi`.

Same idea as example 1 — just using a register instead of hardcoding the address.

---

### 3. Dereferencing yourself — `rdi` points to its own value

**Your code:**
```bash
echo -e ".intel_syntax noprefix\n.global _start\n_start:\n mov rdi, qword ptr [rdi]\n mov rax, 60\n syscall" > p.s
```
What it does: `rdi` holds an address, and you use that address to load a new value *back into* `rdi`. Register pointing to itself — totally valid!

---

### 4. Dereference with offset — multiple values at one address

If address `133700` holds multiple values in a row:
```
133700 → 50
133701 → 42
133702 → 99
```
```asm
mov rax, [rdi]        ; gets 50  (first value)
mov rax, [rdi + 1]    ; gets 42  (second value, 1 byte ahead)
mov rax, [rdi + 2]    ; gets 99  (third value, 2 bytes ahead)
```

---

### 5. Stored addresses — following a pointer chain

**Your code:**
```bash
echo -e ".intel_syntax noprefix\n.global _start\n_start:\n mov rdi, [567800]\n mov rdi, [rdi]\n mov rax, 60\n syscall" > p.s
```
What it does:
- Step 1: load the value at `567800` into `rdi` — this value is itself another address
- Step 2: follow that address to get the actual secret value

> **Analogy:** You open a bag and find a key inside. The key opens the real locker. You had to open two things.

---

### 6. Double dereference — pointer to a pointer (pwn.college challenge)

**Your code:**
```bash
echo -e ".intel_syntax noprefix\n.global _start\n_start:\n mov rdi, qword ptr [rax]\n mov rdi, qword ptr [rdi]\n mov rax, 60\n syscall" > p.s
```
What it does:
- `rax` contains an address → follow it → get `secret2` (which is itself an address)
- Follow `secret2` → get `secret1` (the actual value you need in `rdi`)

**How the addressing works here:**
1. pwn.college sets up `rax` to point at a memory location containing another address
2. `mov rdi, [rax]` — you go to that location, grab what's there (another address) into `rdi`
3. `mov rdi, [rdi]` — now follow *that* address to get the real secret
4. `mov rax, 60` + `syscall` — exit with `rdi` as the exit code (which carries your secret)

---

## About `mov rax, 60` + `syscall`

This always appears at the end. It's **not** passing `60` as data — it means:

> "Run system call number 60" → which is `exit()` on Linux.

`rdi` is the exit code / value you pass out. `rax = 60` just tells the kernel *which* syscall to run. That's why you keep setting `rdi` to the secret — it comes out as the exit code.

---

## Quick Syntax Reference

| Syntax             | Meaning                              |
|--------------------|--------------------------------------|
| `mov rax, 5`       | rax = 5 (plain value)                |
| `mov rax, [5]`     | rax = value stored at address 5      |
| `mov [rax], rbx`   | store rbx at the address rax holds   |
| `lea rax, [rsp+8]` | rax = rsp+8 (the address, not value) |
| `[rsp + rax*8]`    | address: rsp + (rax × 8)            |

---

## Key Registers

| Register | Role                 |
|----------|----------------------|
| `rsp`    | Stack Pointer        |
| `rip`    | Instruction Pointer  |
| `rax`    | General / syscall #  |
| `rdi`    | 1st argument / exit code |
| `rbx`    | General purpose      |
| `rcx`    | General purpose      |

---

## Intel vs AT&T Syntax Note

In **GAS Intel syntax** (`.intel_syntax noprefix`), you sometimes need `qword ptr` to tell the assembler the size:

```asm
mov rdi, qword ptr [133700]
```

In **NASM**, the size is inferred automatically. Same logic, less typing. You're using GAS, so remember `qword ptr` when loading from a raw address.

---

## Practice Checklist 🎯 (pwn.college — Computer Memory)

- [ ] Load from hardcoded address
- [ ] Load using `rax` as pointer
- [ ] Load using `rdi` pointing to itself
- [ ] Load with offset (`[rdi + 1]` etc.)
- [ ] Follow a pointer chain (2 loads)
- [ ] Double dereference (`rax` → address → address → secret)
