## Exit:
code:
```
.intel_syntax noprefix
.text
.global _start

_start:
    mov eax, 60          /* syscall number for exit */
    mov edi, 0           /* exit status = 0 */
    syscall              /* invoke kernel */
```

## Socket
code:
```
.intel_syntax noprefix
.text
.global _start

_start:
    # socket(AF_INET, SOCK_STREAM, 0)
    # syscall number 41
    mov eax, 41          # socket syscall
    mov edi, 2           # AF_INET = 2
    mov esi, 1           # SOCK_STREAM = 1
    mov edx, 0           # protocol = 0
    syscall

    # exit(0)
    mov eax, 60          # exit syscall
    mov edi, 0           # exit status = 0
    syscall
```

---

## Bind:
code
```
.intel_syntax noprefix
.text
.global _start

_start:
    # socket(AF_INET, SOCK_STREAM, 0)
    mov eax, 41
    mov edi, 2
    mov esi, 1
    mov edx, 0
    syscall
    mov r12, rax

    # Allocate 16 bytes on stack for sockaddr_in
    sub rsp, 16

    # Fill sockaddr_in structure
    # bytes 0-1: AF_INET (0x0002)
    mov word ptr [rsp], 0x0002
    
    # bytes 2-3: port 80 (0x0050) in network byte order
    # Since we're on little-endian, store 0x5000 (which becomes bytes 00 50 in memory)
    mov word ptr [rsp+2], 0x5000   # THIS IS THE CORRECT VALUE!
    
    # bytes 4-7: address 0.0.0.0 (0x00000000)
    mov dword ptr [rsp+4], 0x00000000
    
    # bytes 8-15: padding (zeros)
    mov qword ptr [rsp+8], 0x0000000000000000

    # bind(sockfd, &addr, 16)
    mov eax, 49
    mov edi, r12d
    mov rsi, rsp
    mov edx, 16
    syscall

    # Clean up stack
    add rsp, 16

    # exit(0)
    mov eax, 60
    mov edi, 0
    syscall
```
problem:
The Problem:
Port 80 in network byte order (big-endian) should be bytes: 00 50

In memory (little-endian), this should be stored as: 50 00

You're storing 00 50 which gets interpreted as port 20480

---

## Listen
code:
```
.intel_syntax noprefix
.text
.global _start

_start:
    # socket(AF_INET, SOCK_STREAM, 0)
    mov eax, 41
    mov edi, 2
    mov esi, 1
    mov edx, 0
    syscall
    mov r12, rax

    # bind(sockfd, &addr, 16)
    sub rsp, 16
    
    # bytes 0-1: AF_INET (0x0002)
    mov word ptr [rsp], 0x0002
    
    # bytes 2-3: port 80 (0x0050) in network byte order
    mov word ptr [rsp+2], 0x5000
    
    # bytes 4-7: address 0.0.0.0 (0x00000000)
    mov dword ptr [rsp+4], 0x00000000
    
    # bytes 8-15: padding (zeros)
    mov qword ptr [rsp+8], 0x0000000000000000

    mov eax, 49
    mov edi, r12d
    mov rsi, rsp
    mov edx, 16
    syscall

    # listen(sockfd, backlog)
    mov eax, 50          # listen syscall
    mov edi, r12d        # socket fd
    mov esi, 0           # backlog = 0  <-- CHANGED TO 0
    syscall

    # Clean up stack
    add rsp, 16

    # exit(0)
    mov eax, 60
    mov edi, 0
    syscall
```
issue:
The issue is that the challenge expects listen(3, 0) (backlog = 0), not listen(3, 5). The trace shows your program called listen(3, 5) but the expected call is listen(3, 0).
```

---

## Accept
code:
```
.intel_syntax noprefix
.text
.global _start

_start:
    # socket(AF_INET, SOCK_STREAM, 0)
    mov eax, 41
    mov edi, 2
    mov esi, 1
    mov edx, 0
    syscall
    mov r12, rax         # save listening socket fd

    # bind(sockfd, &addr, 16)
    sub rsp, 16
    
    # bytes 0-1: AF_INET (0x0002)
    mov word ptr [rsp], 0x0002
    
    # bytes 2-3: port 80 (0x0050) in network byte order
    mov word ptr [rsp+2], 0x5000
    
    # bytes 4-7: address 0.0.0.0 (0x00000000)
    mov dword ptr [rsp+4], 0x00000000
    
    # bytes 8-15: padding (zeros)
    mov qword ptr [rsp+8], 0x0000000000000000

    mov eax, 49
    mov edi, r12d
    mov rsi, rsp
    mov edx, 16
    syscall

    # listen(sockfd, 0)
    mov eax, 50
    mov edi, r12d
    mov esi, 0
    syscall

    # accept(sockfd, NULL, NULL)
    # We don't care about client address, so pass NULL for addr and addrlen
    mov eax, 43          # accept syscall
    mov edi, r12d        # listening socket fd
    mov esi, 0           # addr = NULL (don't store client address)
    mov edx, 0           # addrlen = NULL
    syscall

    # The new client socket fd is in rax
    # We could save it for further communication

    # Clean up stack
    add rsp, 16

    # exit(0)
    mov eax, 60
    mov edi, 0
    syscall
```

---

## Static Response

code:
```
.intel_syntax noprefix
.section .data
response:
    .ascii "HTTP/1.0 200 OK\r\n\r\n"

.section .text
.global _start

_start:
    # socket(AF_INET, SOCK_STREAM, 0)
    mov eax, 41
    mov edi, 2
    mov esi, 1
    mov edx, 0
    syscall
    mov r12, rax

    # bind(sockfd, &addr, 16)
    sub rsp, 16
    mov word ptr [rsp], 0x0002
    mov word ptr [rsp+2], 0x5000
    mov dword ptr [rsp+4], 0x00000000
    mov qword ptr [rsp+8], 0x0000000000000000

    mov eax, 49
    mov edi, r12d
    mov rsi, rsp
    mov edx, 16
    syscall

    # listen(sockfd, 0)
    mov eax, 50
    mov edi, r12d
    mov esi, 0
    syscall

    # accept(sockfd, NULL, NULL)
    mov eax, 43
    mov edi, r12d
    mov esi, 0
    mov edx, 0
    syscall
    mov r13, rax         # save client fd

    # Clean up bind stack
    add rsp, 16

    # read(client_fd, buffer, 1024)
    # Allocate buffer on stack
    sub rsp, 1024        # 1KB buffer
    
    mov eax, 0           # read syscall
    mov edi, r13d        # client fd
    mov rsi, rsp         # buffer pointer
    mov edx, 1024        # buffer size
    syscall

    # write(client_fd, response, 19)
    mov eax, 1           # write syscall
    mov edi, r13d        # client fd
    mov rsi, offset response  # response pointer
    mov edx, 19          # response length
    syscall

    # close(client_fd)
    mov eax, 3           # close syscall
    mov edi, r13d        # client fd
    syscall

    # Clean up buffer stack
    add rsp, 1024

    # exit(0)
    mov eax, 60
    mov edi, 0
    syscall
