# 07_stack: How the Stack Works

Each program in this folder has its own README. Each one traces how the stack (RSP or ESP) changes from the first instruction to the exit system call.

| Program | Mode | Topic | README |
|---------|------|-------|--------|
| `01.asm` | 64-bit | Basic `push` / `pop` and LIFO order | [01_README.md](01_README.md) |
| `02.asm` | 64-bit | `call`, stack frame (`push rbp` / `mov rbp, rsp`), local variables | [02_README.md](02_README.md) |
| `03.asm` | 64-bit | `call` / `ret` and the return address | [03_README.md](03_README.md) |
| `04.asm` | 32-bit | Stack frame and local variables with ESP and EBP | [04_README.md](04_README.md) |
| `05.asm` | 64-bit | Saving and restoring registers, slot size, little-endian | [05_README.md](05_README.md) |
| `06.asm` | 32-bit | Saving and restoring registers with 4-byte slots | [06_README.md](06_README.md) |

## Key rules shared by every program

* The stack grows **downward**, toward lower addresses.
* `push`: decrease the stack pointer, then write the value. `pop`: read the value, then increase the stack pointer.
* The step is **8 bytes** in 64-bit code and **4 bytes** in 32-bit code.
* `call` pushes the return address, and `ret` pops it into the instruction pointer.
* All six programs leave the stack pointer exactly where they found it (the stack is balanced).

## How the programs were built, run and observed

Environment: Ubuntu 24.04 (WSL2) with NASM 2.16.01, GNU ld 2.42, and GDB 15.1.

```bash
# 64-bit (01, 02, 03, 05)
nasm -f elf64 -g -F dwarf 07_stack/01.asm -o build/07_stack/01.o
ld build/07_stack/01.o -o build/07_stack/01

# 32-bit (04, 06)
nasm -f elf32 -g -F dwarf 07_stack/04.asm -o build/07_stack/04.o
ld -m elf_i386 build/07_stack/04.o -o build/07_stack/04

./build/07_stack/01 ; echo $?     # every program exits with status 0
```

None of the programs prints anything, so each one was single-stepped in GDB, printing the registers and the top of the stack after every instruction:

```gdb
starti                 # stop before the first instruction
x/4gx $rsp             # top 4 stack slots (use x/4wx $esp for 32-bit)
stepi                  # execute one instruction, then repeat
```

The complete output is in [`logs/07_stack_trace_2026-11-12.txt`](../logs/07_stack_trace_2026-11-12.txt).
