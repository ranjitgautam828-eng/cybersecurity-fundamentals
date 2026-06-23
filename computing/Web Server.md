# pwn.college — Assembly Language: Building a Web Server

A progression of x86-64 Linux assembly programs built as part of the pwn.college computing module. Each section builds on the previous, culminating in a full concurrent HTTP server handling GET and POST requests.

---

## 1. Exit

The simplest program: invoke the kernel's `exit` syscall.

```asm
.intel_syntax noprefix
.text
.global _start

_start:
    mov eax, 60          # syscall number for exit
    mov edi, 0           # exit status = 0
    syscall              # invoke kernel
```

---

## 2. Socket

Create a TCP socket using `socket(AF_INET, SOCK_STREAM, 0)`.

```asm
.intel_syntax noprefix
.text
.global _start

_start:
    # socket(AF_INET, SOCK_STREAM, 0)  — syscall 41
    mov eax, 41          # socket syscall
    mov edi, 2           # AF_INET = 2
    mov esi, 1           # SOCK_STREAM = 1
    mov edx, 0           # protocol = 0
    syscall

    mov eax, 60
    mov edi, 0
    syscall
```

---

## 3. Bind

Bind the socket to port 80 on `0.0.0.0`.

> **Gotcha — port byte order:** Port 80 in network (big-endian) byte order is `0x0050`. On a little-endian machine, store it as `0x5000` so the bytes in memory are `50 00`. Storing `0x0050` directly gives port `20480`.

```asm
.intel_syntax noprefix
.text
.global _start

_start:
    mov eax, 41
    mov edi, 2
    mov esi, 1
    mov edx, 0
    syscall
    mov r12, rax                         # save socket fd

    sub rsp, 16                          # allocate sockaddr_in on stack

    mov word ptr  [rsp],   0x0002        # AF_INET
    mov word ptr  [rsp+2], 0x5000        # port 80, network byte order
    mov dword ptr [rsp+4], 0x00000000    # INADDR_ANY
    mov qword ptr [rsp+8], 0             # padding

    # bind(sockfd, &addr, 16)  — syscall 49
    mov eax, 49
    mov edi, r12d
    mov rsi, rsp
    mov edx, 16
    syscall

    add rsp, 16

    mov eax, 60
    mov edi, 0
    syscall
```

---

## 4. Listen

Put the socket in a listening state.

> **Gotcha:** The challenge expects `listen(sockfd, 0)` (backlog = 0). Using any other backlog value fails the checker.

```asm
    # listen(sockfd, 0)  — syscall 50
    mov eax, 50
    mov edi, r12d
    mov esi, 0           # backlog = 0
    syscall
```

---

## 5. Accept

Block until a client connects, then return a new client file descriptor.

```asm
    # accept(sockfd, NULL, NULL)  — syscall 43
    mov eax, 43
    mov edi, r12d        # listening socket fd
    mov esi, 0           # addr = NULL
    mov edx, 0           # addrlen = NULL
    syscall
    # client fd is now in rax
```

---

## 6. Static Response

Accept one connection, read the request, send a fixed HTTP 200 response.

```asm
.intel_syntax noprefix
.section .data
response:
    .ascii "HTTP/1.0 200 OK\r\n\r\n"

.section .text
.global _start

_start:
    mov eax, 41
    mov edi, 2
    mov esi, 1
    mov edx, 0
    syscall
    mov r12, rax

    sub rsp, 16
    mov word ptr  [rsp],   0x0002
    mov word ptr  [rsp+2], 0x5000
    mov dword ptr [rsp+4], 0
    mov qword ptr [rsp+8], 0
    mov eax, 49
    mov edi, r12d
    mov rsi, rsp
    mov edx, 16
    syscall

    mov eax, 50
    mov edi, r12d
    mov esi, 0
    syscall

    mov eax, 43
    mov edi, r12d
    mov esi, 0
    mov edx, 0
    syscall
    mov r13, rax         # client fd

    add rsp, 16

    # read(client_fd, buffer, 1024)
    sub rsp, 1024
    mov eax, 0
    mov edi, r13d
    mov rsi, rsp
    mov edx, 1024
    syscall

    # write(client_fd, response, 19)
    mov eax, 1
    mov edi, r13d
    mov rsi, offset response
    mov edx, 19
    syscall

    # close(client_fd)
    mov eax, 3
    mov edi, r13d
    syscall

    add rsp, 1024
    mov eax, 60
    mov edi, 0
    syscall
```