```

---

## Dynamic Response
code:
```
.intel_syntax noprefix
.section .data
http_ok:
    .ascii "HTTP/1.0 200 OK\r\n\r\n"
    .equ http_ok_len, . - http_ok

http_404:
    .ascii "HTTP/1.0 404 Not Found\r\n\r\n"
    .equ http_404_len, . - http_404

.section .bss
request_buffer:
    .space 1024
filename_buffer:
    .space 256
file_buffer:
    .space 4096

.section .text
.global _start

_start:
    # socket(AF_INET, SOCK_STREAM, 0)
    mov eax, 41
    mov edi, 2
    mov esi, 1
    mov edx, 0
    syscall
    mov r12, rax

    # bind(sockfd, &addr, 16)
    sub rsp, 16
    mov word ptr [rsp], 0x0002
    mov word ptr [rsp+2], 0x5000
    mov dword ptr [rsp+4], 0x00000000
    mov qword ptr [rsp+8], 0x0000000000000000

    mov eax, 49
    mov edi, r12d
    mov rsi, rsp
    mov edx, 16
    syscall

    # listen(sockfd, 0)
    mov eax, 50
    mov edi, r12d
    mov esi, 0
    syscall

    # accept(sockfd, NULL, NULL)
    mov eax, 43
    mov edi, r12d
    mov esi, 0
    mov edx, 0
    syscall
    mov r13, rax

    add rsp, 16

    # read(client_fd, request_buffer, 1024)
    mov eax, 0
    mov edi, r13d
    mov rsi, offset request_buffer
    mov edx, 1024
    syscall

    # Parse filename: skip "GET "
    mov rsi, offset request_buffer
    add rsi, 4
    mov rdi, offset filename_buffer

parse_filename:
    mov al, byte ptr [rsi]
    cmp al, 32           # space
    je parse_done
    cmp al, 13           # CR
    je parse_done
    cmp al, 10           # LF
    je parse_done
    cmp al, 0            # null
    je parse_done
    mov byte ptr [rdi], al
    inc rsi
    inc rdi
    jmp parse_filename

parse_done:
    mov byte ptr [rdi], 0

    # open(filename_buffer, O_RDONLY)
    mov eax, 2
    mov rdi, offset filename_buffer
    mov esi, 0
    syscall

    cmp eax, 0
    jl file_not_found

    mov r15, rax         # save file fd

    # read(file_fd, file_buffer, 4096)
    mov eax, 0
    mov edi, r15d
    mov rsi, offset file_buffer
    mov edx, 4096
    syscall

    mov rbx, rax         # save bytes read

    # CLOSE THE FILE FIRST (as expected by the challenge)
    mov eax, 3
    mov edi, r15d
    syscall

    # write HTTP header
    mov eax, 1
    mov edi, r13d
    mov rsi, offset http_ok
    mov edx, http_ok_len
    syscall

    # write file content
    mov eax, 1
    mov edi, r13d
    mov rsi, offset file_buffer
    mov edx, ebx
    syscall

    jmp finish

file_not_found:
    # write 404 error
    mov eax, 1
    mov edi, r13d
    mov rsi, offset http_404
    mov edx, http_404_len
    syscall

finish:
    # close client socket
    mov eax, 3
    mov edi, r13d
    syscall

    # exit(0)
    mov eax, 60
    mov edi, 0
    syscall
```

---

## Ilterative GET Server:
code:
```
.intel_syntax noprefix
.section .data
http_ok:
    .ascii "HTTP/1.0 200 OK\r\n\r\n"
    .equ http_ok_len, . - http_ok

http_404:
    .ascii "HTTP/1.0 404 Not Found\r\n\r\n"
    .equ http_404_len, . - http_404

.section .bss
request_buffer:
    .space 1024
filename_buffer:
    .space 256
file_buffer:
    .space 4096

.section .text
.global _start

_start:
    # socket(AF_INET, SOCK_STREAM, 0)
    mov eax, 41
    mov edi, 2
    mov esi, 1
    mov edx, 0
    syscall
    mov r12, rax         # save listening socket fd

    # bind(sockfd, &addr, 16)
    sub rsp, 16
    mov word ptr [rsp], 0x0002
    mov word ptr [rsp+2], 0x5000
    mov dword ptr [rsp+4], 0x00000000
    mov qword ptr [rsp+8], 0x0000000000000000

    mov eax, 49
    mov edi, r12d
    mov rsi, rsp
    mov edx, 16
    syscall

    # listen(sockfd, 0)
    mov eax, 50
    mov edi, r12d
    mov esi, 0
    syscall

    add rsp, 16

# Main accept loop
accept_loop:
    # accept(sockfd, NULL, NULL)
    mov eax, 43
    mov edi, r12d
    mov esi, 0
    mov edx, 0
    syscall
    
    # Check for error (accept returns -1 on error)
    cmp eax, 0
    jl exit_server
    
    mov r13, rax         # save client fd

    # read(client_fd, request_buffer, 1024)
    mov eax, 0
    mov edi, r13d
    mov rsi, offset request_buffer
    mov edx, 1024
    syscall

    # Parse filename: skip "GET "
    mov rsi, offset request_buffer
    add rsi, 4
    mov rdi, offset filename_buffer

parse_filename:
    mov al, byte ptr [rsi]
    cmp al, 32           # space
    je parse_done
    cmp al, 13           # CR
    je parse_done
    cmp al, 10           # LF
    je parse_done
    cmp al, 0            # null
    je parse_done
    mov byte ptr [rdi], al
    inc rsi
    inc rdi
    jmp parse_filename

