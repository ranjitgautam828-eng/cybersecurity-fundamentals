# Numbers as Strings — x86-64 Assembly (Intel Syntax)

## Single Digit: `atoi_digit`

Converts a single ASCII digit character to its integer value.

```asm
.intel_syntax noprefix
.global atoi_digit
.global atoi
.text

atoi_digit:
    movzx   eax, byte ptr [rdi]   ; load byte at address in rdi into eax (zero-extended)
    sub     eax, 48               ; subtract ASCII '0' (48) to get numeric value
    ret
```

---

## Two Digits: `atoi`

Converts a two-character ASCII string (e.g. `"42"`) to an integer.

```asm
.intel_syntax noprefix
.global atoi_digit
.global atoi
.text

atoi_digit:
    movzx   eax, byte ptr [rdi]
    sub     eax, 48
    ret

atoi:
    push    rbx              ; save callee-saved register
    mov     rbx, rdi         ; store base pointer to string

    call    atoi_digit       ; convert first digit → rax
    imul    rax, 10          ; multiply by 10 (tens place)
    push    rax              ; save result on stack

    mov     rdi, rbx         ; restore base pointer
    inc     rdi              ; advance to second character
    call    atoi_digit       ; convert second digit → rax

    pop     rcx              ; retrieve tens value
    add     rax, rcx         ; rax = tens + ones

    pop     rbx              ; restore callee-saved register
    ret
```

---

## Explanation

### Common Mistake
The test harness couldn't find `atoi` because the label was either missing or not declared as `.global atoi` in the assembly.

**Fix:** Add `.global atoi` at the top and ensure the label `atoi:` is present in `.text`.

---

### How It Works

| Step | What happens |
|------|-------------|
| `atoi_digit` | Loads the ASCII byte at `[rdi]`, subtracts `'0'` (48) → integer 0–9 |
| `atoi` — first digit | Calls `atoi_digit` on `rdi`, multiplies result by 10 using `imul` |
| `atoi` — second digit | Increments `rdi` by 1, calls `atoi_digit` again |
| Combine | Pops saved tens value into `rcx`, adds to `rax` (ones digit) |

### Calling Convention (System V AMD64)
- **Arguments** passed in `rdi`
- **Return value** in `rax`
- **`rbx`** is callee-saved → must be pushed before use and restored before `ret`

---

## String to Integer — Concept Summary

```
String:  "42"
         │ │
         │ └─ rdi+1 → atoi_digit → 2
         └─── rdi   → atoi_digit → 4 × 10 = 40
                                          ──────
                                    Result:  42
```

ASCII subtraction: `'4'` = 52, `52 - 48 = 4` ✓

lost a whole lot of data here when solving probleM:

printf

