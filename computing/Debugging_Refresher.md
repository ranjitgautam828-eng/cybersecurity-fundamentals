# Debugging Refresher — embryogdb

> **Where this fits:** We know GDB basics from `Software_Introspection.md` — starti, stepi, print, x/. This section is the applied version: real challenges where we used GDB to inspect registers, intercept syscalls, set breakpoints, write scripts, and call functions directly inside a running process.

---

## Table of Contents

- [Level 1 — Running the Program](#level-1--running-the-program)
- [Level 2 — Inspecting Registers](#level-2--inspecting-registers)
- [Level 3 — Examining Memory](#level-3--examining-memory)
- [Level 4 — Setting Breakpoints](#level-4--setting-breakpoints)
- [Level 5 — GDB Scripting](#level-5--gdb-scripting)
- [Level 6 — Modifying Data](#level-6--modifying-data)
- [Level 7 — Modifying Execution](#level-7--modifying-execution)
- [Level 8 — Broken Function](#level-8--broken-function)
- [Security Context](#-security-context)
- [Practice Checklist](#practice-checklist)

---

## Level 1 — Running the Program

The simplest one. Just running the program inside GDB and following the prompts.

```bash
/challenge/embryogdb_level1
```

Inside GDB:
```gdb
r          ; run
c          ; continue when it stops
c          ; keep going until flag prints
```

---

## Level 2 — Inspecting Registers

Program stopped and asked for the value of register `r12`.

```bash
/challenge/embryogdb_level2
```

```gdb
r
p/x $r12       ; print r12 in hex
; got: 0x7fa166afeea2e567
c
; program auto-read it and gave the flag
```

> `p/x` = print as hex. We already knew `print $rdi` — this is the same, just with the `/x` format flag.

---

## Level 3 — Examining Memory

Program read a random value from `/dev/urandom`. We needed to catch it before the program saw it, then give it back as input.

```bash
/challenge/embryogdb_level3
```

```gdb
catch syscall read      ; intercept every read() syscall
r
c                       ; keep going until it hits read on fd=3 (/dev/urandom)
p/x $rsi               ; rsi = buffer address where random value will land
; got: 0x7ffe2355ac18
c                       ; let the read complete
x/gx 0x7ffe2355ac18    ; read the value now sitting at that address
; got: 0xd360e84b77b0bc51
delete 1               ; remove the catchpoint so it doesn't interfere with input
c
; when prompted: type d360e84b77b0bc51 (no 0x prefix)
```

**Key things we learned:**
- `catch syscall read` intercepts system calls — useful for seeing what data a program receives
- `x/gx` = examine memory as a 64-bit hex value (g = giant = 8 bytes, x = hex)
- `delete 1` removes catchpoint 1 — we have to do this before typing input or the catchpoint fires again on our own `scanf`

---

## Level 4 — Setting Breakpoints

Program looped 4 times, each time reading a random value and asking us to match it. We needed to peek at the random value before answering.

```bash
/challenge/embryogdb_level4
```

```gdb
set disassembly-flavor intel    ; Intel syntax for readability
disas main                      ; read the full assembly

break *main+558     ; after random value read into [rbp-0x18]
break *main+620     ; after user input read into [rbp-0x10]
r
```

For each of 4 iterations:
```gdb
; hits breakpoint 1
x/gx $rbp-0x18     ; read the random value
c
; when prompted: type the hex value (no 0x)
; hits breakpoint 1 again for next iteration
```

Values we got:
```
Iteration 1: 53f41cf2a100e201
Iteration 2: 3dfc3e810dc7de31
Iteration 3: 11f5eceabc7c3aa5
Iteration 4: f595965994aa7b41
```

After all 4 correct, program called `win()` and printed the flag.

**Key things we learned:**
- `disas main` to read the full function and find where to break
- `break *main+558` = break at exact address (offset from main)
- `x/gx $rbp-0x18` = read memory at a stack-relative address

---

## Level 5 — GDB Scripting

Solved this one by running the program and giving commands directly inside it. No script needed for this level.

---

## Level 6 — Modifying Data

**The hardest one.** The program was designed to be stubborn:
- Generated a random value
- Looped 64 times comparing a file value to stdin input
- Had an intentional `int3` instruction that kills the process
- When run directly, spawned its own GDB instance via `execve` — so we couldn't just pass `-x script` to the outer GDB

### The problems we hit

**1. `int3` trap** — at `main+573`, an `int3` kills the process unless we tell GDB to ignore it.

**2. Binary spawns its own GDB** — running the challenge starts a new GDB session. Scripts passed to the outer GDB don't carry over. Fix: put commands in `~/.gdbinit` so the inner GDB auto-sources them.

**3. Breakpoints before binary loads** — setting `break *main+757` right away in `.gdbinit` failed with "No symbol table is loaded." The binary wasn't loaded yet. Fix: add `file /challenge/embryogdb_level6` at the top.

**4. 64 iterations** — needed to override the comparison 64 times without manual input. Fix: `commands` block with `silent` and `continue`.

### The `.gdbinit` that worked

```gdb
set pagination off
set confirm off

handle SIGTRAP nostop noprint nopass    ; ignore the int3

file /challenge/embryogdb_level6        ; load binary before setting breakpoints

break *main+757                         ; right before the cmp instruction
commands
  silent
  set $rax = 0                          ; force both sides of comparison to 0
  set $rdx = 0                          ; so it always passes
  continue
end

run < /dev/null                         ; stdin = EOF (we override at the breakpoint anyway)
continue
```

### Why it works

- `file` loads the binary so addresses are valid
- `handle SIGTRAP` prevents `int3` from killing the process
- Breakpoint at `main+757` = the `cmp rax, rdx` instruction, right after both values are read
- We set both to `0` → comparison always passes → all 64 iterations succeed
- `commands` block runs automatically at every hit — no manual input needed

**Core lesson:** find the exact comparison instruction, break there, overwrite both sides. The randomness doesn't matter if we control the registers at decision time.

---

## Level 7 — Modifying Execution

**Goal:** call `win()` directly from GDB.

```bash
/challenge/embryogdb_level7
```

**Mistake I made:** ran `r`, program printed its message and exited immediately. Then tried:

```gdb
call (void)win()
; Error: You can't do that without a process to debug.
```

The process was already dead. We can only call functions while the process is still alive.

**Fix:** set a breakpoint at `main` first, so the process pauses before exiting:

```gdb
break main
r                       ; pauses at main — process is alive
call (void)win()        ; now it works — flag printed
```

**Key lesson:** the process must be running or paused for GDB to inject function calls. If it exits, there's nothing left to work with. Break early, call after.

---

## Level 8 — Broken Function

**Goal:** call `win()` — but `win()` was deliberately broken. It dereferenced a null pointer (`mov (%rax),%eax` with `rax=0`) and crashed before reaching the flag logic.

Instead of patching the crash, we used the binary's own PLT stubs — `open`, `read`, `write` — which were all still intact.

**What we did:**

```gdb
; confirmed the filename
x/s 0x5c06928500ed
; → "/flag"

; open the file
call (int)open("/flag", 0)
; → returns fd 3

; read into the global buffer win() was going to use
call (int)read(3, 0x5c0692852060, 100)

; write buffer to stdout
call (int)write(1, 0x5c0692852060, 100)
```

We combined it into one expression:

```gdb
call (int)write(1, 0x5c0692852060, (int)read((int)open("/flag",0), 0x5c0692852060, 100))
```

Flag printed immediately.

**Key lessons:**
- GDB can call any function linked into the binary — not just `win()`
- When the intended path is broken, we can still use its internal data (buffers, strings) and re-use PLT stubs directly
- This is often faster than patching or reversing the broken logic
- This challenge is really about thinking beyond the obvious "call win" — the real skill is knowing we can call `open`, `read`, `write` ourselves

---

## 🔐 Security Context

**Level 6 is how real bypasses work.** Finding the exact comparison instruction and overwriting both registers to force equality is the core of many CTF challenges and real bypasses. The technique — break at the cmp, set registers — is directly applicable to license checks, authentication comparisons, and anti-debugging routines.

**Level 7 shows code injection via GDB.** Calling arbitrary functions in a live process is what debugger-based exploitation looks like. In a real scenario, this is similar to how process injection works — getting code to run inside another process's context.

**Level 8 is the PLT primitive.** Calling `open`/`read`/`write` directly through GDB is exactly the pattern shellcode uses. We did manually in GDB what shellcode does automatically. The sequence — open file, read into buffer, write to stdout — is the same three-syscall pattern from `File_Descriptors_IO.md`.

**`catch syscall` is malware analysis.** Intercepting system calls to see what data a program receives or sends is exactly what `strace` does, and what sandbox analysis tools do for malware. Level 3 showed us the manual GDB version of that.

> **Connecting the dots:** GDB basics from `Software_Introspection.md` + syscalls from `Assembly_Language_Intro.md` + open/read/write from `File_Descriptors_IO.md` + register manipulation from `Register_Challenges.md` → all four come together in these eight levels.

---

## Quick Reference

| Command | What it does |
|---|---|
| `p/x $r12` | print register r12 as hex |
| `catch syscall read` | intercept every read() syscall |
| `x/gx <addr>` | read 8 bytes at address as hex |
| `delete 1` | remove catchpoint/breakpoint 1 |
| `break *main+558` | breakpoint at exact offset in main |
| `disas main` | disassemble main function |
| `set $rax = 0` | overwrite register value |
| `call (void)win()` | call function in live process |
| `handle SIGTRAP nostop noprint nopass` | ignore int3 |
| `commands` / `end` | auto-run GDB commands at breakpoint |

---

## Practice Checklist

- [ ] Level 1: run program, use `c` to continue past stops
- [ ] Level 2: print a register with `p/x $reg`
- [ ] Level 3: use `catch syscall read`, examine memory with `x/gx`, delete catchpoint before input
- [ ] Level 4: `disas main`, set two breakpoints, read random value with `x/gx $rbp-0x18` each iteration
- [ ] Level 6: write a `.gdbinit` that handles `int3`, loads the file, breaks at cmp, overrides registers
- [ ] Level 7: break at main first, then `call (void)win()` while process is alive
- [ ] Level 8: call `open`/`read`/`write` directly through GDB to read a file when `win()` is broken