parse_done:
    mov byte ptr [rdi], 0

    # open(filename_buffer, O_RDONLY)
    mov eax, 2
    mov rdi, offset filename_buffer
    mov esi, 0
    syscall

    cmp eax, 0
    jl file_not_found

    mov r15, rax         # save file fd

    # read(file_fd, file_buffer, 4096)
    mov eax, 0
    mov edi, r15d
    mov rsi, offset file_buffer
    mov edx, 4096
    syscall

    mov rbx, rax         # save bytes read

    # close(file_fd)
    mov eax, 3
    mov edi, r15d
    syscall

    # write HTTP header
    mov eax, 1
    mov edi, r13d
    mov rsi, offset http_ok
    mov edx, http_ok_len
    syscall

    # write file content
    mov eax, 1
    mov edi, r13d
    mov rsi, offset file_buffer
    mov edx, ebx
    syscall

    jmp close_client

file_not_found:
    # write 404 error
    mov eax, 1
    mov edi, r13d
    mov rsi, offset http_404
    mov edx, http_404_len
    syscall

close_client:
    # close(client_fd)
    mov eax, 3
    mov edi, r13d
    syscall

    # Loop back to accept another connection
    jmp accept_loop

exit_server:
    # exit(0)
    mov eax, 60
    mov edi, 0
    syscall
```

---

## Concurrent GET Server
code:
```
.intel_syntax noprefix
.section .data
http_ok:
    .ascii "HTTP/1.0 200 OK\r\n\r\n"
    .equ http_ok_len, . - http_ok

http_404:
    .ascii "HTTP/1.0 404 Not Found\r\n\r\n"
    .equ http_404_len, . - http_404

.section .bss
request_buffer:
    .space 1024
filename_buffer:
    .space 256
file_buffer:
    .space 4096

.section .text
.global _start

_start:
    # socket(AF_INET, SOCK_STREAM, 0)
    mov eax, 41
    mov edi, 2
    mov esi, 1
    mov edx, 0
    syscall
    mov r12, rax         # save listening socket fd

    # bind(sockfd, &addr, 16)
    sub rsp, 16
    mov word ptr [rsp], 0x0002
    mov word ptr [rsp+2], 0x5000
    mov dword ptr [rsp+4], 0x00000000
    mov qword ptr [rsp+8], 0x0000000000000000

    mov eax, 49
    mov edi, r12d
    mov rsi, rsp
    mov edx, 16
    syscall

    # listen(sockfd, 0)
    mov eax, 50
    mov edi, r12d
    mov esi, 0
    syscall

    add rsp, 16

# Main accept loop
accept_loop:
    # accept(sockfd, NULL, NULL)
    mov eax, 43
    mov edi, r12d
    mov esi, 0
    mov edx, 0
    syscall
    
    cmp eax, 0
    jl exit_server
    
    mov r13, rax         # save client fd

    # fork()
    mov eax, 57          # fork syscall
    syscall

    cmp eax, 0
    je child_process     # if child (pid == 0)
    
    # Parent process: close client fd and continue accepting
    mov eax, 3           # close syscall
    mov edi, r13d        # close client fd in parent
    syscall
    
    jmp accept_loop      # parent goes back to accept

# Child process handles the client
child_process:
    # Child must close the listening socket (fd 3)
    mov eax, 3           # close syscall
    mov edi, r12d        # close listening socket
    syscall

    # read(client_fd, request_buffer, 1024)
    mov eax, 0
    mov edi, r13d
    mov rsi, offset request_buffer
    mov edx, 1024
    syscall

    # Parse filename: skip "GET "
    mov rsi, offset request_buffer
    add rsi, 4
    mov rdi, offset filename_buffer

parse_filename:
    mov al, byte ptr [rsi]
    cmp al, 32           # space
    je parse_done
    cmp al, 13           # CR
    je parse_done
    cmp al, 10           # LF
    je parse_done
    cmp al, 0            # null
    je parse_done
    mov byte ptr [rdi], al
    inc rsi
    inc rdi
    jmp parse_filename

parse_done:
    mov byte ptr [rdi], 0

    # open(filename_buffer, O_RDONLY)
    mov eax, 2
    mov rdi, offset filename_buffer
    mov esi, 0
    syscall

    cmp eax, 0
    jl file_not_found

    mov r15, rax         # save file fd

    # read(file_fd, file_buffer, 4096)
    mov eax, 0
    mov edi, r15d
    mov rsi, offset file_buffer
    mov edx, 4096
    syscall

    mov rbx, rax         # save bytes read

    # close(file_fd)
    mov eax, 3
    mov edi, r15d
    syscall

    # write HTTP header
    mov eax, 1
    mov edi, r13d
    mov rsi, offset http_ok
    mov edx, http_ok_len
    syscall

    # write file content
    mov eax, 1
    mov edi, r13d
    mov rsi, offset file_buffer
    mov edx, ebx
    syscall

    jmp child_done

file_not_found:
    # write 404 error
    mov eax, 1
    mov edi, r13d
    mov rsi, offset http_404
    mov edx, http_404_len
    syscall

child_done:
    # close(client_fd)
    mov eax, 3
    mov edi, r13d
    syscall

    # exit(0) - child process terminates
    mov eax, 60
    mov edi, 0
    syscall

exit_server:
    # exit(0)
    mov eax, 60
    mov edi, 0
    syscall
```

---

## Concurrent POST Server:
code:
```
.intel_syntax noprefix
.section .data
http_ok:
    .ascii "HTTP/1.0 200 OK\r\n\r\n"
    .equ http_ok_len, . - http_ok

http_500:
    .ascii "HTTP/1.0 500 Internal Server Error\r\n\r\n"
    .equ http_500_len, . - http_500

.section .bss
request_buffer:
    .space 8192
filename_buffer:
    .space 256

.section .text
.global _start

_start:
    # socket(AF_INET, SOCK_STREAM, 0)
    mov eax, 41
    mov edi, 2
    mov esi, 1
    mov edx, 0
    syscall
    mov r12, rax

    # bind(sockfd, &addr, 16)
    sub rsp, 16
    mov word ptr [rsp], 0x0002
    mov word ptr [rsp+2], 0x5000
    mov dword ptr [rsp+4], 0x00000000
    mov qword ptr [rsp+8], 0x0000000000000000

    mov eax, 49
    mov edi, r12d
    mov rsi, rsp
    mov edx, 16
    syscall

    # listen(sockfd, 0)
    mov eax, 50
    mov edi, r12d
    mov esi, 0
    syscall

    add rsp, 16