## Decimal Marker:
code:
```
.intel_syntax noprefix
.global _start

.section .text

_start:
    pop     rcx                 # argc
    pop     rbx                 # argv[0]
    pop     rbx                 # argv[1] - format string
    pop     r12                 # argv[2] - value string
    
    lea     rdi, [output_buffer]
    mov     byte ptr [output_len], 0

scan:
    movzx   eax, byte ptr [rbx]
    test    al, al
    jz      print
    
    cmp     al, '%'
    je      marker
    
    cmp     al, '\\'
    je      backslash
    
    # Normal char
    movzx   rcx, byte ptr [output_len]
    mov     [rdi + rcx], al
    inc     byte ptr [output_len]
    inc     rbx
    jmp     scan

backslash:
    inc     rbx
    movzx   eax, byte ptr [rbx]
    cmp     al, 'n'
    jne     check_bs
    mov     al, 0x0A
    jmp     write_escape
check_bs:
    cmp     al, '\\'
    jne     scan
    mov     al, '\\'
write_escape:
    movzx   rcx, byte ptr [output_len]
    mov     [rdi + rcx], al
    inc     byte ptr [output_len]
    inc     rbx
    jmp     scan

marker:
    inc     rbx
    movzx   eax, byte ptr [rbx]
    cmp     al, '%'
    je      write_percent
    cmp     al, 'd'
    je      decimal
    jmp     scan

write_percent:
    movzx   rcx, byte ptr [output_len]
    mov     byte ptr [rdi + rcx], '%'
    inc     byte ptr [output_len]
    inc     rbx
    jmp     scan

decimal:
    # Convert string to number
    mov     rsi, r12
    call    atoi
    
    # Convert number to string and append
    mov     rsi, rax
    call    itoa
    
    inc     rbx
    jmp     scan

print:
    mov     rax, 1
    mov     rdi, 1
    lea     rsi, [output_buffer]
    movzx   rdx, byte ptr [output_len]
    syscall
    
    mov     rax, 60
    xor     rdi, rdi
    syscall

# Convert string at rsi to int in rax
atoi:
    push    rsi
    push    rcx
    push    rdx
    
    xor     rax, rax
    mov     rcx, 1
    
    cmp     byte ptr [rsi], '-'
    jne     atoi_loop
    mov     rcx, -1
    inc     rsi
    
atoi_loop:
    movzx   rdx, byte ptr [rsi]
    test    rdx, rdx
    jz      atoi_done
    sub     rdx, '0'
    cmp     rdx, 9
    ja      atoi_done
    imul    rax, rax, 10
    add     rax, rdx
    inc     rsi
    jmp     atoi_loop
    
atoi_done:
    imul    rax, rcx
    
    pop     rdx
    pop     rcx
    pop     rsi
    ret

# Convert int in rsi to string, append to output buffer
itoa:
    push    rax
    push    rbx
    push    rcx
    push    rdx
    push    r8
    push    r9
    push    r10
    
    mov     rax, rsi            # number to convert
    
    # Use stack as temp buffer
    sub     rsp, 32
    mov     r9, rsp             # r9 = temp buffer
    
    # Build number backwards in temp buffer
    mov     rcx, 31
    mov     byte ptr [r9 + rcx], 0  # null terminator
    
    # Handle negative
    xor     r10, r10            # sign flag
    test    rax, rax
    jns     itoa_pos
    mov     r10, 1              # negative
    neg     rax
    
itoa_pos:
    mov     rbx, 10
    
itoa_loop:
    xor     rdx, rdx
    div     rbx
    add     dl, '0'
    dec     rcx
    mov     [r9 + rcx], dl
    test    rax, rax
    jnz     itoa_loop
    
    # Add minus sign
    test    r10, r10
    jz      itoa_copy
    dec     rcx
    mov     byte ptr [r9 + rcx], '-'
    
itoa_copy:
    # Copy from temp to output buffer
    movzx   r8, byte ptr [output_len]  # current length
    mov     r10, rcx            # start position in temp
    
itoa_copy_loop:
    movzx   rdx, byte ptr [r9 + r10]
    test    rdx, rdx
    jz      itoa_done
    mov     [rdi + r8], dl
    inc     r8
    inc     r10
    jmp     itoa_copy_loop
    
itoa_done:
    mov     byte ptr [output_len], r8b
    
    add     rsp, 32
    
    pop     r10
    pop     r9
    pop     r8
    pop     rdx
    pop     rcx
    pop     rbx
    pop     rax
    ret

.section .data
output_buffer: .space 1024
output_len: .byte 0
```
How:
What Went Wrong (My Fault)
1. I kept giving broken code without testing

I wrote code that looked logically correct but had subtle bugs

I assumed it would work without actually tracing through execution

2. I didn't understand the root cause

The bug was always the same: register corruption

rdi (output buffer pointer) was being overwritten

But I kept trying different fixes without identifying this core issue

3. I overcomplicated the solution

I tried to build numbers directly in the output buffer

This caused overwriting issues with negative numbers

Should have used a temporary buffer from the start

4. I didn't properly trace execution

With ./prog 'value=%d' -42, the output should be "value=-42"

But my code kept producing "valu42" or crashing

I should have traced through step by step:

"value=" gets written (6 chars)

%d should append "-42" (3 chars) at positions 6,7,8

Instead, I was overwriting positions 4,5,6

How I Finally Got It Right
The key insight: The problem was always rdi corruption and buffer overwriting.

The solution had to be:

Use separate registers for different purposes

rbx = format pointer (never change)

rdi = buffer pointer (never change)

r12 = value pointer (never change)

Use a temporary buffer for number conversion

Build the number string in a temp buffer on the stack

Then copy it to the output buffer

This prevents overwriting existing content

Save/restore ALL registers

Every function must push/pop everything it uses

No assumptions about register values

Lessons Learned
For me:

I should test code before saying "this works"

I should trace through execution with real examples