---

## 7. Dynamic Response (Single Request)

Open the file path from the GET request and serve its contents, or return 404.

```asm
.intel_syntax noprefix
.section .data
http_ok:
    .ascii "HTTP/1.0 200 OK\r\n\r\n"
    .equ http_ok_len, . - http_ok

http_404:
    .ascii "HTTP/1.0 404 Not Found\r\n\r\n"
    .equ http_404_len, . - http_404

.section .bss
request_buffer:  .space 1024
filename_buffer: .space 256
file_buffer:     .space 4096

.section .text
.global _start

_start:
    # socket / bind / listen / accept  (same as before)
    # ...
    mov r13, rax         # client fd

    # read request
    mov eax, 0
    mov edi, r13d
    mov rsi, offset request_buffer
    mov edx, 1024
    syscall

    # parse filename — skip "GET "
    mov rsi, offset request_buffer
    add rsi, 4
    mov rdi, offset filename_buffer

parse_filename:
    mov al, byte ptr [rsi]
    cmp al, 32 ; je parse_done
    cmp al, 13 ; je parse_done
    cmp al, 10 ; je parse_done
    cmp al, 0  ; je parse_done
    mov byte ptr [rdi], al
    inc rsi
    inc rdi
    jmp parse_filename

parse_done:
    mov byte ptr [rdi], 0

    # open(filename, O_RDONLY)
    mov eax, 2
    mov rdi, offset filename_buffer
    mov esi, 0
    syscall

    cmp eax, 0
    jl file_not_found
    mov r15, rax

    # read file content
    mov eax, 0
    mov edi, r15d
    mov rsi, offset file_buffer
    mov edx, 4096
    syscall
    mov rbx, rax

    mov eax, 3 ; mov edi, r15d ; syscall   # close file

    # send header then content
    mov eax, 1 ; mov edi, r13d
    mov rsi, offset http_ok ; mov edx, http_ok_len ; syscall

    mov eax, 1 ; mov edi, r13d
    mov rsi, offset file_buffer ; mov edx, ebx ; syscall
    jmp finish

file_not_found:
    mov eax, 1 ; mov edi, r13d
    mov rsi, offset http_404 ; mov edx, http_404_len ; syscall

finish:
    mov eax, 3 ; mov edi, r13d ; syscall   # close client
    mov eax, 60 ; mov edi, 0 ; syscall
```

---

## 8. Iterative GET Server

Loop indefinitely, accepting and serving one connection at a time.

Key addition: after `accept`, process the request, then `jmp accept_loop`.

```asm
accept_loop:
    mov eax, 43
    mov edi, r12d
    mov esi, 0
    mov edx, 0
    syscall
    cmp eax, 0 ; jl exit_server
    mov r13, rax

    # ... read, parse, serve ...

close_client:
    mov eax, 3 ; mov edi, r13d ; syscall
    jmp accept_loop

exit_server:
    mov eax, 60 ; mov edi, 0 ; syscall
```

---

## 9. Concurrent GET Server

Fork a child process for each connection so multiple clients are served simultaneously.