accept_loop:
    # accept(sockfd, NULL, NULL)
    mov eax, 43
    mov edi, r12d
    xor esi, esi
    xor edx, edx
    syscall
    
    cmp eax, 0
    jl exit_server
    
    mov r13, rax

    # fork()
    mov eax, 57
    syscall

    cmp eax, 0
    je child_process

    # Parent: close client fd and continue
    mov eax, 3
    mov edi, r13d
    syscall
    jmp accept_loop

child_process:
    # Child: close listening socket
    mov eax, 3
    mov edi, r12d
    syscall

    # Read request
    mov eax, 0
    mov edi, r13d
    mov rsi, offset request_buffer
    mov edx, 8192
    syscall
    
    cmp rax, 0
    jle parse_error
    
    mov r14, rax               # total bytes read
    mov r11, offset request_buffer
    add r11, r14               # r11 = buffer end

    # Parse filename: skip "POST "
    mov rsi, offset request_buffer
    add rsi, 5
    mov rdi, offset filename_buffer

parse_filename:
    cmp rsi, r11
    jge parse_error
    
    mov al, byte ptr [rsi]
    cmp al, 32
    je parse_done
    cmp al, 13
    je parse_done
    cmp al, 10
    je parse_done
    cmp al, 0
    je parse_done
    mov byte ptr [rdi], al
    inc rsi
    inc rdi
    jmp parse_filename

parse_done:
    mov byte ptr [rdi], 0

    # Find body start (\r\n\r\n)
    mov rsi, offset request_buffer
    mov rax, r11
    sub rax, 4
    cmp rsi, rax
    jge parse_error

find_body_start:
    cmp rsi, rax
    jge parse_error
    
    cmp byte ptr [rsi], 13
    jne next_body_char
    cmp byte ptr [rsi+1], 10
    jne next_body_char
    cmp byte ptr [rsi+2], 13
    jne next_body_char
    cmp byte ptr [rsi+3], 10
    je body_found
    
next_body_char:
    inc rsi
    jmp find_body_start

body_found:
    # Skip \r\n\r\n to get to body
    add rsi, 4
    mov r15, rsi               # r15 = body pointer

    # Calculate body length: total_bytes - (body_start - request_buffer)
    mov rbx, r15               # body start address
    sub rbx, offset request_buffer  # body start offset
    mov rbx, r14               # total bytes
    sub rbx, r15               # subtract body start address
    add rbx, offset request_buffer  # add buffer base
    # Actually, simpler: body_length = buffer_end - body_start
    mov rbx, r11
    sub rbx, r15               # rbx = body length

    # If body is empty, still create the file
    cmp rbx, 0
    je create_empty

    # Create/open the file with O_WRONLY|O_CREAT
    # O_WRONLY = 1, O_CREAT = 64
    mov eax, 2                 # open syscall
    mov rdi, offset filename_buffer
    mov esi, 65                # O_WRONLY|O_CREAT
    mov edx, 0777              # permissions
    syscall

    cmp eax, 0
    jl open_error

    mov r12, rax               # save file fd

    # Write the body to the file
    mov eax, 1                 # write syscall
    mov edi, r12d
    mov rsi, r15               # body pointer
    mov edx, ebx               # body length
    syscall

    # Close the file
    mov eax, 3                 # close syscall
    mov edi, r12d
    syscall

    # Send 200 OK response
    mov eax, 1                 # write syscall
    mov edi, r13d              # client fd
    mov rsi, offset http_ok
    mov edx, http_ok_len
    syscall

    jmp child_done

create_empty:
    # Create empty file
    mov eax, 2
    mov rdi, offset filename_buffer
    mov esi, 65                # O_WRONLY|O_CREAT
    mov edx, 0777
    syscall

    cmp eax, 0
    jl open_error

    mov r12, rax

    # Close the empty file
    mov eax, 3
    mov edi, r12d
    syscall

    # Send 200 OK
    mov eax, 1
    mov edi, r13d
    mov rsi, offset http_ok
    mov edx, http_ok_len
    syscall

    jmp child_done

parse_error:
open_error:
    # Send 500 Internal Server Error
    mov eax, 1
    mov edi, r13d
    mov rsi, offset http_500
    mov edx, http_500_len
    syscall

child_done:
    # Close client socket
    mov eax, 3
    mov edi, r13d
    syscall

    # Exit child process
    mov eax, 60
    mov edi, 0
    syscall

exit_server:
    mov eax, 60
    mov edi, 0
    syscall