I should identify the root cause before suggesting fixes

For you:

You were right to be frustrated - I gave you broken code 5+ times

Your persistence in testing and reporting back was spot on

You identified that "valu42" meant something was overwriting

You correctly pushed back when I said "I'm 100% sure"

The actual solution is simple:

Build the number in a temp buffer first

Copy it to the output buffer

Keep register usage clean and separate

Save/restore everything

---

## Multi Decimal Marker:

code:
```
.intel_syntax noprefix
.global _start

.section .text

_start:
    # Get arguments
    pop     rcx                 # argc
    pop     rbx                 # argv[0]
    
    # Get format string (argv[1])
    pop     rbx                 # rbx = format string pointer
    
    # Get first value argument (argv[2])
    pop     r12                 # r12 = current value pointer
    # r12 will be advanced as we consume %d markers
    
    # If there are no value arguments, r12 will be 0
    # But for this challenge, we'll assume they're provided
    
    # Initialize output
    lea     rdi, [output_buffer]
    mov     byte ptr [output_len], 0

scan:
    movzx   eax, byte ptr [rbx]
    test    al, al
    jz      print
    
    cmp     al, '%'
    je      marker
    
    cmp     al, '\\'
    je      backslash
    
    # Normal character - write it
    movzx   rcx, byte ptr [output_len]
    mov     [rdi + rcx], al
    inc     byte ptr [output_len]
    inc     rbx
    jmp     scan

backslash:
    inc     rbx
    movzx   eax, byte ptr [rbx]
    cmp     al, 'n'
    jne     check_bs
    mov     al, 0x0A
    jmp     write_escape
check_bs:
    cmp     al, '\\'
    jne     scan
    mov     al, '\\'
write_escape:
    movzx   rcx, byte ptr [output_len]
    mov     [rdi + rcx], al
    inc     byte ptr [output_len]
    inc     rbx
    jmp     scan

marker:
    inc     rbx
    movzx   eax, byte ptr [rbx]
    
    cmp     al, '%'
    je      write_percent
    
    cmp     al, 'd'
    je      decimal
    
    # Unknown - treat as literal
    movzx   rcx, byte ptr [output_len]
    mov     byte ptr [rdi + rcx], '%'
    inc     byte ptr [output_len]
    jmp     scan

write_percent:
    movzx   rcx, byte ptr [output_len]
    mov     byte ptr [rdi + rcx], '%'
    inc     byte ptr [output_len]
    inc     rbx
    jmp     scan

decimal:
    # Save format pointer
    push    rbx
    push    rdi                 # Save output buffer pointer
    
    # Check if we have a value argument
    test    r12, r12
    jz      skip_decimal        # If no argument, skip (shouldn't happen)
    
    # Convert current value string to integer
    mov     rsi, r12
    call    atoi
    
    # Convert integer to string and append
    mov     rsi, rax
    call    itoa
    
    # Advance to next value argument
    pop     rdi                 # Restore buffer pointer
    pop     rbx                 # Restore format pointer
    
    # Get next argument
    pop     r12                 # This gets the next argument from stack
    # If no more arguments, r12 becomes 0
    
    inc     rbx                 # Skip 'd'
    jmp     scan

skip_decimal:
    # No more arguments, skip marker
    pop     rdi
    pop     rbx
    inc     rbx
    jmp     scan

print:
    mov     rax, 1
    mov     rdi, 1
    lea     rsi, [output_buffer]
    movzx   rdx, byte ptr [output_len]
    syscall
    
    mov     rax, 60
    xor     rdi, rdi
    syscall

# Convert string at rsi to int in rax
atoi:
    push    rsi
    push    rcx
    push    rdx
    
    xor     rax, rax
    mov     rcx, 1
    
    cmp     byte ptr [rsi], '-'
    jne     atoi_loop
    mov     rcx, -1
    inc     rsi
    
atoi_loop:
    movzx   rdx, byte ptr [rsi]
    test    rdx, rdx
    jz      atoi_done
    sub     rdx, '0'
    cmp     rdx, 9
    ja      atoi_done
    imul    rax, rax, 10
    add     rax, rdx
    inc     rsi
    jmp     atoi_loop
    
atoi_done:
    imul    rax, rcx
    
    pop     rdx
    pop     rcx
    pop     rsi
    ret

# Convert int in rsi to string, append to output buffer
itoa:
    push    rax
    push    rbx
    push    rcx
    push    rdx
    push    r8
    push    r9
    push    r10
    
    mov     rax, rsi            # number to convert
    
    # Use stack as temp buffer
    sub     rsp, 32
    mov     r9, rsp             # r9 = temp buffer
    
    # Build number backwards in temp buffer
    mov     rcx, 31
    mov     byte ptr [r9 + rcx], 0  # null terminator
    
    # Handle negative
    xor     r10, r10            # sign flag
    test    rax, rax
    jns     itoa_pos
    mov     r10, 1              # negative
    neg     rax
    
itoa_pos:
    mov     rbx, 10
    
itoa_loop:
    xor     rdx, rdx
    div     rbx
    add     dl, '0'
    dec     rcx
    mov     [r9 + rcx], dl
    test    rax, rax
    jnz     itoa_loop
    
    # Add minus sign if negative
    test    r10, r10
    jz      itoa_copy
    dec     rcx
    mov     byte ptr [r9 + rcx], '-'
    
itoa_copy:
    # Copy from temp to output buffer
    movzx   r8, byte ptr [output_len]  # current length
    mov     r10, rcx            # start position in temp
    
itoa_copy_loop:
    movzx   rdx, byte ptr [r9 + r10]
    test    rdx, rdx
    jz      itoa_done
    mov     [rdi + r8], dl
    inc     r8
    inc     r10
    jmp     itoa_copy_loop
    
itoa_done:
    mov     byte ptr [output_len], r8b
    
    add     rsp, 32
    
    pop     r10
    pop     r9
    pop     r8
    pop     rdx
    pop     rcx
    pop     rbx
    pop     rax
    ret

.section .data
output_buffer: .space 1024
output_len: .byte 0
```
---

