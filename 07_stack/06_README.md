# 06.asm: Saving and Restoring Registers with the Stack (32-bit)

## What the program does

```nasm
BITS 32
_start:
    mov eax, 0x11223344
    mov ebx, 0x1234

    push eax
    push ebx
    nop

    pop ebx
    pop eax

    mov eax, 1
    xor ebx, ebx
    int 0x80
```

This is the **32-bit version of [05.asm](05_README.md)**. It loads a full 32-bit value into EAX and a small value into EBX, pushes both, and pops them back in reverse order so each register is restored. It then exits through `int 0x80` (EAX = 1 is `sys_exit`, and EBX = 0 is the exit code).

## Build and run

```bash
nasm -f elf32 -g -F dwarf 07_stack/06.asm -o build/07_stack/06.o
ld -m elf_i386 build/07_stack/06.o -o build/07_stack/06
./build/07_stack/06 ; echo $?
```

Result: there is no output, and the exit status is `0`. The stack was traced in GDB (see [`logs/07_stack_trace_2026-11-12.txt`](../logs/07_stack_trace_2026-11-12.txt)).

## How the stack is used in this program

In 32-bit mode the stack pointer is **ESP**, and every `push` and `pop` moves **4 bytes**:

* `push X` is equivalent to `ESP = ESP - 4`, then `[ESP] = X` (32 bits).
* `pop Y` is equivalent to `Y = [ESP]`, then `ESP = ESP + 4`.

As in 05.asm, the stack is temporary storage for two registers, and LIFO order means they are popped in the reverse of the order they were pushed.

## Step-by-step trace (values observed in GDB)

ESP is shown as the last 3 hex digits of `0xffffdXXX`.

| # | Instruction | ESP after | Change | Stack (top first) | EAX | EBX |
|---|-------------|-----------|--------|-------------------|-----|-----|
| 0 | *(entry)* | `2b0` | n/a | `1`, `0xffffd402`, … | 0 | 0 |
| 1 | `mov eax, 0x11223344` | `2b0` | 0 | unchanged | `0x11223344` | 0 |
| 2 | `mov ebx, 0x1234` | `2b0` | 0 | unchanged | `0x11223344` | `0x1234` |
| 3 | `push eax` | `2ac` | **-4** | `0x11223344`, `1`, … | same | same |
| 4 | `push ebx` | `2a8` | **-4** | `0x1234`, `0x11223344`, `1`, … | same | same |
| 5 | `nop` | `2a8` | 0 | same | same | same |
| 6 | `pop ebx` | `2ac` | **+4** | `0x11223344`, `1`, … | same | `0x1234` |
| 7 | `pop eax` | `2b0` | **+4** | `1`, … | `0x11223344` | `0x1234` |
| 8 | `mov eax, 1` | `2b0` | 0 | unchanged | 1 | `0x1234` |
| 9 | `xor ebx, ebx` | `2b0` | 0 | unchanged | 1 | 0 |
| 10 | `int 0x80` | n/a | n/a | n/a | exit(0) | |

### Stack picture at the `nop`

```
 higher addresses
 0xffffd2b4 | 0xffffd402 |  argv[0]
 0xffffd2b0 | 0x00000001 |  argc          <- ESP at entry / exit
 0xffffd2ac | 0x11223344 |  pushed EAX
 0xffffd2a8 | 0x00001234 |  pushed EBX    <- ESP (top)
 lower addresses
```

GDB output at the `nop`:

```
esp=0xffffd2a8  eax=0x11223344  ebx=0x1234
0xffffd2a8: 0x00001234  0x11223344  0x00000001  0xffffd402
```

### The same memory, byte by byte (`x/8xb $esp` at the `nop`)

```
0xffffd2a8: 0x34 0x12 0x00 0x00 | 0x44 0x33 0x22 0x11
             └─ EBX = 0x1234 ─┘   └─ EAX = 0x11223344 ─┘
```

## Comparing with the 64-bit version (05.asm)

| | 05.asm (64-bit) | 06.asm (32-bit) |
|---|---|---|
| Stack pointer | RSP | ESP |
| Bytes per push or pop | 8 | 4 |
| RSP/ESP after two pushes | `e190 → e180` (16 bytes) | `2b0 → 2a8` (8 bytes) |
| `0x1234` stored as | `34 12 00 00 00 00 00 00` | `34 12 00 00` |
| Large value stored as | `88 77 66 55 44 33 22 11` | `44 33 22 11` |
| Exit | `syscall`, RAX = 60 | `int 0x80`, EAX = 1 |

## Observations from running it

1. **Each push or pop moved ESP by exactly 4.** The two pushes used 8 bytes of stack in total, half of what the same pattern used in 05.asm.
2. **Little-endian storage again.** `0x11223344` sits in memory as `44 33 22 11`, and `0x1234` as `34 12 00 00` (padded to 4 bytes, not 8).
3. **The two pushed values are contiguous.** One 8-byte dump at ESP contains both values back to back. In 05.asm the same dump would only reach the first value.
4. **Reverse-order popping restored both registers.** After `pop ebx` and `pop eax`, EBX = `0x1234` and EAX = `0x11223344`, the same as before the pushes.
5. **The stack is balanced.** ESP returned to `0xffffd2b0`, its value at entry. Both restored registers are then overwritten for the exit call: EAX = 1 and EBX = 0. EBX doubles as the exit-code register in 32-bit Linux, which is why it is cleared last.
6. The 32-bit stack lives near `0xffffd2b0`, the top of a 4 GB address space, rather than at `0x7fffffffe…` as in the 64-bit programs.

## Summary

ESP goes `2b0 → 2ac → 2a8` as EAX and EBX are saved, then `2a8 → 2ac → 2b0` as they are restored in reverse order. The program follows the same LIFO save/restore idea as 05.asm, but with 4-byte stack slots because it is 32-bit code.