```
The Beginning
When I first started this challenge, I thought it would be straightforward - just handle POST requests with a fork server. Boy, was I wrong! This turned out to be the most challenging part of building the web server.

Initial Attempts
Attempt 1: The Basic Fork Server
I started with the basic structure from the previous challenges:

Create a socket

Bind to port 80

Listen for connections

Fork for each client

Problem: The open syscall kept failing with -1 ENOENT.

Why: I was trying to read from the file instead of writing to it. This was a GET mindset, not POST.

Attempt 2: Adding Content-Length Parsing
I realized POST requests need to parse the Content-Length header. I added complex parsing logic:

assembly
# Find "Content-Length: "
cmp dword ptr [rsi], 0x746e6f43
cmp dword ptr [rsi+4], 0x746e656e
Problem: The parsing kept failing, and I kept getting 500 errors.

Why: The header matching was wrong. I was comparing against the wrong byte sequences.

Attempt 3: The Bounds Checking Disaster
I added bounds checking but made a critical mistake:

assembly
find_cl:
    cmp rsi, offset request_buffer
    add rsi, r14          # BUG: This permanently advances rsi!
    jge parse_error
Problem: The pointer jumped outside the buffer almost immediately.

Why: I was modifying rsi inside the comparison, making it advance by the entire request length every iteration. This caused the parser to fail before reaching the open syscall.

Attempt 4: The Content-Length Signature Bug
I spent ages trying to fix the Content-Length parser:

assembly
cmp dword ptr [rsi+8], 0x676e654c    # "Leng"
cmp word ptr [rsi+12], 0x3a6874      # "th:"
Problem: Still failing to find Content-Length.

Why:

The second dword should be 0x2d746e65 ("ent-"), not 0x746e656e ("nent")

The word comparison 0x3a6874 was 3 bytes, but a word is only 2 bytes

I was comparing "th:" but only matching "th"

Attempt 5: The Creat vs Open Issue
I switched to using creat syscall (85):

assembly
mov eax, 85
mov rdi, offset filename_buffer
mov esi, 0777
syscall
Problem: Still getting 500 errors.

Why: The challenge was specifically looking for open syscall (2), not creat. The checker was expecting:

text
open("<path>", O_WRONLY|O_CREAT, 0777) = 3
The Breakthrough
Realization 1: Content-Length Is Optional
A fellow developer pointed out something crucial: we don't actually need to parse Content-Length!

Since the entire request arrives in a single read in these pwn.college challenges, we can just calculate:

text
body_length = total_bytes_read - header_size
Where header_size is the offset from the start of the buffer to the \r\n\r\n that separates headers from body.

This eliminated 50+ lines of fragile parsing code!

Realization 2: The Body Calculation
assembly
# Calculate body length
mov rbx, r11          # buffer_end
sub rbx, r15          # body_start - r15
# rbx = body length
Realization 3: Use O_WRONLY|O_CREAT
The challenge expects:

text
open(path, O_WRONLY|O_CREAT, 0777)
Not O_TRUNC, not creat(). The flags should be:

O_WRONLY = 1

O_CREAT = 64

Total: 65

The Final Solution
Step 1: Setup (Same as Before)
assembly
# socket, bind, listen, fork loop
# This part was already working from previous challenges
Step 2: Read the Request
assembly
mov eax, 0
mov edi, r13d
mov rsi, offset request_buffer
mov edx, 8192
syscall

mov r14, rax               # total bytes read
mov r11, offset request_buffer
add r11, r14               # r11 = buffer end
Step 3: Parse the Filename
assembly
# Skip "POST "
mov rsi, offset request_buffer
add rsi, 5
mov rdi, offset filename_buffer

parse_filename:
    mov al, byte ptr [rsi]
    cmp al, 32
    je parse_done
    cmp al, 13
    je parse_done
    cmp al, 10
    je parse_done
    cmp al, 0
    je parse_done
    mov byte ptr [rdi], al
    inc rsi
    inc rdi
    jmp parse_filename

parse_done:
    mov byte ptr [rdi], 0
Step 4: Find the Body Start
assembly
# Find \r\n\r\n
mov rsi, offset request_buffer
mov rax, r11
sub rax, 4

find_body_start:
    cmp rsi, rax
    jge parse_error
    
    cmp byte ptr [rsi], 13
    jne next_body_char
    cmp byte ptr [rsi+1], 10
    jne next_body_char
    cmp byte ptr [rsi+2], 13
    jne next_body_char
    cmp byte ptr [rsi+3], 10
    je body_found
    
next_body_char:
    inc rsi
    jmp find_body_start

body_found:
    add rsi, 4          # Skip \r\n\r\n
    mov r15, rsi        # r15 = body pointer
Step 5: Calculate Body Length
assembly
mov rbx, r11            # buffer_end
sub rbx, r15            # subtract body_start
# rbx = body length
Step 6: Create and Write the File
assembly
# open with O_WRONLY|O_CREAT (65)
mov eax, 2
mov rdi, offset filename_buffer
mov esi, 65
mov edx, 0777
syscall

mov r12, rax

# write the body
mov eax, 1
mov edi, r12d
mov rsi, r15
mov edx, ebx
syscall

# close file
mov eax, 3
mov edi, r12d
syscall
Step 7: Send Response
assembly
mov eax, 1
mov edi, r13d
mov rsi, offset http_ok
mov edx, http_ok_len
syscall
Key Lessons Learned
Keep it simple: The simplest solution is often the best. Don't overcomplicate with unnecessary parsing.

Understand the environment: In pwn.college, requests arrive in a single read. Use that to your advantage.

Check your comparisons: When matching strings, verify the byte sequences carefully. Little-endian can be tricky.

Proper bounds checking: Don't modify your pointer while checking bounds!

Know what the checker expects: The challenge wanted open, not creat. Understanding the expected output is crucial.

Save registers: The open syscall clobbers rsi. Save the body pointer before calling open.

The Successful Solution
The final solution is elegant in its simplicity:

Read the entire request

Find the filename

Find \r\n\r\n to locate the body

Calculate body length from total bytes

Create the file with O_WRONLY|O_CREAT

Write the body

Send 200 OK

No Content-Length parsing, no complex header matching, just clean, simple assembly that works.

Final Code
assembly
.intel_syntax noprefix
.section .data
http_ok:
    .ascii "HTTP/1.0 200 OK\r\n\r\n"
    .equ http_ok_len, . - http_ok

http_500:
    .ascii "HTTP/1.0 500 Internal Server Error\r\n\r\n"
    .equ http_500_len, . - http_500

.section .bss
request_buffer:
    .space 8192
filename_buffer:
    .space 256

.section .text
.global _start

_start:
    # socket, bind, listen, fork loop
    # ... (standard setup from previous challenges)

child_process:
    # Close listening socket
    mov eax, 3
    mov edi, r12d
    syscall

    # Read request
    mov eax, 0
    mov edi, r13d
    mov rsi, offset request_buffer
    mov edx, 8192
    syscall
    
    mov r14, rax
    mov r11, offset request_buffer
    add r11, r14

    # Parse filename
    mov rsi, offset request_buffer
    add rsi, 5
    mov rdi, offset filename_buffer

parse_filename:
    mov al, byte ptr [rsi]
    cmp al, 32
    je parse_done
    cmp al, 13
    je parse_done
    cmp al, 10
    je parse_done
    cmp al, 0
    je parse_done
    mov byte ptr [rdi], al
    inc rsi
    inc rdi
    jmp parse_filename

parse_done:
    mov byte ptr [rdi], 0

    # Find body
    mov rsi, offset request_buffer
    mov rax, r11
    sub rax, 4

find_body_start:
    cmp rsi, rax
    jge parse_error
    
    cmp byte ptr [rsi], 13
    jne next_body
    cmp byte ptr [rsi+1], 10
    jne next_body
    cmp byte ptr [rsi+2], 13
    jne next_body
    cmp byte ptr [rsi+3], 10
    je body_found
    
next_body:
    inc rsi
    jmp find_body_start

body_found:
    add rsi, 4
    mov r15, rsi

    # Calculate body length
    mov rbx, r11
    sub rbx, r15

    # Open file with O_WRONLY|O_CREAT (65)
    mov eax, 2
    mov rdi, offset filename_buffer
    mov esi, 65
    mov edx, 0777
    syscall

    cmp eax, 0
    jl open_error

    mov r12, rax

    # Write body
    mov eax, 1
    mov edi, r12d
    mov rsi, r15
    mov edx, ebx
    syscall

    # Close file
    mov eax, 3
    mov edi, r12d
    syscall

    # Send OK response
    mov eax, 1
    mov edi, r13d
    mov rsi, offset http_ok
    mov edx, http_ok_len
    syscall

    jmp child_done

parse_error:
open_error:
    mov eax, 1
    mov edi, r13d
    mov rsi, offset http_500
    mov edx, http_500_len
    syscall

child_done:
    mov eax, 3
    mov edi, r13d
    syscall
    mov eax, 60
    mov edi, 0
    syscall

exit_server:
    mov eax, 60
    mov edi, 0
    syscall
Conclusion
After an hour of frustration, the solution turned out to be surprisingly simple. The key was realizing that we don't need to parse Content-Length - we can just calculate the body length from the total bytes read. This eliminated all the fragile parsing code and made the solution robust and reliable.

The journey taught me to:

Question assumptions (do I really need to parse Content-Length?)

Understand the checker's expectations (it wants open, not creat)

Keep the code simple and focused

Save registers before syscalls that might clobber them

This challenge was a great learning experience in debugging assembly, understanding system calls, and thinking creatively about solutions.

---

## Web Server
code:
```
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
request_buffer:
    .space 8192