```asm
accept_loop:
    mov eax, 43 ; mov edi, r12d ; xor esi, esi ; xor edx, edx ; syscall
    cmp eax, 0 ; jl exit_server
    mov r13, rax

    mov eax, 57          # fork()
    syscall

    cmp eax, 0
    je child_process

    # Parent: close client fd, go back to accepting
    mov eax, 3 ; mov edi, r13d ; syscall
    jmp accept_loop

child_process:
    # Child: close listening socket, handle request, exit
    mov eax, 3 ; mov edi, r12d ; syscall
    # ... read, parse, serve, close client ...
    mov eax, 60 ; mov edi, 0 ; syscall

exit_server:
    mov eax, 60 ; mov edi, 0 ; syscall
```

---

## 10. Concurrent POST Server

Handle POST requests: parse the filename from the request line, find the body after `\r\n\r\n`, and write it to disk.

### Key insights

**Content-Length parsing is unnecessary.** Since pwn.college delivers the entire request in a single `read`, body length can be calculated as:

```
body_length = total_bytes_read − (body_start − buffer_start)
```

**File flags the checker expects:**

| Flag | Value |
|------|-------|
| `O_WRONLY` | 1 |
| `O_CREAT`  | 64 |
| Combined   | **65** |

Use `open` (syscall 2) with flags `65` and mode `0777`. The checker specifically looks for `open`, not `creat` (syscall 85).

### Body-finding loop

```asm
find_body_start:
    cmp rsi, rax         # rax = buffer_end - 4
    jge parse_error

    cmp byte ptr [rsi],   13 ; jne next_body
    cmp byte ptr [rsi+1], 10 ; jne next_body
    cmp byte ptr [rsi+2], 13 ; jne next_body
    cmp byte ptr [rsi+3], 10 ; je  body_found

next_body:
    inc rsi
    jmp find_body_start

body_found:
    add rsi, 4           # skip \r\n\r\n
    mov r15, rsi         # r15 = body pointer
```

### Body length + file write

```asm
    mov rbx, r11         # r11 = buffer_end
    sub rbx, r15         # rbx = body length

    mov eax, 2
    mov rdi, offset filename_buffer
    mov esi, 65          # O_WRONLY|O_CREAT
    mov edx, 0777
    syscall
    mov r12, rax

    mov eax, 1 ; mov edi, r12d
    mov rsi, r15 ; mov edx, ebx ; syscall   # write body

    mov eax, 3 ; mov edi, r12d ; syscall    # close file
```

### Lessons learned

- Don't modify the bounds pointer (`rax`) inside the search loop — only advance `rsi`.
- The `open` syscall clobbers `rsi`; save the body pointer in `r15` before calling it.
- Use `creat` only if the checker explicitly expects it; here it expects `open`.

---

## 11. Full Concurrent Web Server (GET + POST)

Combines everything: fork-per-connection, method routing, GET file serving, POST file writing, and correct HTTP error codes.

### Method detection

Comparing a multi-byte dword is fragile due to endianness. Byte-by-byte comparison is safer:

```asm
    mov rsi, offset request_buffer

    cmp byte ptr [rsi],   'G' ; jne check_post
    cmp byte ptr [rsi+1], 'E' ; jne check_post
    cmp byte ptr [rsi+2], 'T' ; jne check_post
    cmp byte ptr [rsi+3], ' ' ; jne check_post
    jmp handle_get

check_post:
    cmp byte ptr [rsi],   'P' ; jne method_not_allowed
    cmp byte ptr [rsi+1], 'O' ; jne method_not_allowed
    cmp byte ptr [rsi+2], 'S' ; jne method_not_allowed
    cmp byte ptr [rsi+3], 'T' ; jne method_not_allowed
    jmp handle_post
```

> Note on dword comparison: `"GET "` in memory is bytes `47 45 54 20`. As a little-endian dword that is `0x20544547`, not `0x544547`. Getting this wrong causes silent mismatches.

### HTTP response codes used

| Status | When |
|--------|------|
| `200 OK` | Successful GET or POST |
| `404 Not Found` | GET — file does not exist |
| `405 Method Not Allowed` | Request is neither GET nor POST |
| `500 Internal Server Error` | Parse error, `open` failure, etc. |

### Full server code

