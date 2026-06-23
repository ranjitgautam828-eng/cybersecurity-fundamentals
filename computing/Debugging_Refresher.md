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