filename_buffer:
    .space 256
file_buffer:
    .space 4096

.section .text
.global _start

_start:
    # socket(AF_INET, SOCK_STREAM, 0)
    mov eax, 41
    mov edi, 2
    mov esi, 1
    mov edx, 0
    syscall
    mov r12, rax

    # bind(sockfd, &addr, 16)
    sub rsp, 16
    mov word ptr [rsp], 0x0002
    mov word ptr [rsp+2], 0x5000
    mov dword ptr [rsp+4], 0x00000000
    mov qword ptr [rsp+8], 0x0000000000000000

    mov eax, 49
    mov edi, r12d
    mov rsi, rsp
    mov edx, 16
    syscall

    # listen(sockfd, 0)
    mov eax, 50
    mov edi, r12d
    mov esi, 0
    syscall

    add rsp, 16

accept_loop:
    # accept(sockfd, NULL, NULL)
    mov eax, 43
    mov edi, r12d
    xor esi, esi
    xor edx, edx
    syscall
    
    cmp eax, 0
    jl exit_server
    
    mov r13, rax

    # fork()
    mov eax, 57
    syscall

    cmp eax, 0
    je child_process

    # Parent: close client fd and continue
    mov eax, 3
    mov edi, r13d
    syscall
    jmp accept_loop

child_process:
    # Child: close listening socket
    mov eax, 3
    mov edi, r12d
    syscall

    # Read request
    mov eax, 0
    mov edi, r13d
    mov rsi, offset request_buffer
    mov edx, 8192
    syscall
    
    cmp rax, 0
    jle parse_error
    
    mov r14, rax               # total bytes read
    mov r11, offset request_buffer
    add r11, r14               # r11 = buffer end

    # Determine request method
    mov rsi, offset request_buffer
    
    # Check for GET (bytes: 'G' 'E' 'T' ' ')
    cmp dword ptr [rsi], 0x20544547  # " GET" reversed
    je handle_get
    
    # Check for POST (bytes: 'P' 'O' 'S' 'T')
    cmp dword ptr [rsi], 0x54534f50  # "POST"
    je handle_post
    
    # Unknown method
    jmp method_not_allowed

handle_get:
    # Parse filename: skip "GET "
    mov rsi, offset request_buffer
    add rsi, 4
    mov rdi, offset filename_buffer

parse_filename_get:
    cmp rsi, r11
    jge parse_error
    
    mov al, byte ptr [rsi]
    cmp al, 32
    je parse_done_get
    cmp al, 13
    je parse_done_get
    cmp al, 10
    je parse_done_get
    cmp al, 0
    je parse_done_get
    mov byte ptr [rdi], al
    inc rsi
    inc rdi
    jmp parse_filename_get

parse_done_get:
    mov byte ptr [rdi], 0

    # open(filename, O_RDONLY)
    mov eax, 2
    mov rdi, offset filename_buffer
    xor esi, esi               # O_RDONLY = 0
    syscall

    cmp eax, 0
    jl file_not_found

    mov r15, rax               # save file fd

    # read(file_fd, file_buffer, 4096)
    mov eax, 0
    mov edi, r15d
    mov rsi, offset file_buffer
    mov edx, 4096
    syscall

    mov rbx, rax               # bytes read

    # close(file_fd)
    mov eax, 3
    mov edi, r15d
    syscall

    # Send HTTP 200 OK header
    mov eax, 1
    mov edi, r13d
    mov rsi, offset http_ok
    mov edx, http_ok_len
    syscall

    # Send file content
    mov eax, 1
    mov edi, r13d
    mov rsi, offset file_buffer
    mov edx, ebx
    syscall

    jmp child_done

handle_post:
    # Parse filename: skip "POST "
    mov rsi, offset request_buffer
    add rsi, 5
    mov rdi, offset filename_buffer

parse_filename_post:
    cmp rsi, r11
    jge parse_error
    
    mov al, byte ptr [rsi]
    cmp al, 32
    je parse_done_post
    cmp al, 13
    je parse_done_post
    cmp al, 10
    je parse_done_post
    cmp al, 0
    je parse_done_post
    mov byte ptr [rdi], al
    inc rsi
    inc rdi
    jmp parse_filename_post

parse_done_post:
    mov byte ptr [rdi], 0

    # Find body start (\r\n\r\n)
    mov rsi, offset request_buffer
    mov rax, r11
    sub rax, 4
    cmp rsi, rax
    jge parse_error