```asm
.intel_syntax noprefix
.section .data
http_ok:
    .ascii "HTTP/1.0 200 OK\r\n\r\n"
    .equ http_ok_len, . - http_ok

http_404:
    .ascii "HTTP/1.0 404 Not Found\r\n\r\n"
    .equ http_404_len, . - http_404

http_500:
    .ascii "HTTP/1.0 500 Internal Server Error\r\n\r\n"
    .equ http_500_len, . - http_500

http_method_not_allowed:
    .ascii "HTTP/1.0 405 Method Not Allowed\r\n\r\n"
    .equ http_method_not_allowed_len, . - http_method_not_allowed

.section .bss
request_buffer:  .space 8192
filename_buffer: .space 256
file_buffer:     .space 4096

.section .text
.global _start

_start:
    mov eax, 41 ; mov edi, 2 ; mov esi, 1 ; mov edx, 0 ; syscall
    mov r12, rax

    sub rsp, 16
    mov word ptr  [rsp],   0x0002
    mov word ptr  [rsp+2], 0x5000
    mov dword ptr [rsp+4], 0
    mov qword ptr [rsp+8], 0
    mov eax, 49 ; mov edi, r12d ; mov rsi, rsp ; mov edx, 16 ; syscall
    mov eax, 50 ; mov edi, r12d ; mov esi, 0 ; syscall
    add rsp, 16

accept_loop:
    mov eax, 43 ; mov edi, r12d ; xor esi, esi ; xor edx, edx ; syscall
    cmp eax, 0 ; jl exit_server
    mov r13, rax

    mov eax, 57 ; syscall       # fork
    cmp eax, 0 ; je child_process

    mov eax, 3 ; mov edi, r13d ; syscall   # parent: close client
    jmp accept_loop

child_process:
    mov eax, 3 ; mov edi, r12d ; syscall   # child: close listener

    mov eax, 0 ; mov edi, r13d
    mov rsi, offset request_buffer ; mov edx, 8192 ; syscall
    cmp rax, 0 ; jle parse_error
    mov r14, rax
    mov r11, offset request_buffer ; add r11, r14

    mov rsi, offset request_buffer
    cmp byte ptr [rsi],   'G' ; jne check_post
    cmp byte ptr [rsi+1], 'E' ; jne check_post
    cmp byte ptr [rsi+2], 'T' ; jne check_post
    cmp byte ptr [rsi+3], ' ' ; jne check_post
    jmp handle_get

check_post:
    cmp byte ptr [rsi],   'P' ; jne method_not_allowed
    cmp byte ptr [rsi+1], 'O' ; jne method_not_allowed
    cmp byte ptr [rsi+2], 'S' ; jne method_not_allowed
    cmp byte ptr [rsi+3], 'T' ; jne method_not_allowed
    jmp handle_post

handle_get:
    mov rsi, offset request_buffer ; add rsi, 4
    mov rdi, offset filename_buffer

parse_filename_get:
    cmp rsi, r11 ; jge parse_error
    mov al, byte ptr [rsi]
    cmp al, 32 ; je parse_done_get
    cmp al, 13 ; je parse_done_get
    cmp al, 10 ; je parse_done_get
    cmp al, 0  ; je parse_done_get
    mov byte ptr [rdi], al ; inc rsi ; inc rdi
    jmp parse_filename_get

parse_done_get:
    mov byte ptr [rdi], 0

    mov eax, 2 ; mov rdi, offset filename_buffer ; xor esi, esi ; syscall
    cmp eax, 0 ; jl file_not_found
    mov r15, rax

    mov eax, 0 ; mov edi, r15d
    mov rsi, offset file_buffer ; mov edx, 4096 ; syscall
    mov rbx, rax

    mov eax, 3 ; mov edi, r15d ; syscall   # close file

    mov eax, 1 ; mov edi, r13d
    mov rsi, offset http_ok ; mov edx, http_ok_len ; syscall

    mov eax, 1 ; mov edi, r13d
    mov rsi, offset file_buffer ; mov edx, ebx ; syscall
    jmp child_done

handle_post:
    mov rsi, offset request_buffer ; add rsi, 5
    mov rdi, offset filename_buffer

parse_filename_post:
    cmp rsi, r11 ; jge parse_error
    mov al, byte ptr [rsi]
    cmp al, 32 ; je parse_done_post
    cmp al, 13 ; je parse_done_post
    cmp al, 10 ; je parse_done_post
    cmp al, 0  ; je parse_done_post
    mov byte ptr [rdi], al ; inc rsi ; inc rdi
    jmp parse_filename_post

parse_done_post:
    mov byte ptr [rdi], 0

    mov rsi, offset request_buffer
    mov rax, r11 ; sub rax, 4
    cmp rsi, rax ; jge parse_error

find_body_start:
    cmp rsi, rax ; jge parse_error
    cmp byte ptr [rsi],   13 ; jne next_body
    cmp byte ptr [rsi+1], 10 ; jne next_body
    cmp byte ptr [rsi+2], 13 ; jne next_body
    cmp byte ptr [rsi+3], 10 ; je  body_found
next_body:
    inc rsi ; jmp find_body_start

body_found:
    add rsi, 4
    mov r15, rsi            # body pointer
    mov rbx, r11 ; sub rbx, r15    # body length

    mov eax, 2 ; mov rdi, offset filename_buffer
    mov esi, 65 ; mov edx, 0777 ; syscall
    cmp eax, 0 ; jl open_error
    mov r12, rax

    mov eax, 1 ; mov edi, r12d
    mov rsi, r15 ; mov edx, ebx ; syscall

    mov eax, 3 ; mov edi, r12d ; syscall   # close file

    mov eax, 1 ; mov edi, r13d
    mov rsi, offset http_ok ; mov edx, http_ok_len ; syscall
    jmp child_done

file_not_found:
    mov eax, 1 ; mov edi, r13d
    mov rsi, offset http_404 ; mov edx, http_404_len ; syscall
    jmp child_done

method_not_allowed:
    mov eax, 1 ; mov edi, r13d
    mov rsi, offset http_method_not_allowed
    mov edx, http_method_not_allowed_len ; syscall
    jmp child_done

parse_error:
open_error:
    mov eax, 1 ; mov edi, r13d
    mov rsi, offset http_500 ; mov edx, http_500_len ; syscall

child_done:
    mov eax, 3 ; mov edi, r13d ; syscall
    mov eax, 60 ; mov edi, 0 ; syscall

exit_server:
    mov eax, 60 ; mov edi, 0 ; syscall
```

