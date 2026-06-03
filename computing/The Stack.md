# The Stack

> Notes on stack memory, offsets, program arguments, and pop — for x86-64 Linux assembly.

---

## Table of Contents

- [What is the Stack?](#what-is-the-stack)
- [Stack Offsets](#stack-offsets)
- [Program Arguments on the Stack](#program-arguments-on-the-stack)
- [Popping from the Stack](#popping-from-the-stack)

---

## What is the Stack?

The stack is a **pre-allocated region of memory** available to your program without any setup required. It is used as scratch space during execution and holds data about how the program was launched, accumulating more data as the program runs.

- The **`rsp` register** (Stack Pointer) always points to the **top of the stack**
- You get this memory for free — no setup needed

### Example: Read `argc` and exit with its value

```asm
# p.s
.intel_syntax noprefix
.global _start
_start:
    mov rdi, qword ptr [rsp]   # load argc from top of stack
    mov rax, 60                # syscall: exit
    syscall
```

```bash
echo -e ".intel_syntax noprefix\n.global _start\n_start:\n mov rdi, qword ptr [rsp]\n mov rax, 60\n syscall" > p.s
as -o p.o p.s && ld -o p p.o && /challenge/check p
```

> **Note:** `qword ptr` is used to explicitly denote a 64-bit (8-byte) memory read. It is optional in most exercises but good practice for clarity.

> **Confusion clarified:** No need to use `mov` or anything else when using `pop` — `pop` is the instruction itself, and it automatically takes the top value of the stack. You don't assign the argument value manually; `pop rdi` handles it all in one step.

---

## Stack Offsets

When you need a value that isn't at the very top of the stack, use an **offset** with the syntax `[rsp + value]`.

Each slot in the stack is **8 bytes apart**.

> **Confusion clarified:** When accessing the stack from the top [rsp], the first step away is `[rsp+8]`, the next is `[rsp+16]`, and so on — each slot is 8 bytes further.

### Example: Read value at `[rsp+8]`

```asm
# p.s
.intel_syntax noprefix
.global _start
_start:
    mov rdi, qword ptr [rsp+8]  # load value 8 bytes into the stack
    mov rax, 60
    syscall
```

```bash
echo -e ".intel_syntax noprefix\n.global _start\n_start:\n mov rdi, qword ptr [rsp+8]\n mov rax, 60\n syscall" > p.s
as -o p.o p.s && ld -o p p.o && /challenge/check p
```

---

## Program Arguments on the Stack

At program entry on **x86-64 Linux**, the stack is laid out as follows:

| Offset         | Contents                            |
|----------------|-------------------------------------|
| `[rsp]`        | `argc` — argument count             |
| `[rsp+8]`      | `argv[0]` — pointer to program name |
| `[rsp+16]`     | `argv[1]` — pointer to first argument |
| `...`          | `...`                               |
| `[rsp+8*argc]` | `NULL` terminator                   |

Each `argv[i]` is an **8-byte pointer** to a null-terminated string stored elsewhere in memory.  
To get the actual string data, you must **dereference the pointer**.

### Example: Exit with the integer value of the first argument

```asm
# p.s
.intel_syntax noprefix
.global _start
_start:
    mov rdi, [rsp+16]   # load pointer to argv[1]
    mov rdi, [rdi]      # dereference: load the actual value
    mov rax, 60
    syscall
```

```bash
echo -e ".intel_syntax noprefix\n.global _start\n_start:\n mov rdi, [rsp+16]\n mov rdi, [rdi]\n mov rax,60\n syscall" > p.s
as -o p.o p.s && ld -o p p.o && /challenge/check p
```

> **Remember:**
> - `[rsp]` → `argc`
> - `[rsp+8]` → `argv[0]` (program name)
> - `[rsp+16]` → `argv[1]` (first argument)

---

## Popping from the Stack

The stack follows **LIFO** (Last In, First Out) — like stacking books, you always remove from the top first.

### `pop rdi` does two things atomically:

1. Reads the value at `[rsp]` into `rdi` (same as `mov rdi, [rsp]`)
2. Advances `rsp` by 8 (moves the stack pointer to the next slot)

### Memory diagram

**Before `pop rdi`:**

```
Address    │ Contents
+───────────────────────+
│ ...        │ ...      │
+───────────────────────+
│ 1337000    │ 3        │  ◀── rsp points here (argc = 3)
+───────────────────────+
│ 1337008    │ ???      │
+───────────────────────+

Register │ Contents
+────────────────────────+
│ rsp     │ 1337000      │
+────────────────────────+
│ rdi     │ 0            │
+────────────────────────+
```

**After `pop rdi`:**

```
Address    │ Contents
+───────────────────────+
│ ...        │ ...      │
+───────────────────────+
│ 1337000    │ 3        │
+───────────────────────+
│ 1337008    │ ???      │  ◀── rsp now points here
+───────────────────────+

Register │ Contents
+────────────────────────+
│ rsp     │ 1337008      │
+────────────────────────+
│ rdi     │ 3            │  ◀── received popped value
+────────────────────────+
```

### Example: Pop `argc` and exit with its value

```asm
# p.s
.intel_syntax noprefix
.global _start
_start:
    pop rdi          # pop argc from stack into rdi; rsp advances by 8
    mov rax, 60      # syscall: exit
    syscall
```

```bash
echo -e ".intel_syntax noprefix\n.global _start\n_start:\n pop rdi\n mov rax,60\n syscall" > p.s
as -o p.o p.s && ld -o p p.o && /challenge/check p
```

---

## Quick Reference

| Instruction / Syntax    | Meaning                                              |
|-------------------------|------------------------------------------------------|
| `rsp`                   | Stack pointer — always points to top of stack        |
| `[rsp]`                 | Dereference top of stack (read value at that address)|
| `[rsp+8]`               | Read value 8 bytes below the top                     |
| `qword ptr [rsp]`       | Explicit 64-bit read (optional but clear)            |
| `pop rdi`               | Read `[rsp]` into `rdi`, then `rsp += 8`            |
| `argc`                  | Always at `[rsp]` on program entry                  |
| `argv[0]`               | Pointer at `[rsp+8]`                                |
| `argv[1]`               | Pointer at `[rsp+16]`                               |