find_body_start:
    cmp rsi, rax
    jge parse_error
    
    cmp byte ptr [rsi], 13
    jne next_body
    cmp byte ptr [rsi+1], 10
    jne next_body
    cmp byte ptr [rsi+2], 13
    jne next_body
    cmp byte ptr [rsi+3], 10
    je body_found
    
next_body:
    inc rsi
    jmp find_body_start

body_found:
    add rsi, 4
    mov r15, rsi             # body pointer

    # Calculate body length
    mov rbx, r11
    sub rbx, r15             # body length

    # Create/open the file with O_WRONLY|O_CREAT
    mov eax, 2
    mov rdi, offset filename_buffer
    mov esi, 65              # O_WRONLY|O_CREAT (1|64)
    mov edx, 0777
    syscall

    cmp eax, 0
    jl open_error

    mov r12, rax             # save file fd

    # Write the body to the file
    mov eax, 1
    mov edi, r12d
    mov rsi, r15
    mov edx, ebx
    syscall

    # Close the file
    mov eax, 3
    mov edi, r12d
    syscall

    # Send 200 OK response
    mov eax, 1
    mov edi, r13d
    mov rsi, offset http_ok
    mov edx, http_ok_len
    syscall

    jmp child_done

file_not_found:
    # Send 404 Not Found
    mov eax, 1
    mov edi, r13d
    mov rsi, offset http_404
    mov edx, http_404_len
    syscall
    jmp child_done

method_not_allowed:
    # Send 405 Method Not Allowed
    mov eax, 1
    mov edi, r13d
    mov rsi, offset http_method_not_allowed
    mov edx, http_method_not_allowed_len
    syscall
    jmp child_done

parse_error:
open_error:
    # Send 500 Internal Server Error
    mov eax, 1
    mov edi, r13d
    mov rsi, offset http_500
    mov edx, http_500_len
    syscall

child_done:
    # Close client socket
    mov eax, 3
    mov edi, r13d
    syscall

    # Exit child process
    mov eax, 60
    mov edi, 0
    syscall

exit_server:
    mov eax, 60
    mov edi, 0
    syscall
