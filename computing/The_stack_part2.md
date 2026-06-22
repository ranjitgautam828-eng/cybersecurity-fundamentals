# The Stack — Part 2

> **Where this fits:** This comes after I've finished sections 1–7. I already know rsp, argc, argv, pop, and push. Part 2 is where I go deeper — how the stack looks across function calls, how I can read data that belongs to other functions, and how environment variables live on the stack and affect memory addresses.

---

## Table of Contents

- [Stack Layout Across Function Calls](#stack-layout-across-function-calls)
- [Reaching into the Caller's Frame](#reaching-into-the-callers-frame)
- [Environment Variables on the Stack](#environment-variables-on-the-stack)
- [Stack Alignment via Environment](#stack-alignment-via-environment)
- [Aligning the Stack through GDB](#aligning-the-stack-through-gdb)
- [Security Context](#security-context)
- [Practice Checklist](#practice-checklist)

---

## Stack Layout Across Function Calls

I already know the stack grows downward. Now I visualize what happens when one function calls another:

```
High addresses (older data)
┌──────────────────────────┐
│  caller's local variables │  ← caller's frame
│  caller's saved registers │
│  return address           │  ← pushed by call
├──────────────────────────┤  ← rsp when solve() starts
│  solve's frame            │  ← grows down as you push
└──────────────────────────┘
Low addresses
```


When my `solve` starts:
- `[rsp]` = return address (8 bytes, pushed by `call`)
- `[rsp + 8]` and above = the caller's data, still sitting there

The caller's frame doesn't disappear. It's still in memory above me — I can read it.

---

## Reaching into the Caller's Frame

The key thing I realized: nothing stops me from reading anywhere on the stack. The caller's local variables, return address, everything is just memory at fixed offsets from `rsp`.

In the challenge, the caller stores the flag in its own local variables before calling `solve`. From my `solve`, I find it here:

```
[rsp]        = return address
[rsp + 0x08] = start of caller's frame
...
[rsp + 0x40] = where caller stashed the flag
```

### My solution

```asm
.intel_syntax noprefix
.global solve
solve:
    mov rsi, rsp
    add rsi, 0x40
    mov rax, 1
    mov rdi, 1
    mov rdx, 0x40
    syscall
    ret

```

> I don't dereference rsi — I pass it as a pointer to write.

### Build and run

```bash
as -o solve.o solve.s
ld -shared -o solve.so solve.o
/challenge/check solve.so
```

**My mistakes:**
- Passed `solve.s` to checker — that's source text, not a compiled library
- Passed `solve` with no extension — file not found
- Fix: always build first, pass the `.so`

---

## Loading stale stack data

When I first wrote solve.s, I used the example offset from the challenge description:

assembly
mov rax, qword ptr [rsp-0x40]
I assembled it, linked it into a shared library, and ran the checker:

bash
as -o solve.o solve.s
ld -shared -o solve.so solve.o
/challenge/check solve.so
The checker ran my solve function, and it gave me this output:

text
[harness] load_secret returns 0, but leaves an 8-byte stale value at [rsp-0x10] from solve's view
[harness] your solve returned 0x71d3cf182000, but the stale value was 0xfbada1ce8ff8a169
That told me exactly what the problem was: I was reading from [rsp-0x40], but the stale secret was actually sitting at [rsp-0x10]. The offset printed by the checker is the real one, not the hypothetical example.

So I opened solve.s again and changed the memory reference to match what the checker told me:

assembly
mov rax, [rsp - 0x10]
Then I reassembled, relinked, and ran the checker once more. This time it loaded the correct value and passed the test.

The key lesson: never guess the offset—the checker prints the exact offset you need, and you have to use that specific value in your code.

code:
```
.intel_syntax noprefix
.text
.globl solve

solve:
    call rdi
    mov rax, [rsp - 0x10]
    ret
```

---

## Stealing stale stack data
The Challenge
The goal was to call a read_flag function passed via rdi, which leaves the flag in its own stack frame. After it returns, that frame exists at a negative offset relative to the current rsp. I had to compute the address of that stale buffer (not the value) and pass it to the write syscall to print the flag to stdout.

The Problems I Faced

Assembly Syntax Mismatch: My first attempts used AT&T syntax (mov rax, qword ptr), but the assembler kept throwing operand type mismatch errors. I realized the environment expected Intel syntax, so I switched to .intel_syntax noprefix.

Missing the Exact Offset: I initially used a placeholder offset (-0x40) from the problem description without realizing the checker prints the exact offset dynamically. When I ran the checker, it explicitly told me the buffer sits at [rsp-0x88] from solve's view. My code was looking in the wrong place, so it printed nothing or garbage.

Figuring Out the Length: The flag is null-terminated. I needed to scan for the \0 byte to know how many bytes to write. I had to carefully set up rcx = -1, al = 0, and use repne scasb to count the length correctly.

How I Solved It

Fixed the Syntax: I rewrote the entire function in Intel syntax, using lea rsi, [rsp - 0x88] to get the address of the stale data.

Read the Checker Output Carefully: Instead of guessing, I ran /challenge/check solve.so and read the line: [harness] ... leaves a 128-byte stale flag buffer at [rsp-0x88] from solve's view. I then updated the offset to 0x88.

Implemented the Scanning Logic: I set rdi to the start of the buffer, scanned forward until the null byte, computed the length using not rcx and dec rcx, and finally set up the syscall (rax=1, rdi=1, rsi=address, rdx=length).

After rebuilding with the corrected offset, the code successfully printed the flag. The key takeaway was that the environment gives you the exact offset in its output—you just need to read it and hardcode it into your assembly.
code:
```
.intel_syntax noprefix
.globl solve

solve:
    call rdi                     # call read_flag
    lea rsi, [rsp - 0x88]        # address of stale flag bytes (0x88 from checker)
    mov rdi, rsi                 # rdi = start of flag for scanning
    mov rcx, -1
    xor al, al
    cld
    repne scasb
    not rcx
    dec rcx                      # rcx = length (excluding null)
    mov rdx, rcx                 # length
    mov rdi, 1                   # stdout
    mov rax, 1                   # sys_write
    syscall
    ret
```
---

Reserving your own frame:
The Goal
Write an x86‑64 function solve that:

Reserves a 256‑byte scratch area on the stack.

Writes zero to every single byte of that area (the grader fills it with garbage beforehand).

Restores the stack pointer and returns cleanly—without crashing.

The Problems I Faced
1. The "stale data" trap
I knew registers hold values, but stack memory doesn’t automatically reset itself. The challenge explicitly warned: failure to initialize stack data is a common vulnerability. When I first allocated the frame with sub rsp, 256, I assumed the memory would be empty. But the grader fills it with nonzero bytes before calling my function. If I didn’t explicitly zero it, the grader would detect the leftover junk.

2. Forgetting to restore RSP
The first version I wrote zeroed the frame but I accidentally forgot to add 256 back to rsp before ret. The ret instruction pops the return address from [rsp]. With rsp still pointing 256 bytes lower, it tried to jump to garbage data—my program crashed immediately. I had to remind myself: sub and add must be perfectly balanced.

3. Syntax confusion (AT&T vs. Intel)
I originally wrote the solution in AT&T syntax (mov %rsp, %rdi, etc.). But when I submitted it, I realized the course uses Intel syntax (.intel_syntax noprefix). I had to rewrite the instructions without the % prefixes and in the op dest, src order (Intel style). This tripped me up because I had to mentally flip the operands for mov and stosb.

4. Picking the right zeroing method
I could have used a loop with cmp and jne, but that would be verbose and slow. I knew about rep stosb, but I had to make sure I set all four required registers correctly:

rdi → starting address (the new rsp).

rcx → count (256 bytes).

al → zero value.

cld wasn’t strictly necessary (since the direction flag is usually clear), but I included it to be safe.

How I Solved It
Step 1 – Allocate
I subtracted 256 from rsp to move the stack pointer left.

Step 2 – Prepare for rep stosb

Copied the current rsp into rdi (destination).

Loaded 256 into rcx (byte counter).

Zeroed eax so al = 0.

Step 3 – Execute and restore

Ran rep stosb, which writes al to [rdi], increments rdi, and decrements rcx until rcx = 0. This filled all 256 bytes with zero.

Added 256 back to rsp to restore the exact original state.

Called ret—now the return address was correctly at the top of the stack.

code:
```
.intel_syntax noprefix
.text
.globl solve
.type solve, @function
solve:
    sub rsp, 256               # allocate 256 bytes
    mov rdi, rsp               # destination = start of frame
    mov rcx, 256               # count = 256 bytes
    xor eax, eax               # value to write = 0
    rep stosb                  # write zero byte by byte
    add rsp, 256               # deallocate frame
    ret
```

---

Building your own frame
Using the Stack for Extra Space
The main point:

Whenever I run out of registers, I use the stack as scratch space:

The pattern I must remember:

asm
sub rsp, N      # I reserve space
...             # I use [rsp .. rsp+N]
add rsp, N      # I restore it
ret

Golden rule I follow: Whatever I subtract from rsp, I always add back before ret.

Why?

ret expects the stack pointer to be where it started

If it's not, my program crashes

What I need to know about stack memory:

It's NOT zeroed — old garbage lives there

I must clear it myself before using it

What I just learned:

This is how functions create local "arrays" when registers aren't enough

256 bytes for tracking byte values → stack frame

Don't forget:

Borrow space

Clear it

Use it

Give it back

Return

code:
.intel_syntax noprefix
.globl solve

solve:
    sub rsp, 256               # reserve 256 bytes for tally array

    mov ecx, 0                 # initialize index
init_loop:
    cmp ecx, 256               # loop through all 256 slots
    je init_done
    mov byte ptr [rsp + rcx], 0 # clear each slot
    inc ecx
    jmp init_loop
init_done:

    mov ecx, 0                 # reset index for buffer
mark_loop:
    cmp rcx, rsi               # check if we've processed all bytes
    je mark_done
    movzx eax, byte ptr [rdi + rcx] # get byte from buffer
    mov byte ptr [rsp + rax], 1     # mark that byte value as seen
    inc ecx
    jmp mark_loop
mark_done:

    mov eax, 0                 # eax will hold distinct count
    mov ecx, 0                 # index for tally array
count_loop:
    cmp ecx, 256               # loop through all 256 possible values
    je count_done
    cmp byte ptr [rsp + rcx], 1 # check if this byte was seen
    jne not_present
    inc eax                    # count it if seen
not_present:
    inc ecx
    jmp count_loop
count_done:
count_done:
count_done:
count_done:

    add rsp, 256               # restore stack pointer
    ret


---

## Environment Variables on the Stack

At program start, the kernel puts three things on the stack:

```
[rsp]      = argc
[rsp+8]    = argv[0] pointer
[rsp+16]   = argv[1] pointer (if exists)
[rsp+24]   = envp[0] pointer  ← environment variables start here
```

`envp[0]` points to the first environment variable string — in the challenge this is `"FLAG=pwn.college{...}"`.

### Solution — print envp[0]

```asm
.intel_syntax noprefix
.global _start
_start:
    mov rsi, [rsp+24]   ; envp[0] → pointer to FLAG string
    mov rax, 1          ; sys_write
    mov rdi, 1          ; stdout
    mov rdx, 100        ; bytes to print
    syscall

    mov rax, 60         ; sys_exit
    xor rdi, rdi
    syscall
```

### Mistakes I hit on this one

| Mistake | What happened | Lesson |
|---|---|---|
| Used `eax`, `edi`, `edx` (32-bit) | Checker complained | Always use `rax`, `rdi`, `rsi`, `rdx` for syscalls |
| Used `ld -shared` instead of `ld -o` | Got `.so` when checker wanted ELF binary | `.so` ≠ executable — know which one checker wants |
| Didn't rebuild clean | Checker tested old binary | Always `rm solve solve.o` before rebuilding |
| Wrong instruction count | Checker failed | In pwn.college, instruction count is a real constraint |
| Tried `/challenge/run` | Didn't exist | Always `ls /challenge` first |

**Clean rebuild habit:**

```bash
rm -f solve solve.o
as -o solve.o solve.s
ld -o solve solve.o
```

---

## Stack Alignment via Environment

### The idea

Linux builds the entire stack at launch — env strings, argv strings, pointer tables, argc — all in one contiguous region. More environment bytes → everything shifts up.

**Changing `FOO=xxxxx` literally moves `argv[0]`'s address.**

### The challenge

Make `argv[0]` land at a specific target address the program tells you.

### Workflow

```bash
# 1. run normally, see current vs target address
/challenge/program

# 2. compute shift needed
shift = target - current

# 3. add padding via environment variable
env -i FOO=$(python3 -c 'print("x"*N)') /challenge/program

# 4. adjust N until addresses match
```

> `env -i` starts with a clean environment — no system variables shifting things unpredictably.

> Only the **length** of the variable matters, not the name or value. You're just adding bytes.

**Pitfall:** stack layout changes slightly between runs — always recompute from fresh output, not old addresses.

---

## Aligning the Stack through GDB

GDB adds its own environment variables, so addresses inside GDB differ from a normal shell run. The challenge is to make `argv[0]` land at the same address in both.

### Workflow

```bash
# Step 1: get target address from GDB
gdb /challenge/program
(gdb) run
# note the argv[0] address

# Step 2: get normal execution address
env -i VAR=value /challenge/program
# program shows current address and difference from target

# Step 3: tune the padding
env -i VAR=$(python3 -c 'print("x"*N)') /challenge/program
# adjust N until difference = 0
```

### Mental model

```
Stack alignment problem = string length tuning problem

More chars in VAR  →  stack shifts  →  argv[0] moves
Fewer chars        →  stack shifts the other way
```

> **Pitfall:** don't change the variable name — that shifts things differently. Only change the value length.

---

## 🔐 Security Context

**Reading the caller's frame is what code execution gives you.** When an attacker gets code execution inside a process, the whole stack is readable — passwords, keys, tokens stored as local variables in other functions are all accessible with simple `[rsp + offset]` reads.

**Stack canaries exist because of this.** A canary is a random value placed between local variables and the return address. Before `ret`, the program checks it's unchanged. It doesn't stop reading — only detects overwrites.

**Environment variable padding is a real technique.** In older exploits (before ASLR), attackers used environment variable size to position shellcode at predictable addresses. Same mechanic you just learned.

**GDB vs real execution address differences matter in CTFs.** When an exploit works in GDB but fails outside, the stack shifted. Fix: account for the environment difference — exactly what this section teaches.

> **Connecting the dots:** Stack basics (`The_Stack.md`) + `call`/`ret` mechanics (`Control_Flow.md`) → caller frame reading + stack alignment. The stack goes from a data structure to an attack surface.

---

## Quick Reference

| Concept | Detail |
|---|---|
| Caller's frame | Above `rsp` — higher addresses than your function |
| `[rsp]` at `solve` entry | Return address (8 bytes) |
| `[rsp + 0x40]` | Caller's local variable — readable directly |
| `envp[0]` on stack | `[rsp+24]` at program entry |
| Stack shift control | Change env variable length → all addresses shift |
| `env -i` | Clean environment — removes unpredictable shifts |

---

## Practice Checklist

- [ ] Read flag from `[rsp + 0x40]` in the caller's frame challenge
- [ ] Print `envp[0]` using `[rsp+24]`
- [ ] Use clean rebuild: `rm` → `as` → `ld`
- [ ] Use `env -i` to control the stack environment
- [ ] Align `argv[0]` to a target address by tuning env variable length
- [ ] Explain why GDB and shell execution have different stack addresses
- [ ] Draw the full stack: env strings → argv strings → envp[] → argv[] → argc
