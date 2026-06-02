# CPU Architecture & Assembly Language

## What is a CPU?

The CPU is the main processor of a computer. You give it instructions using **assembly language** — every program (Python, C++, etc.) eventually gets translated into assembly for the CPU to run.

---

## CPU Architectures

There are two major ones:

- **x86** — made by Intel, most common in learning/study material
- **ARM** — also very common; together with x86, these cover most CPUs worldwide

---

## Assembly Language Basics

Assembly has two parts:

- **Operands** — the data being worked on
- **Operations** — actions like `ADD`, `SUB`, `MULT`

Assembly is essentially a human-readable version of the binary code the CPU actually reads.

---

## Registers

Registers are tiny storage slots inside the CPU.

- Very limited — usually only **10–20 general purpose registers**
- Example of moving a value into a register:

```asm
mov rax, 42
```

---

## Moving Between Registers

You can also move a value **from one register to another** using the same `mov` instruction — just use two registers instead of a number.

```asm
mov rdi, rsi   ; copies the value inside rsi into rdi
```

The value in `rsi` doesn't disappear — it gets **copied** into `rdi`. Think of it like copy-paste, not cut-paste.

### Example

If `rsi` already holds the exit code (put there by the OS or whatever ran your program), you can forward it directly:

```asm
mov rdi, rsi   ; copy rsi's value into rdi (this becomes our exit code)
mov rax, 60    ; syscall 60 = exit
syscall        ; exit with whatever value was in rsi
```

To try it out:

```bash
echo -e "mov rdi, rsi\nmov rax, 60\nsyscall" > p.s
as -o p.o p.s
ld -o p p.o
./p
echo $?
```

> The exit code you get depends on what value `rsi` happened to hold when the program started — that's set by the OS/shell at launch.

---

## Syscalls (System Calls)

Just like assembly lets your program talk to the CPU, **syscalls** let your program talk to the **operating system**.

- Each syscall has a number (starts from 0)
- Linux has around **330 syscalls**
- You put the syscall number in `rax`, then run `syscall`

```asm
mov rax, 42   ; pick syscall #42
syscall       ; run it
```

---

## Exit Codes

To exit a program, you use **syscall 60** (exit). It takes one parameter — the exit code — passed through `rdi`.

```asm
mov rax, 60   ; syscall number for exit
mov rdi, 0    ; exit code (0 = success)
syscall
```

Change `rdi` to any number to set a custom exit code.

---

## Building an Executable

Three steps to go from code → runnable program:

### 1. Write your assembly file

Save it as `program.s`. Always start with the Intel syntax directive:

```asm
.intel_syntax noprefix
.global _start
_start:
    mov rdi, 42
    mov rax, 60
    syscall
```

> `.global _start` and `_start:` tell the linker where your program begins. Without it, you'll get a warning (it still works, but this silences it).

### 2. Assemble into an object file

```bash
as -o program.o program.s
```

This converts your assembly into binary code, but it's not runnable yet.

### 3. Link into a final executable

```bash
ld -o program program.o
```

This links the object file into a real executable.

### Run it

```bash
./program
echo $?   # prints the exit code
```

Output:
```
42
```

---

## Full Example — Explained

Goal: write a program that exits with code **42**.

---

### The Assembly Code (`program.s`)

```asm
.intel_syntax noprefix   ; (1)
.global _start           ; (2)
_start:                  ; (3)
    mov rdi, 42          ; (4)
    mov rax, 60          ; (5)
    syscall              ; (6)
```

**(1) `.intel_syntax noprefix`**
This is a directive (an instruction to the assembler, not to the CPU).
It tells `as` — the assembler tool — that your code is written in Intel syntax.
Without this line, the assembler assumes AT&T syntax, which looks different and would misread your code.

**(2) `.global _start`**
This makes the `_start` label visible to the **linker** (`ld`).
The linker is the tool that turns your assembled code into a final runnable file — it needs to know where your program starts. `.global` exposes the label so the linker can see it.

**(3) `_start:`**
This is a **label** — it's like putting a sticky note on this line of code saying "this is where the program begins."
The linker looks for this exact label to know the entry point of your program.

**(4) `mov rdi, 42`**
Put the value `42` into the register `rdi`.
`rdi` is where the **first parameter** of a syscall goes.
For the exit syscall, that parameter is the **exit code** — so we're saying "exit with code 42."

**(5) `mov rax, 60`**
Put the value `60` into the register `rax`.
`rax` is always where the **syscall number** goes.
Syscall 60 on Linux = `exit`. So we're telling the OS: "I want to call the exit function."

> Notice: we load `rdi` before `rax`. Order doesn't matter here — both registers just need to be set before we hit `syscall`. It's just a style choice.

**(6) `syscall`**
This is the trigger — it says "go do it now."
The CPU looks at `rax` to know which syscall to run (60 = exit), then looks at `rdi` for the parameter (42 = exit code), and hands control to the OS. The program ends here.

---

### Step 1 — Assemble

```bash
as -o program.o program.s
```

- `as` reads your `program.s` file
- Converts it into binary machine code
- Outputs `program.o` — an **object file**

The object file has your code in binary, but it's not a complete runnable program yet. Think of it as a compiled piece — it still needs to be packaged.

---

### Step 2 — Link

```bash
ld -o program program.o
```

- `ld` takes your object file(s) and links them into a final executable
- Outputs a file called `program`
- This is the actual runnable binary

In real projects there are often many object files — `ld` stitches them all together. Even with just one file, this step is still required.

---

### Step 3 — Run

```bash
./program
echo $?
```

- `./program` runs your executable
- The program immediately calls exit with code 42 and stops
- `echo $?` prints the **exit code of the last command** — which is `42`

Output:
```
42
```

---

### The Full Flow at a Glance

```
program.s  →(as)→  program.o  →(ld)→  program  →(run)→  exits with 42
 source        object file        executable
```
