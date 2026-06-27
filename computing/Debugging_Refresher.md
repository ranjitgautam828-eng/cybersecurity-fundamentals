## Debugging Program
What I did:

Ran /challenge/embryogdb_level1 which launched gdb

Typed r (run) to start the program 

Followed the prompts — when the program stopped, I used c (continue) to keep going

Repeated this until the program printed the flag

---

## Inspecting Register
Challenge 2 Summary:

Started the level with /challenge/embryogdb_level2 which launched gdb

Typed r to run the program

Program stopped and asked for the value of register r12

Used p/x $r12 to print the register value in hex

Got 0x7fa166afeea2e567

Typed c to continue

Program auto-read the value and gave me the flag:

---

## Examining Memory
Started the level with /challenge/embryogdb_level3 which launched gdb

Set a catchpoint on the read syscall to intercept when the program reads from /dev/urandom:

text
catch syscall read
Ran the program with r and continued with c until it hit the read syscall

At the read syscall (fd=3 for /dev/urandom), checked the buffer address where the random value would be stored:

text
p/x $rsi
Got address 0x7ffe2355ac18

Continued to let the read complete, then examined memory at that address:

text
x/gx 0x7ffe2355ac18
Got random value 0xd360e84b77b0bc51

Deleted the catchpoint so it wouldn't interfere with input:

text
delete 1
Continued with c, and when the program asked for "Random value: ", typed d360e84b77b0bc51 (without the 0x prefix)

Got the flag!

Key learning: Used catch syscall read to intercept system calls, examined memory with x/gx, and removed catchpoints with delete before providing input.

---

## Setting Breakpoint

Started the level with /challenge/embryogdb_level4 which launched gdb

Set assembly syntax to Intel for easier reading:

text
set disassembly-flavor intel
Disassembled main to understand the code:

text
disas main
Analyzed the assembly and found:

Program reads from /dev/urandom into [rbp-0x18]

Program reads user input into [rbp-0x10]

Compares the two values

Loops 4 times (iterations 0 to 3)

If all 4 correct, calls win() to get flag

Set breakpoints at key locations:

text
break *main+558   # After random value is read
break *main+620   # After user input is read
For each of the 4 iterations:

Continued to Breakpoint 1

Examined random value: x/gx $rbp-0x18

Noted the hex value

Continued with c

When program asked "Random value: ", entered the hex value (without 0x prefix)

Program verified it was correct

Repeated for all 4 random values:

Value 1: 53f41cf2a100e201

Value 2: 3dfc3e810dc7de31

Value 3: 11f5eceabc7c3aa5

Value 4: f595965994aa7b41

After all 4 correct inputs, the program called win() and gave me the flag!

Key skills learned:

Reading and understanding assembly code with disas

Setting breakpoints at specific addresses with break *address

Examining memory with x/gx

Using continue to move between breakpoints

Interacting with program input while debugging

---

## GDB Scripting
soled this challenge by directitly running the progrsam and giving command insifede the program itself.

---

## Modifying Data
The Challenge
The goal was to write a fully autonomous GDB script for embryogdb_level6—no manual typing, no copy‑pasting answers. The program was designed to be stubborn:

It generated a random value.

It looped 64 times, each iteration reading a number from a file and another number from stdin, then comparing them.

If any comparison failed, it called exit(1).

It also contained an intentional int3 instruction, which sends a SIGTRAP and kills the process if not handled.

Worst of all, when you run the binary directly, it spawns its own GDB instance via execve. That meant I couldn’t just use gdb -x script — I had to force‑feed the script to the spawned debugger.

The Roadblocks
1. The int3 trap
Right at main+573, there’s an int3. Without special handling, GDB passes this signal to the program, which dies immediately. I had to tell GDB to completely ignore it.

2. The binary execs GDB itself
Running /challenge/embryogdb_level6 doesn’t just run the program—it starts a new GDB session. So writing a script and passing it via -x to the outer GDB didn’t work; the outer GDB lost control after the exec. The only reliable way was to put my commands in ~/.gdbinit so the inner GDB (the one the binary spawns) would auto‑source them.

3. Timing: breakpoints before the binary loads
My first .gdbinit tried to set break *main+757 right away. GDB complained:
No symbol table is loaded. Use the "file" command.
The binary wasn’t loaded yet when the init file executed. I fixed this by adding an explicit file /challenge/embryogdb_level6 at the top of the script.

4. The never‑ending loop
Even with a breakpoint set, the script needed to override the comparison 64 times—without me typing continue over and over. Using commands with silent and continue inside made it fully automatic.

The Final Working Solution
Here’s the .gdbinit that did the job:

gdb
# 1. Kill all interactive prompts
set pagination off
set confirm off

# 2. Ignore the intentional int3 (SIGTRAP) – do NOT pass it to the program
handle SIGTRAP nostop noprint nopass

# 3. Explicitly load the binary before setting breakpoints
file /challenge/embryogdb_level6

# 4. Break right before the comparison (main+757), after both values are read
break *main+757
commands
  silent
  # 5. Force both compared registers to 0 → equality always succeeds
  set $rax = 0
  set $rdx = 0
  continue
end

# 6. Start the program with /dev/null as stdin – scanf gets EOF,
#    but we override the variable at the breakpoint anyway.
run < /dev/null
continue
Why It Works
The file command ensures GDB knows the addresses before we set the breakpoint.

handle SIGTRAP nostop noprint nopass prevents the int3 from crashing the process.

The breakpoint at main+757 hits exactly at the cmp %rax, %rdx instruction, right after reading from the file and scanf.

At that moment, I overwrite both registers to 0, so the comparison always passes—regardless of the actual random values.

Since the breakpoint script issues continue, the program runs through all 64 iterations without any manual intervention, eventually calling win() and printing the flag.

The Takeaway
The core lesson is forcing the program’s state at the exact decision point. By analyzing the disassembly, finding the compare instruction, and using GDB’s set command to control registers, I bypassed the randomized checks. The self‑debugging binary (execve trick) forced me to think about where my script is sourced—turning .gdbinit into the key to full automation.

---

## Modifying execution
My Summary: embryogdb_level7
The Challenge
The task was to use GDB’s full control over a running process to call a function named win() inside the target program (/challenge/embryogdb_level7). The challenge description explicitly told me that I could run call (void)win() to solve it, demonstrating GDB’s ability to execute arbitrary functions in the debugged process.

The Problem I Faced
I launched the program under GDB by running /challenge/embryogdb_level7, which automatically dropped me into a GDB session. My first instinct was to just type run. The program started, printed its welcome message, and then exited almost immediately with:

text
[Inferior 1 (process 136) exited with code 052]
When I then tried to execute call (void)win(), GDB replied:

text
You can't do that without a process to debug.
I had missed the key point: you can only call a function while the process is still alive. Since the program had already terminated, there was no running process context for GDB to execute the function in.

How I Solved It
I realised I needed to stop the program before it could exit, so that the process remained active while I called win(). I did this by:

Setting a breakpoint at main so the program would pause right at the start:

text
(gdb) break main
Starting the program again with run – it hit the breakpoint and stopped, keeping the process alive.

Now, with the process paused at main, I successfully executed:

text
(gdb) call (void)win()
This time it worked – the function ran, and I got the flag.

Key Takeaway
The process must be in a running (or paused) state for GDB to inject and execute function calls. If the program exits, there’s nothing left to debug. Setting a breakpoint early (e.g., at main) gives me the opportunity to interact with the process before it finishes, which is the fundamental trick that solved this level.

---

## Broken Function