## String Marker

code:
```
.intel_syntax noprefix
.global _start

.section .text

_start:
    # Get arguments
    pop     rcx                 # argc
    pop     rbx                 # argv[0]
    
    # Get format string (argv[1])
    pop     rbx                 # rbx = format string pointer
    
    # Get first value argument (argv[2])
    pop     r12                 # r12 = current value pointer
    # r12 will be advanced as we consume %d and %s markers
    
    # Initialize output
    lea     rdi, [output_buffer]
    mov     byte ptr [output_len], 0

scan:
    movzx   eax, byte ptr [rbx]
    test    al, al
    jz      print
    
    cmp     al, '%'
    je      marker
    
    cmp     al, '\\'
    je      backslash
    
    # Normal character - write it
    movzx   rcx, byte ptr [output_len]
    mov     [rdi + rcx], al
    inc     byte ptr [output_len]
    inc     rbx
    jmp     scan

backslash:
    inc     rbx
    movzx   eax, byte ptr [rbx]
    cmp     al, 'n'
    jne     check_bs
    mov     al, 0x0A
    jmp     write_escape
check_bs:
    cmp     al, '\\'
    jne     scan
    mov     al, '\\'
write_escape:
    movzx   rcx, byte ptr [output_len]
    mov     [rdi + rcx], al
    inc     byte ptr [output_len]
    inc     rbx
    jmp     scan

marker:
    inc     rbx
    movzx   eax, byte ptr [rbx]
    
    cmp     al, '%'
    je      write_percent
    
    cmp     al, 'd'
    je      decimal
    
    cmp     al, 's'
    je      string
    
    # Unknown - treat as literal
    movzx   rcx, byte ptr [output_len]
    mov     byte ptr [rdi + rcx], '%'
    inc     byte ptr [output_len]
    jmp     scan

write_percent:
    movzx   rcx, byte ptr [output_len]
    mov     byte ptr [rdi + rcx], '%'
    inc     byte ptr [output_len]
    inc     rbx
    jmp     scan

decimal:
    # Save format pointer
    push    rbx
    push    rdi                 # Save output buffer pointer
    
    # Check if we have a value argument
    test    r12, r12
    jz      skip_decimal
    
    # Convert current value string to integer
    mov     rsi, r12
    call    atoi
    
    # Convert integer to string and append
    mov     rsi, rax
    call    itoa
    
    # Advance to next value argument
    pop     rdi                 # Restore buffer pointer
    pop     rbx                 # Restore format pointer
    
    # Get next argument
    pop     r12
    
    inc     rbx                 # Skip 'd'
    jmp     scan

skip_decimal:
    pop     rdi
    pop     rbx
    inc     rbx
    jmp     scan

string:
    # Save format pointer
    push    rbx
    push    rdi                 # Save output buffer pointer
    
    # Check if we have a value argument
    test    r12, r12
    jz      skip_string
    
    # Copy the string directly
    mov     rsi, r12
    call    copy_string
    
    # Advance to next value argument
    pop     rdi                 # Restore buffer pointer
    pop     rbx                 # Restore format pointer
    
    # Get next argument
    pop     r12
    
    inc     rbx                 # Skip 's'
    jmp     scan

skip_string:
    pop     rdi
    pop     rbx
    inc     rbx
    jmp     scan

print:
    mov     rax, 1
    mov     rdi, 1
    lea     rsi, [output_buffer]
    movzx   rdx, byte ptr [output_len]
    syscall
    
    mov     rax, 60
    xor     rdi, rdi
    syscall

# Convert string at rsi to int in rax
atoi:
    push    rsi
    push    rcx
    push    rdx
    
    xor     rax, rax
    mov     rcx, 1
    
    cmp     byte ptr [rsi], '-'
    jne     atoi_loop
    mov     rcx, -1
    inc     rsi
    
atoi_loop:
    movzx   rdx, byte ptr [rsi]
    test    rdx, rdx
    jz      atoi_done
    sub     rdx, '0'
    cmp     rdx, 9
    ja      atoi_done
    imul    rax, rax, 10
    add     rax, rdx
    inc     rsi
    jmp     atoi_loop
    
atoi_done:
    imul    rax, rcx
    
    pop     rdx
    pop     rcx
    pop     rsi
    ret

# Copy string from rsi to output buffer (literal copy, no escaping)
copy_string:
    push    rax
    push    rcx
    push    rsi
    push    rdi
    
    movzx   rcx, byte ptr [output_len]  # current length
    
copy_loop:
    movzx   rax, byte ptr [rsi]
    test    rax, rax
    jz      copy_done
    mov     [rdi + rcx], al
    inc     rcx
    inc     rsi
    jmp     copy_loop
    
copy_done:
    mov     byte ptr [output_len], cl
    
    pop     rdi
    pop     rsi
    pop     rcx
    pop     rax
    ret

# Convert int in rsi to string, append to output buffer
itoa:
    push    rax
    push    rbx
    push    rcx
    push    rdx
    push    r8
    push    r9
    push    r10
    
    mov     rax, rsi            # number to convert
    
    # Use stack as temp buffer
    sub     rsp, 32
    mov     r9, rsp             # r9 = temp buffer
    
    # Build number backwards in temp buffer
    mov     rcx, 31
    mov     byte ptr [r9 + rcx], 0  # null terminator
    
    # Handle negative
    xor     r10, r10            # sign flag
    test    rax, rax
    jns     itoa_pos
    mov     r10, 1              # negative
    neg     rax
    
itoa_pos:
    mov     rbx, 10
    
itoa_loop:
    xor     rdx, rdx
    div     rbx
    add     dl, '0'
    dec     rcx
    mov     [r9 + rcx], dl
    test    rax, rax
    jnz     itoa_loop
    
    # Add minus sign if negative
    test    r10, r10
    jz      itoa_copy
    dec     rcx
    mov     byte ptr [r9 + rcx], '-'
    
itoa_copy:
    # Copy from temp to output buffer
    movzx   r8, byte ptr [output_len]  # current length
    mov     r10, rcx            # start position in temp
    
itoa_copy_loop:
    movzx   rdx, byte ptr [r9 + r10]
    test    rdx, rdx
    jz      itoa_done
    mov     [rdi + r8], dl
    inc     r8
    inc     r10
    jmp     itoa_copy_loop
    
itoa_done:
    mov     byte ptr [output_len], r8b
    
    add     rsp, 32
    
    pop     r10
    pop     r9
    pop     r8
    pop     rdx
    pop     rcx
    pop     rbx
    pop     rax
    ret

.section .data
output_buffer: .space 1024
output_len: .byte 0
```

