# 01.asm: Basic `PUSH` and `POP` (64-bit)

## What the program does

```nasm
_start:
    mov rax, 100
    mov rbx, 200

    push rax
    push rbx
    nop

    pop rcx
    pop rdx
    nop

    mov rax, 60
    xor rdi, rdi
    syscall
```

The program puts 100 in RAX and 200 in RBX. It pushes both values onto the stack, then pops them back into **different** registers (RCX and RDX), and exits with status 0. It shows the **Last-In, First-Out (LIFO)** rule: the value pushed last (200) is the first one popped.

## Build and run

```bash
nasm -f elf64 -g -F dwarf 07_stack/01.asm -o build/07_stack/01.o
ld build/07_stack/01.o -o build/07_stack/01
./build/07_stack/01 ; echo $?
```

Result: there is no console output, and the exit status is `0`. Because the program prints nothing, I watched the stack by single-stepping it in GDB (full trace in [`logs/07_stack_trace_2026-11-12.txt`](../logs/07_stack_trace_2026-11-12.txt)).

## How the stack works (the rules this program uses)

* On x86-64 the stack **grows downward**, toward lower addresses.
* **RSP** always holds the address of the item currently on top of the stack.
* `push X` is equivalent to `RSP = RSP - 8`, then `[RSP] = X`.
* `pop Y` is equivalent to `Y = [RSP]`, then `RSP = RSP + 8`.
* In 64-bit mode every push and pop moves **8 bytes**, however small the value is.

## Where the stack starts

Before the first instruction runs, the Linux kernel has already set up the stack. In GDB it looked like this:

```
RSP = 0x7fffffffe190
0x7fffffffe190:  0x0000000000000001   <- argc (1 = just the program name)
0x7fffffffe198:  0x00007fffffffe402   <- argv[0] (pointer to the program path)
```

The program never uses `argc` or `argv`, but they are the "floor" our pushes are stacked on.

## Step-by-step trace (values observed in GDB)

RSP is shown as the last 3 hex digits of `0x7fffffffeXXX`. The stack contents column lists the top of the stack first.

| # | Instruction | RSP after | Change | Stack (top first) | Registers changed |
|---|-------------|-----------|--------|-------------------|-------------------|
| 0 | *(entry)* | `e190` | n/a | `1 (argc)`, `argv[0]` | all 0 |
| 1 | `mov rax, 100` | `e190` | 0 | unchanged | RAX = 0x64 |
| 2 | `mov rbx, 200` | `e190` | 0 | unchanged | RBX = 0xc8 |
| 3 | `push rax` | `e188` | **-8** | `0x64`, `1`, … | none |
| 4 | `push rbx` | `e180` | **-8** | `0xc8`, `0x64`, `1`, … | none |
| 5 | `nop` | `e180` | 0 | `0xc8`, `0x64`, `1`, … | none |
| 6 | `pop rcx` | `e188` | **+8** | `0x64`, `1`, … | RCX = 0xc8 (200) |
| 7 | `pop rdx` | `e190` | **+8** | `1`, … | RDX = 0x64 (100) |
| 8 | `nop` | `e190` | 0 | `1`, … | none |
| 9 | `mov rax, 60` | `e190` | 0 | unchanged | RAX = 60 (`sys_exit`) |
| 10 | `xor rdi, rdi` | `e190` | 0 | unchanged | RDI = 0 (exit code) |
| 11 | `syscall` | n/a | n/a | n/a | process exits, status 0 |

### Stack picture at the first `nop` (after both pushes)

```
 higher addresses
 0x7fffffffe198 | 0x00007fffffffe402 |  argv[0]
 0x7fffffffe190 | 0x0000000000000001 |  argc            <- RSP at entry / at exit
 0x7fffffffe188 | 0x0000000000000064 |  pushed RAX (100)
 0x7fffffffe180 | 0x00000000000000c8 |  pushed RBX (200) <- RSP here (top)
 lower addresses (stack grows this way)
```

GDB output at this point:

```
rsp=0x00007fffffffe180  rax=0x64  rbx=0xc8  rcx=0  rdx=0
0x7fffffffe180: 0x00000000000000c8  0x0000000000000064
```

### Stack picture at the second `nop` (after both pops)

```
 0x7fffffffe190 | 0x0000000000000001 |  argc   <- RSP (back where it started)
 0x7fffffffe188 | 0x0000000000000064 |  (stale: no longer part of the stack)
 0x7fffffffe180 | 0x00000000000000c8 |  (stale: no longer part of the stack)
```

## Observations from running it

1. **RSP moved by exactly 8 on every push and pop.** The values 100 and 200 fit in one byte, but each still took a full 8-byte slot.
2. **LIFO reverses the order.** RBX was pushed last, so its value (200) came out first and went into RCX. RAX's value (100) came out second and went into RDX. After the pops, RCX = 200 and RDX = 100, which is the reverse of the push order.
3. **`push` copies, it does not move.** RAX and RBX still held 100 and 200 after the pushes.
4. **`pop` does not erase memory.** After both pops I ran `x/2gx $rsp-16` and the old values were still there: `0x7fffffffe180: 0xc8 0x64`. Only RSP moved. Anything below RSP counts as free space that the next push will overwrite.
5. **The stack is balanced.** There were 2 pushes and 2 pops, so RSP ended at `0x7fffffffe190`, exactly where it started. A program or function must leave RSP where it found it.
6. **Instruction encoding detail:** `objdump` shows that NASM assembled `mov rax, 100` as `mov eax, 0x64`. Writing to a 32-bit register zero-fills the upper 32 bits, so the result is the same and the instruction is shorter.
7. **Absolute addresses are not fixed.** GDB disables address randomisation (ASLR), which is why the trace always shows `e190`. When I passed an extra argument (`hello`), the entry RSP became `0x7fffffffe180` and `argc` became 2. Outside GDB the stack is placed at a random address on every run. The **relative** movement (-8, -8, +8, +8) never changes.

## Summary

Execution starts with RSP pointing at `argc`. Two pushes lower RSP by 16 bytes and store 200 on top of 100. Two pops read those values back in reverse order into RCX and RDX and raise RSP by 16 bytes, so the stack is back to its original state when the exit syscall runs.
