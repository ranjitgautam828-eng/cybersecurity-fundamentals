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