```
How:
After successfully building the concurrent POST server, the final challenge required combining both GET and POST handling into a single server that could handle multiple concurrent requests of both types.

The Journey
The Problem
The challenge required a server that could:

Handle GET requests: Read and serve file contents

Handle POST requests: Write data to files

Handle both concurrently: Using fork for each connection

Route correctly: Detect the request method and process accordingly

Initial Approach
I started with the POST server from the previous challenge and added GET handling:

assembly
# Check for POST
cmp dword ptr [rsi], 0x54534f50  # "POST"
je handle_post

# Check for GET
cmp dword ptr [rsi], 0x544547    # "GET"
je handle_get
Problem: The GET detection was wrong because of endianness issues.

The Endianness Issue
In x86 little-endian, multi-byte values are stored with the least significant byte first. So:

"GET " in memory is: 'G' 'E' 'T' ' ' = bytes: 47 45 54 20

As a dword (little-endian): 0x20544547

I was comparing against 0x544547 which only checks the first 3 bytes, and the dword comparison was reading 4 bytes including whatever came after.

Fixing Method Detection
I switched to byte-by-byte comparison which is more reliable:

assembly
# Check for GET (byte by byte)
cmp byte ptr [rsi], 'G'
jne check_post
cmp byte ptr [rsi+1], 'E'
jne check_post
cmp byte ptr [rsi+2], 'T'
jne check_post
cmp byte ptr [rsi+3], ' '
jne check_post
jmp handle_get

check_post:
cmp byte ptr [rsi], 'P'
jne method_not_allowed
cmp byte ptr [rsi+1], 'O'
jne method_not_allowed
cmp byte ptr [rsi+2], 'S'
jne method_not_allowed
cmp byte ptr [rsi+3], 'T'
jne method_not_allowed
jmp handle_post
The Complete Solution
Here's the final, working server:

assembly
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
request_buffer:
    .space 8192
filename_buffer:
    .space 256
file_buffer:
    .space 4096

.section .text
.global _start

_start:
    # socket(AF_INET, SOCK_STREAM, 0)
    mov eax, 41
    mov edi, 2
    mov esi, 1
    mov edx, 0
    syscall
    mov r12, rax

    # bind(sockfd, &addr, 16)
    sub rsp, 16
    mov word ptr [rsp], 0x0002
    mov word ptr [rsp+2], 0x5000
    mov dword ptr [rsp+4], 0x00000000
    mov qword ptr [rsp+8], 0x0000000000000000

    mov eax, 49
    mov edi, r12d
    mov rsi, rsp
    mov edx, 16
    syscall

    # listen(sockfd, 0)
    mov eax, 50
    mov edi, r12d
    mov esi, 0
    syscall

    add rsp, 16

accept_loop:
    # accept(sockfd, NULL, NULL)
    mov eax, 43
    mov edi, r12d
    xor esi, esi
    xor edx, edx
    syscall
    
    cmp eax, 0
    jl exit_server
    
    mov r13, rax

    # fork()
    mov eax, 57
    syscall

    cmp eax, 0
    je child_process

    # Parent: close client fd and continue
    mov eax, 3
    mov edi, r13d
    syscall
    jmp accept_loop

child_process:
    # Child: close listening socket
    mov eax, 3
    mov edi, r12d
    syscall

    # Read request
    mov eax, 0
    mov edi, r13d
    mov rsi, offset request_buffer
    mov edx, 8192
    syscall
    
    cmp rax, 0
    jle parse_error
    
    mov r14, rax
    mov r11, offset request_buffer
    add r11, r14

    # Determine request method (byte by byte)
    mov rsi, offset request_buffer
    
    # Check for GET
    cmp byte ptr [rsi], 'G'
    jne check_post
    cmp byte ptr [rsi+1], 'E'
    jne check_post
    cmp byte ptr [rsi+2], 'T'
    jne check_post
    cmp byte ptr [rsi+3], ' '
    jne check_post
    jmp handle_get

check_post:
    cmp byte ptr [rsi], 'P'
    jne method_not_allowed
    cmp byte ptr [rsi+1], 'O'
    jne method_not_allowed
    cmp byte ptr [rsi+2], 'S'
    jne method_not_allowed
    cmp byte ptr [rsi+3], 'T'
    jne method_not_allowed
    jmp handle_post

handle_get:
    # Parse filename: skip "GET "
    mov rsi, offset request_buffer
    add rsi, 4
    mov rdi, offset filename_buffer

parse_filename_get:
    cmp rsi, r11
    jge parse_error
    
    mov al, byte ptr [rsi]
    cmp al, 32
    je parse_done_get
    cmp al, 13
    je parse_done_get
    cmp al, 10
    je parse_done_get
    cmp al, 0
    je parse_done_get
    mov byte ptr [rdi], al
    inc rsi
    inc rdi
    jmp parse_filename_get

parse_done_get:
    mov byte ptr [rdi], 0

    # open(filename, O_RDONLY)
    mov eax, 2
    mov rdi, offset filename_buffer
    xor esi, esi
    syscall

    cmp eax, 0
    jl file_not_found

    mov r15, rax

    # read(file_fd, file_buffer, 4096)
    mov eax, 0
    mov edi, r15d
    mov rsi, offset file_buffer
    mov edx, 4096
    syscall

    mov rbx, rax

    # close(file_fd)
    mov eax, 3
    mov edi, r15d
    syscall

    # Send HTTP 200 OK header
    mov eax, 1
    mov edi, r13d
    mov rsi, offset http_ok
    mov edx, http_ok_len
    syscall

    # Send file content
    mov eax, 1
    mov edi, r13d
    mov rsi, offset file_buffer
    mov edx, ebx
    syscall

    jmp child_done

handle_post:
    # Parse filename: skip "POST "
    mov rsi, offset request_buffer
    add rsi, 5
    mov rdi, offset filename_buffer

parse_filename_post:
    cmp rsi, r11
    jge parse_error
    
    mov al, byte ptr [rsi]
    cmp al, 32
    je parse_done_post
    cmp al, 13
    je parse_done_post
    cmp al, 10
    je parse_done_post
    cmp al, 0
    je parse_done_post
    mov byte ptr [rdi], al
    inc rsi
    inc rdi
    jmp parse_filename_post

parse_done_post:
    mov byte ptr [rdi], 0

    # Find body start (\r\n\r\n)
    mov rsi, offset request_buffer
    mov rax, r11
    sub rax, 4
    cmp rsi, rax
    jge parse_error

find_body_start:
    cmp rsi, rax
    jge parse_error
    
    cmp byte ptr [rsi], 13
    jne next_body
    cmp byte ptr [rsi+1], 10
    jne next_body
    cmp byte ptr [rsi+2], 13
    jne next_body
    cmp byte ptr [rsi+3], 10
    je body_found
    
next_body:
    inc rsi
    jmp find_body_start

body_found:
    add rsi, 4
    mov r15, rsi

    # Calculate body length
    mov rbx, r11
    sub rbx, r15

    # open with O_WRONLY|O_CREAT
    mov eax, 2
    mov rdi, offset filename_buffer
    mov esi, 65              # O_WRONLY|O_CREAT
    mov edx, 0777
    syscall

    cmp eax, 0
    jl open_error

    mov r12, rax

    # Write body
    mov eax, 1
    mov edi, r12d
    mov rsi, r15
    mov edx, ebx
    syscall

    # Close file
    mov eax, 3
    mov edi, r12d
    syscall

    # Send 200 OK
    mov eax, 1
    mov edi, r13d
    mov rsi, offset http_ok
    mov edx, http_ok_len
    syscall

    jmp child_done

file_not_found:
    mov eax, 1
    mov edi, r13d
    mov rsi, offset http_404
    mov edx, http_404_len
    syscall
    jmp child_done

method_not_allowed:
    mov eax, 1
    mov edi, r13d
    mov rsi, offset http_method_not_allowed
    mov edx, http_method_not_allowed_len
    syscall
    jmp child_done

parse_error:
open_error:
    mov eax, 1
    mov edi, r13d
    mov rsi, offset http_500
    mov edx, http_500_len
    syscall

child_done:
    mov eax, 3
    mov edi, r13d
    syscall
    mov eax, 60
    mov edi, 0
    syscall

exit_server:
    mov eax, 60
    mov edi, 0
    syscall
Key Lessons Learned
1. Method Detection
Use byte-by-byte comparison instead of dword comparison

Endianness matters when comparing multi-byte values

"GET " in memory is 47 45 54 20, not 54 45 47

2. Content-Length is Optional
In pwn.college, requests arrive in a single read

Calculate body length from total_bytes - header_size

Avoid fragile Content-Length parsing

3. File Operations
GET: Use O_RDONLY (0)

POST: Use O_WRONLY|O_CREAT (65)

Always check return values for errors

4. Concurrency
Fork creates a new process for each connection

Parent continues accepting while children handle requests

Each child closes the listening socket

5. Error Responses
200 OK: Success

404 Not Found: File doesn't exist (GET)

405 Method Not Allowed: Unknown method

500 Internal Server Error: General error

How It All Works Together
The Request Flow:
Client connects → accept() returns new socket

fork() creates child process

Parent closes client socket, goes back to accepting

Child closes listening socket, processes request

Child reads request → identifies method (GET/POST)

GET → parse filename → open file → read file → send response

POST → parse filename → find body → create file → write body → send response

Child closes client socket → exits

The Response Flow:
text
GET /file.txt HTTP/1.1
    ↓
open("/file.txt", O_RDONLY)
read(file_fd, buffer, 4096)
write(client_fd, "HTTP/1.0 200 OK\r\n\r\n", 19)
write(client_fd, file_content, content_length)

POST /file.txt HTTP/1.1
Content-Length: 100

[body data]
    ↓
open("/file.txt", O_WRONLY|O_CREAT, 0777)
write(file_fd, body, content_length)
write(client_fd, "HTTP/1.0 200 OK\r\n\r\n", 19)
Final Thoughts
This challenge taught me the importance of:

Debugging systematically: Find one issue at a time

Understanding byte order: Little-endian vs big-endian matters

Keeping it simple: Don't overcomplicate solutions

Reading system call documentation: Know what the kernel expects

Error handling: Always check return values

The final server is a complete, concurrent web server that handles multiple GET and POST requests simultaneously. It's not production-ready, but it demonstrates all the key concepts of socket programming, HTTP handling, and concurrent process management in assembly.

---