---

## Syscall Reference

| Syscall | Number | Key Registers |
|---------|--------|---------------|
| `read`   | 0  | rdi=fd, rsi=buf, rdx=count |
| `write`  | 1  | rdi=fd, rsi=buf, rdx=count |
| `open`   | 2  | rdi=path, rsi=flags, rdx=mode |
| `close`  | 3  | rdi=fd |
| `socket` | 41 | rdi=domain, rsi=type, rdx=proto |
| `accept` | 43 | rdi=fd, rsi=addr, rdx=addrlen |
| `bind`   | 49 | rdi=fd, rsi=addr, rdx=addrlen |
| `listen` | 50 | rdi=fd, rsi=backlog |
| `fork`   | 57 | — |
| `exit`   | 60 | rdi=status |

## Common Pitfalls

| Issue | Fix |
|-------|-----|
| Port in wrong byte order | Store port 80 as `0x5000`, not `0x0050` |
| Wrong backlog in `listen` | Use `0` unless told otherwise |
| `creat` vs `open` | Checker expects `open(path, 65, 0777)` |
| `rsi` clobbered by `open` | Save body pointer in `r15` before the call |
| Advancing bounds pointer in loop | Only advance `rsi`; keep `rax` (limit) fixed |
| Dword method check endianness | Use byte-by-byte comparison for `GET`/`POST` |
