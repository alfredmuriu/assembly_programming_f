# 02.asm: Function Call with a Stack Frame and Local Variables (64-bit)

## What the program does

```nasm
_start:
    call work
    mov rax, 60
    xor rdi, rdi
    syscall

work:
    push rbp              ; prologue
    mov rbp, rsp
    sub rsp, 16

    mov qword [rbp - 8], 100    ; local variable 1
    mov qword [rbp - 16], 200   ; local variable 2

    mov rax, [rbp - 8]
    mov rbx, [rbp - 16]
    nop

    mov rsp, rbp          ; epilogue
    pop rbp
    ret
```

`_start` calls a function named `work`. `work` builds a **stack frame** and reserves 16 bytes on the stack for two 8-byte local variables. It stores 100 and 200 in them, reads them back into RAX and RBX, removes the frame, and returns. `_start` then exits with status 0.

## Build and run

```bash
nasm -f elf64 -g -F dwarf 07_stack/02.asm -o build/07_stack/02.o
ld build/07_stack/02.o -o build/07_stack/02
./build/07_stack/02 ; echo $?
```

Result: there is no output, and the exit status is `0`. I traced the stack with GDB `stepi` (see [`logs/07_stack_trace_2026-11-12.txt`](../logs/07_stack_trace_2026-11-12.txt)).

Addresses from `objdump`:

```
401000 <_start>:     call 40100f <work>
401005:              mov  eax,0x3c          <- the instruction after the call (return address)
40100f <work>:       push rbp
...
401034:              ret
```

## How the stack is used in this program

The stack does three jobs here:

1. **Holds the return address.** `call` pushes the address of the next instruction so that `ret` knows where to go back to.
2. **Saves the caller's RBP.** `push rbp` stores the caller's frame pointer so it can be restored before returning.
3. **Holds local variables.** `sub rsp, 16` reserves space that is accessed as `[rbp - 8]` and `[rbp - 16]`.

**RBP (the base/frame pointer)** stays fixed for the whole body of `work`. That makes it a stable reference point, so local variables are at constant negative offsets from it, even if RSP were to move.

## Step-by-step trace (values observed in GDB)

RSP and RBP are shown as the last 3 hex digits of `0x7fffffffeXXX`.

| # | Instruction | RSP after | Change | RBP after | What happened on the stack |
|---|-------------|-----------|--------|-----------|----------------------------|
| 0 | *(entry)* | `e190` | n/a | 0 | Top of stack is `argc = 1` |
| 1 | `call work` | `e188` | **-8** | 0 | Pushed return address **`0x401005`**, jumped to `work` |
| 2 | `push rbp` | `e180` | **-8** | 0 | Pushed old RBP (`0x0`) |
| 3 | `mov rbp, rsp` | `e180` | 0 | **`e180`** | RBP now marks the base of this frame |
| 4 | `sub rsp, 16` | `e170` | **-16** | `e180` | Reserved 16 bytes for 2 locals (they read as 0) |
| 5 | `mov [rbp-8], 100` | `e170` | 0 | `e180` | `0x7fffffffe178` = `0x64` |
| 6 | `mov [rbp-16], 200` | `e170` | 0 | `e180` | `0x7fffffffe170` = `0xc8` |
| 7 | `mov rax, [rbp-8]` | `e170` | 0 | `e180` | RAX = 0x64 (read, stack unchanged) |
| 8 | `mov rbx, [rbp-16]` | `e170` | 0 | `e180` | RBX = 0xc8 (read, stack unchanged) |
| 9 | `nop` | `e170` | 0 | `e180` | (inspection point) |
| 10 | `mov rsp, rbp` | `e180` | **+16** | `e180` | Locals released in one step |
| 11 | `pop rbp` | `e188` | **+8** | **0** | Caller's RBP (`0`) restored |
| 12 | `ret` | `e190` | **+8** | 0 | Popped `0x401005` into RIP, back in `_start` |
| 13 | `mov rax, 60` | `e190` | 0 | 0 | none |
| 14 | `xor rdi, rdi` | `e190` | 0 | 0 | none |
| 15 | `syscall` | n/a | n/a | n/a | exit(0) |

### The stack frame at the `nop` (deepest point)

```
 higher addresses
 0x7fffffffe198 | 0x00007fffffffe402 |  argv[0]
 0x7fffffffe190 | 0x0000000000000001 |  argc                 <- RSP before call / after ret
 0x7fffffffe188 | 0x0000000000401005 |  return address       (pushed by call)
 0x7fffffffe180 | 0x0000000000000000 |  saved RBP            <- RBP
 0x7fffffffe178 | 0x0000000000000064 |  [rbp-8]  local = 100
 0x7fffffffe170 | 0x00000000000000c8 |  [rbp-16] local = 200  <- RSP
 lower addresses
```

GDB output at the `nop`:

```
rsp=0x00007fffffffe170  rbp=0x00007fffffffe180  rax=0x64  rbx=0xc8
0x7fffffffe170: 0x00000000000000c8  0x0000000000000064
0x7fffffffe180: 0x0000000000000000  0x0000000000401005
```

This one memory dump shows the whole frame: both locals, the saved RBP, and the return address.

## Observations from running it

1. **`call` is a push plus a jump.** Right after `call work`, RSP dropped by 8 and `[RSP]` held `0x401005`, which is the address of `mov eax, 0x3c` in `_start` (confirmed with `objdump`).
2. **`ret` is a pop into RIP.** After `ret`, RIP = `0x401005` and RSP rose back to `e190`.
3. **The epilogue undoes the prologue in reverse order.** The prologue did `push rbp`, `mov rbp, rsp`, `sub rsp, 16`. The epilogue did `mov rsp, rbp` (undoes the `sub`), then `pop rbp` (undoes the `push`), then `ret` (undoes the `call`). `mov rsp, rbp` followed by `pop rbp` is exactly what the single `leave` instruction does.
4. **Locals are freed by moving RSP, not by clearing memory.** After returning to `_start`, I examined memory below RSP (`x/4gx $rsp-32`). The old frame was still intact: `0xc8`, `0x64`, `0x0`, `0x401005`. That memory is now garbage and will be overwritten by the next call.
5. **New stack space is not guaranteed to be zero.** The two locals read as `0` before they were written, but only because this region had never been used. In a larger program `sub rsp, 16` gives you whatever was left there before, so locals must be initialised.
6. **Stack alignment.** At entry RSP was `...e190`, a multiple of 16. `call` made it `...e188`, and `push rbp` brought it back to a 16-byte boundary (`...e180`). `sub rsp, 16` kept it aligned. The System V ABI requires this alignment before calling library functions.
7. **The stack is balanced.** RSP and RBP finish with exactly the values they started with (`e190` and `0`).
8. The exit code is 0 because RDI was zeroed. The 100 and 200 left in RAX and RBX are not the program's result. RAX is overwritten with 60 for the exit syscall.

## Summary

`call` pushes the return address, and `work`'s prologue saves RBP and carves out 16 bytes for locals. The function reads and writes those locals through RBP-relative addresses, and the epilogue plus `ret` unwind everything in reverse order. RSP goes `e190 → e188 → e180 → e170` on the way in and `e170 → e180 → e188 → e190` on the way out.
