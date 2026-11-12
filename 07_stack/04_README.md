# 04.asm: Stack Frame and Local Variables (32-bit)

## What the program does

```nasm
BITS 32
_start:
    call work

    ; Linux i386 exit syscall
    mov eax, 1
    xor ebx, ebx
    int 0x80

work:
    push ebp              ; prologue
    mov ebp, esp
    sub esp, 8

    mov dword [ebp - 4], 100    ; local variable 1
    mov dword [ebp - 8], 200    ; local variable 2

    mov eax, [ebp - 4]
    mov ebx, [ebp - 8]
    nop

    mov esp, ebp          ; epilogue
    pop ebp
    ret
```

This is the **32-bit version of [02.asm](02_README.md)**. `_start` calls `work`, which builds a stack frame, reserves 8 bytes for two 4-byte locals, stores 100 and 200 in them, loads them into EAX and EBX, removes the frame, and returns. The program exits through the 32-bit Linux system call interface (`int 0x80`, EAX = 1).

The registers are ESP, EBP, and EIP instead of RSP, RBP, and RIP, and every stack slot is **4 bytes** instead of 8.

## Build and run

32-bit programs need different assembler and linker options:

```bash
nasm -f elf32 -g -F dwarf 07_stack/04.asm -o build/07_stack/04.o
ld -m elf_i386 build/07_stack/04.o -o build/07_stack/04
./build/07_stack/04 ; echo $?
```

Result: there is no output, and the exit status is `0`. It ran without problems as a 32-bit binary on Ubuntu 24.04 under WSL2. The stack was traced in GDB (see [`logs/07_stack_trace_2026-11-12.txt`](../logs/07_stack_trace_2026-11-12.txt)).

Addresses from `objdump`:

```
08049000 <_start>:  call 804900e <work>
08049005:           mov  eax,0x1        <- return address
0804900e <work>:    push ebp
...
0804902c:           ret
```

## How the stack is used in this program

The stack is used in the same three ways as in 02.asm:

1. `call` pushes the 4-byte return address `0x08049005`;
2. `push ebp` saves the caller's EBP;
3. `sub esp, 8` reserves room for two `dword` locals at `[ebp-4]` and `[ebp-8]`.

In 32-bit mode, `push` and `pop` change ESP by **4**, and `call` and `ret` move a 4-byte address.

## Step-by-step trace (values observed in GDB)

ESP and EBP are shown as the last 3 hex digits of `0xffffdXXX`.

| # | Instruction | ESP after | Change | EBP after | What happened on the stack |
|---|-------------|-----------|--------|-----------|----------------------------|
| 0 | *(entry)* | `2b0` | n/a | 0 | Top of stack is `argc = 1` |
| 1 | `call work` | `2ac` | **-4** | 0 | Pushed return address **`0x08049005`** |
| 2 | `push ebp` | `2a8` | **-4** | 0 | Pushed old EBP (`0x0`) |
| 3 | `mov ebp, esp` | `2a8` | 0 | **`2a8`** | EBP marks the base of the frame |
| 4 | `sub esp, 8` | `2a0` | **-8** | `2a8` | Reserved 8 bytes for 2 locals |
| 5 | `mov [ebp-4], 100` | `2a0` | 0 | `2a8` | `0xffffd2a4` = `0x64` |
| 6 | `mov [ebp-8], 200` | `2a0` | 0 | `2a8` | `0xffffd2a0` = `0xc8` |
| 7 | `mov eax, [ebp-4]` | `2a0` | 0 | `2a8` | EAX = 0x64 |
| 8 | `mov ebx, [ebp-8]` | `2a0` | 0 | `2a8` | EBX = 0xc8 |
| 9 | `nop` | `2a0` | 0 | `2a8` | (inspection point) |
| 10 | `mov esp, ebp` | `2a8` | **+8** | `2a8` | Locals released |
| 11 | `pop ebp` | `2ac` | **+4** | **0** | Caller's EBP restored |
| 12 | `ret` | `2b0` | **+4** | 0 | Popped `0x08049005` into EIP |
| 13 | `mov eax, 1` | `2b0` | 0 | 0 | EAX = 1 (`sys_exit`) |
| 14 | `xor ebx, ebx` | `2b0` | 0 | 0 | EBX = 0 (exit code) |
| 15 | `int 0x80` | n/a | n/a | n/a | exit(0) |

### The stack frame at the `nop`

```
 higher addresses
 0xffffd2b4 | 0xffffd402 |  argv[0]
 0xffffd2b0 | 0x00000001 |  argc                 <- ESP before call / after ret
 0xffffd2ac | 0x08049005 |  return address       (pushed by call)
 0xffffd2a8 | 0x00000000 |  saved EBP            <- EBP
 0xffffd2a4 | 0x00000064 |  [ebp-4] local = 100
 0xffffd2a0 | 0x000000c8 |  [ebp-8] local = 200  <- ESP
 lower addresses
```

GDB output at the `nop`:

```
esp=0xffffd2a0  ebp=0xffffd2a8  eax=0x64  ebx=0xc8
0xffffd2a0: 0x000000c8  0x00000064  0x00000000  0x08049005
```

## Comparing with the 64-bit version (02.asm)

| | 02.asm (64-bit) | 04.asm (32-bit) |
|---|---|---|
| Stack pointer, frame pointer | RSP, RBP | ESP, EBP |
| Size of each push or pop | 8 bytes | 4 bytes |
| Return address pushed by `call` | `0x0000000000401005` (8 bytes) | `0x08049005` (4 bytes) |
| Space reserved for locals | `sub rsp, 16` | `sub esp, 8` |
| Local offsets | `[rbp-8]`, `[rbp-16]` | `[ebp-4]`, `[ebp-8]` |
| Total frame depth at the `nop` | 32 bytes (`e190 → e170`) | 16 bytes (`2b0 → 2a0`) |
| Stack address region | `0x7fffffffe…` | `0xffffd…` |
| Exit | `syscall`, RAX = 60, RDI = code | `int 0x80`, EAX = 1, EBX = code |

## Observations from running it

1. **Every stack step was 4 bytes.** `call`, `push ebp`, `pop ebp`, and `ret` each changed ESP by exactly 4. `sub esp, 8` reserved two 4-byte slots.
2. **The return address on the stack was `0x08049005`**, which is the `mov eax, 1` instruction right after `call work` (`call` at `0x08049000` is 5 bytes long).
3. **The 32-bit stack is in a different place.** A 32-bit process can only address 4 GB, so its stack is just below `0xffffffff` (around `0xffffd2b0`), instead of the 64-bit `0x7fffffffe190`.
4. **The frame has the same shape as the 64-bit one.** The order (return address, saved EBP, locals) and the reverse-order epilogue (`mov esp, ebp`, `pop ebp`, `ret`) are identical. Only the slot size changes.
5. **The stack is balanced.** ESP ended at `0xffffd2b0` and EBP at `0`, the same values as at entry.
6. Freed locals stay in memory below ESP until something overwrites them, just as in 02.asm.

## Summary

ESP goes `2b0 → 2ac` (call) `→ 2a8` (push ebp) `→ 2a0` (locals), then back `2a0 → 2a8 → 2ac → 2b0`. The frame is built and torn down in mirror-image order, with every slot 4 bytes wide because this is 32-bit code.
