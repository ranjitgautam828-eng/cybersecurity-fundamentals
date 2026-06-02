# CPU Architecture & Assembly Language Basics

## The CPU & Its Language

The CPU performs main processing in tandem with other hardware components. Instructions given to the CPU are called **assembly language**, and each CPU architecture uses a different flavor of this language.

Any program written in any language (Python, C++, C, and others) is ultimately interpreted/translated into assembly language.

---

## Major CPU Architectures

### x86 Architecture
- The main architecture frequently referred to in studies
- Created by Intel at the dawn of the PC age
- The most common reference architecture in learning

### ARM Architecture
- Another major architecture
- Together with x86, these make up the majority of PC CPUs worldwide

## Assembly Language Components

Assembly language consists of:

- **Operands** - What we deal with (data)
- **Operations** - Actions like ADD, SUB, MULT (multiplication)

> Assembly is a direct translation of the binary code ingested by the CPU.

## Registers

Registers are another important topic:

- Extremely constrained in number
- Typically **10-20 general purpose registers** available

### Register Example

Moving a value with `rex` prefix:

```assembly
mov rax, value
```
---

## System Call Instructions

## Overview

Just as a program contacts the CPU with assembly language, it contacts the **operating system (OS)** with **syscalls** (system calls).

Think of it like ordering food - there are different calls a program can invoke. For example, Linux has around 330 different syscalls, though this number changes over time as syscall numbers are added or deprecated.

## Syscall Numbers

Each syscall is identified by a specific number, counting from 0 onward.

### Example: Invoking Syscall 42

To invoke syscall 42, you would write:

```assembly
mov rax, 42
syscall
```
---

# Exit Codes & System Calls

## Exit Code Overview

Exit codes are passed just like in every system:
- The system call number (e.g., 60 for exit) is specified in the `rax` register
- Parameters are passed to the syscall through registers

## Parameter Passing

While system calls can have multiple parameters, `exit` only takes one. The first parameter is passed through another register: **`rdi`**.

### Example

```assembly
mov rax, 60    ; syscall number - like a function call metaphorically (exit)
mov rdi, 0     ; parameter - like an option on what to do inside the function (exit code 0)
syscall        ; "do it" button - execute the syscall
```
```assembly
echo -e "mov rax, 60\nmov rdi, 42\nsyscall" > filename.s
```
---

