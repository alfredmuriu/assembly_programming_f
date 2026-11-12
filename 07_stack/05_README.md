# 05.asm: Saving and Restoring Registers with the Stack (64-bit)

## What the program does

```nasm
_start:
    mov rax, 0x1122334455667788
    mov rbx, 0x1234

    push rax
    push rbx
    nop

    pop rbx
    pop rax

    mov rax, 60
    xor rdi, rdi
    syscall
```

The program loads a full 64-bit value into RAX and a small value into RBX, then pushes both. It pops them back **in reverse order** (`pop rbx` first, then `pop rax`), so each register gets back its own original value. This is the standard save-and-restore pattern. The program then exits with status 0.

The distinctive values (`11 22 33 44 55 66 77 88`) make it easy to see the size of each slot and the **byte order** in memory.

## Build and run

```bash
nasm -f elf64 -g -F dwarf 07_stack/05.asm -o build/07_stack/05.o
ld build/07_stack/05.o -o build/07_stack/05
./build/07_stack/05 ; echo $?
```

Result: there is no output, and the exit status is `0`. The stack was traced in GDB (see [`logs/07_stack_trace_2026-11-12.txt`](../logs/07_stack_trace_2026-11-12.txt)).

## How the stack is used in this program

The stack is used as **temporary storage** for two registers. Each `push` lowers RSP by 8 and stores 8 bytes. Because the stack is LIFO, the registers must be popped in the **opposite** order to the pushes to land back in the same registers:

```
push rax   ─┐         ┌─ pop rax
push rbx   ─┼─┐     ┌─┼─ pop rbx
            │ └─────┘ │
            └─────────┘
```

## Step-by-step trace (values observed in GDB)

RSP is shown as the last 3 hex digits of `0x7fffffffeXXX`.

| # | Instruction | RSP after | Change | Stack (top first) | RAX | RBX |
|---|-------------|-----------|--------|-------------------|-----|-----|
| 0 | *(entry)* | `e190` | n/a | `1`, … | 0 | 0 |
| 1 | `mov rax, 0x1122334455667788` | `e190` | 0 | unchanged | `0x1122334455667788` | 0 |
| 2 | `mov rbx, 0x1234` | `e190` | 0 | unchanged | `0x1122334455667788` | `0x1234` |
| 3 | `push rax` | `e188` | **-8** | `0x1122334455667788`, `1`, … | same | same |
| 4 | `push rbx` | `e180` | **-8** | `0x1234`, `0x1122334455667788`, `1`, … | same | same |
| 5 | `nop` | `e180` | 0 | same | same | same |
| 6 | `pop rbx` | `e188` | **+8** | `0x1122334455667788`, `1`, … | same | `0x1234` |
| 7 | `pop rax` | `e190` | **+8** | `1`, … | `0x1122334455667788` | `0x1234` |
| 8 | `mov rax, 60` | `e190` | 0 | unchanged | 60 | `0x1234` |
| 9 | `xor rdi, rdi` | `e190` | 0 | unchanged | 60 | `0x1234` |
| 10 | `syscall` | n/a | n/a | n/a | exit(0) | |

### Stack picture at the `nop`

```
 higher addresses
 0x7fffffffe190 | 0x0000000000000001 |  argc            <- RSP at entry / exit
 0x7fffffffe188 | 0x1122334455667788 |  pushed RAX
 0x7fffffffe180 | 0x0000000000001234 |  pushed RBX      <- RSP (top)
 lower addresses
```

### The same memory, byte by byte (`x/16xb $rsp` at the `nop`)

```
0x7fffffffe180: 0x34 0x12 0x00 0x00 0x00 0x00 0x00 0x00   <- RBX = 0x1234
0x7fffffffe188: 0x88 0x77 0x66 0x55 0x44 0x33 0x22 0x11   <- RAX = 0x1122334455667788
```

## Observations from running it

1. **Every push uses a full 8 bytes.** `0x1234` only needs 2 bytes, but `push rbx` still lowered RSP by 8 and stored `34 12 00 00 00 00 00 00`, with the unused upper bytes set to zero. The large value used all 8 bytes.
2. **x86 is little-endian.** In memory, the lowest byte of each value comes first. `0x1122334455667788` is stored as `88 77 66 55 44 33 22 11`, which reads "backwards" in the byte dump. When GDB shows the slot as a whole 8-byte word (`x/gx`), it reassembles the bytes and the value reads normally.
3. **Popping in reverse order restores the registers correctly.** After `pop rbx` and `pop rax`, RAX = `0x1122334455667788` and RBX = `0x1234`, exactly their original values. Compare this with [01.asm](01_README.md), where popping into different registers (`rcx` then `rdx`) moved the values into new registers in reverse order.
4. **What would go wrong with the wrong order:** if the code did `pop rax` first, RAX would receive `0x1234` (the top of the stack) and RBX would receive `0x1122334455667788`. The two values would be swapped.
5. **Instruction encoding:** `objdump` shows `movabs rax, 0x1122334455667788` (a 10-byte instruction carrying a full 64-bit immediate) and `mov ebx, 0x1234` (NASM used the shorter 32-bit form because the value fits).
6. **The stack is balanced.** RSP went `e190 → e188 → e180 → e188 → e190`. The restored RAX is overwritten with 60 straight away for the exit call, so this program only demonstrates the restore. In a real function you would save registers this way at the start and restore them before `ret`.

## Summary

Two registers are saved on the stack (RSP -16) and restored from it in reverse order (RSP +16). The byte dump shows that each value occupies an 8-byte slot stored little-endian. Popping in the opposite order to the pushes gives every register back its own original value.