---

## Hex Byte Esxapes

code:
```
.intel_syntax noprefix
.global _start

.section .text

_start:
    # Get arguments
    pop     rcx                 # argc
    pop     rbx                 # argv[0]
    
    # Get format string (argv[1])
    pop     rbx                 # rbx = format string pointer
    
    # Get first value argument (argv[2])
    pop     r12                 # r12 = current value pointer
    
    # Initialize output
    lea     rdi, [output_buffer]
    mov     byte ptr [output_len], 0

scan:
    movzx   eax, byte ptr [rbx]
    test    al, al
    jz      print
    
    cmp     al, '%'
    je      marker
    
    cmp     al, '\\'
    je      backslash
    
    # Normal character - write it
    movzx   rcx, byte ptr [output_len]
    mov     [rdi + rcx], al
    inc     byte ptr [output_len]
    inc     rbx
    jmp     scan

backslash:
    inc     rbx
    movzx   eax, byte ptr [rbx]
    
    cmp     al, 'n'
    je      newline
    
    cmp     al, '\\'
    je      backslash_char
    
    cmp     al, 'x'
    je      hex_escape
    
    # Unknown escape - treat as literal
    movzx   rcx, byte ptr [output_len]
    mov     byte ptr [rdi + rcx], '\\'
    inc     byte ptr [output_len]
    jmp     scan

newline:
    movzx   rcx, byte ptr [output_len]
    mov     byte ptr [rdi + rcx], 0x0A
    inc     byte ptr [output_len]
    inc     rbx
    jmp     scan

backslash_char:
    movzx   rcx, byte ptr [output_len]
    mov     byte ptr [rdi + rcx], '\\'
    inc     byte ptr [output_len]
    inc     rbx
    jmp     scan

hex_escape:
    # Skip 'x'
    inc     rbx
    
    # Get first hex digit
    movzx   eax, byte ptr [rbx]
    call    hex_to_nibble
    cmp     al, 0xFF
    je      hex_invalid
    
    mov     dl, al              # Save first nibble in dl
    
    # Get second hex digit
    inc     rbx
    movzx   eax, byte ptr [rbx]
    call    hex_to_nibble
    cmp     al, 0xFF
    je      hex_invalid
    
    # Combine nibbles: (first << 4) | second
    shl     dl, 4
    or      dl, al
    
    # Write the byte
    movzx   rcx, byte ptr [output_len]
    mov     [rdi + rcx], dl
    inc     byte ptr [output_len]
    
    inc     rbx                 # Skip second digit
    jmp     scan

hex_invalid:
    # Invalid hex escape - treat as literal
    movzx   rcx, byte ptr [output_len]
    mov     byte ptr [rdi + rcx], '\\'
    inc     byte ptr [output_len]
    inc     rbx
    jmp     scan

marker:
    inc     rbx
    movzx   eax, byte ptr [rbx]
    
    cmp     al, '%'
    je      write_percent
    
    cmp     al, 'd'
    je      decimal
    
    cmp     al, 's'
    je      string
    
    # Unknown - treat as literal
    movzx   rcx, byte ptr [output_len]
    mov     byte ptr [rdi + rcx], '%'
    inc     byte ptr [output_len]
    jmp     scan

write_percent:
    movzx   rcx, byte ptr [output_len]
    mov     byte ptr [rdi + rcx], '%'
    inc     byte ptr [output_len]
    inc     rbx
    jmp     scan

decimal:
    push    rbx
    push    rdi
    
    test    r12, r12
    jz      skip_decimal
    
    mov     rsi, r12
    call    atoi
    
    mov     rsi, rax
    call    itoa
    
    pop     rdi
    pop     rbx
    
    pop     r12
    
    inc     rbx
    jmp     scan

skip_decimal:
    pop     rdi
    pop     rbx
    inc     rbx
    jmp     scan

string:
    push    rbx
    push    rdi
    
    test    r12, r12
    jz      skip_string
    
    mov     rsi, r12
    call    copy_string
    
    pop     rdi
    pop     rbx
    
    pop     r12
    
    inc     rbx
    jmp     scan

skip_string:
    pop     rdi
    pop     rbx
    inc     rbx
    jmp     scan

print:
    mov     rax, 1
    mov     rdi, 1
    lea     rsi, [output_buffer]
    movzx   rdx, byte ptr [output_len]
    syscall
    
    mov     rax, 60
    xor     rdi, rdi
    syscall

# Convert hex character in al to nibble (0-15) or 0xFF if invalid
hex_to_nibble:
    push    rdx
    
    cmp     al, '0'
    jl      hex_invalid_char
    cmp     al, '9'
    jle     hex_digit_0_9
    
    cmp     al, 'A'
    jl      hex_invalid_char
    cmp     al, 'F'
    jle     hex_digit_A_F
    
    cmp     al, 'a'
    jl      hex_invalid_char
    cmp     al, 'f'
    jle     hex_digit_a_f
    
hex_invalid_char:
    mov     al, 0xFF
    pop     rdx
    ret

hex_digit_0_9:
    sub     al, '0'
    pop     rdx
    ret

hex_digit_A_F:
    sub     al, 'A'
    add     al, 10
    pop     rdx
    ret

hex_digit_a_f:
    sub     al, 'a'
    add     al, 10
    pop     rdx
    ret

# Convert string at rsi to int in rax
atoi:
    push    rsi
    push    rcx
    push    rdx
    
    xor     rax, rax
    mov     rcx, 1
    
    cmp     byte ptr [rsi], '-'
    jne     atoi_loop
    mov     rcx, -1
    inc     rsi
    
atoi_loop:
    movzx   rdx, byte ptr [rsi]
    test    rdx, rdx
    jz      atoi_done
    sub     rdx, '0'
    cmp     rdx, 9
    ja      atoi_done
    imul    rax, rax, 10
    add     rax, rdx
    inc     rsi
    jmp     atoi_loop
    
atoi_done:
    imul    rax, rcx
    
    pop     rdx
    pop     rcx
    pop     rsi
    ret

# Copy string from rsi to output buffer (literal copy)
copy_string:
    push    rax
    push    rcx
    push    rsi
    push    rdi
    
    movzx   rcx, byte ptr [output_len]
    
copy_loop:
    movzx   rax, byte ptr [rsi]
    test    rax, rax
    jz      copy_done
    mov     [rdi + rcx], al
    inc     rcx
    inc     rsi
    jmp     copy_loop
    
copy_done:
    mov     byte ptr [output_len], cl
    
    pop     rdi
    pop     rsi
    pop     rcx
    pop     rax
    ret

# Convert int in rsi to string, append to output buffer
itoa:
    push    rax
    push    rbx
    push    rcx
    push    rdx
    push    r8
    push    r9
    push    r10
    
    mov     rax, rsi
    
    sub     rsp, 32
    mov     r9, rsp
    
    mov     rcx, 31
    mov     byte ptr [r9 + rcx], 0
    
    xor     r10, r10
    test    rax, rax
    jns     itoa_pos
    mov     r10, 1
    neg     rax
    
itoa_pos:
    mov     rbx, 10
    
itoa_loop:
    xor     rdx, rdx
    div     rbx
    add     dl, '0'
    dec     rcx
    mov     [r9 + rcx], dl
    test    rax, rax
    jnz     itoa_loop
    
    test    r10, r10
    jz      itoa_copy
    dec     rcx
    mov     byte ptr [r9 + rcx], '-'
    
itoa_copy:
    movzx   r8, byte ptr [output_len]
    mov     r10, rcx
    
itoa_copy_loop:
    movzx   rdx, byte ptr [r9 + r10]
    test    rdx, rdx
    jz      itoa_done
    mov     [rdi + r8], dl
    inc     r8
    inc     r10
    jmp     itoa_copy_loop
    
itoa_done:
    mov     byte ptr [output_len], r8b
    
    add     rsp, 32
    
    pop     r10
    pop     r9
    pop     r8
    pop     rdx
    pop     rcx
    pop     rbx
    pop     rax
    ret

.section .data
output_buffer: .space 1024
output_len: .byte 0
```
Problem:
The issue is in the hex escape handling. The problem is that when I combine the nibbles, I'm using al incorrectly. Let me trace through \x41:

First digit '4' → hex_to_nibble returns 4 in al

I save it in cl: mov cl, al (cl = 4)

Second digit '1' → hex_to_nibble returns 1 in al

I combine: shl cl, 4 (cl = 0x40), or cl, al (cl = 0x41)

BUG: Then I write mov byte ptr [rdi + rcx], cl but I'm using rcx which has been modified!

The output shows \x00\x01\x02 which means the hex conversion is way off

The problem is I'm using rcx for both the output position AND the hex value.
