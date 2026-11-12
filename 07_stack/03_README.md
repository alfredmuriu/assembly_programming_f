# 03.asm: `CALL` / `RET` and the Return Address (64-bit)

## What the program does

```nasm
_start:
    mov rax, 10
    call calculate

after_call:
    ; RAX should now be 15
    nop

    mov rax, 60
    xor rdi, rdi
    syscall

calculate:
    add rax, 5
    ret
```

`_start` puts 10 in RAX and calls `calculate`, which adds 5 and returns. Execution continues at the `after_call` label with RAX = 15, and the program then exits with status 0.

`calculate` never pushes or pops anything itself. The only stack activity comes from `call` and `ret`, which makes this the clearest program for seeing how the **return address** works.

## Build and run

```bash
nasm -f elf64 -g -F dwarf 07_stack/03.asm -o build/07_stack/03.o
ld build/07_stack/03.o -o build/07_stack/03
./build/07_stack/03 ; echo $?
```

Result: there is no output, and the exit status is `0`. The stack was traced in GDB (see [`logs/07_stack_trace_2026-11-12.txt`](../logs/07_stack_trace_2026-11-12.txt)).

Addresses from `objdump`:

```
401000 <_start>:      mov  eax,0xa
401005:               call 401015 <calculate>     (5 bytes long)
40100a <after_call>:  nop                         <- next instruction after the call
...
401015 <calculate>:   add  rax,0x5
401019:               ret
```

## How the stack is used in this program

* **`call calculate`** does two things:
  1. it pushes the address of the next instruction (`0x40100a`) onto the stack, so `RSP = RSP - 8` and `[RSP] = 0x40100a`;
  2. it jumps to `calculate`.
* **`ret`** does the reverse: it pops the value on top of the stack into RIP, so `RIP = [RSP]` and `RSP = RSP + 8`.

The argument (10) and the result (15) travel in **RAX**, not on the stack. The stack is used only to remember where to come back to.

## Step-by-step trace (values observed in GDB)

| # | Instruction | RSP after | Change | Top of stack | RIP after | RAX |
|---|-------------|-----------|--------|--------------|-----------|-----|
| 0 | *(entry)* | `0x7fffffffe190` | n/a | `1` (argc) | `0x401000` | 0 |
| 1 | `mov rax, 10` | `0x7fffffffe190` | 0 | `1` | `0x401005` | 0xa |
| 2 | `call calculate` | `0x7fffffffe188` | **-8** | **`0x40100a`** | `0x401015` | 0xa |
| 3 | `add rax, 5` | `0x7fffffffe188` | 0 | `0x40100a` | `0x401019` | **0xf** |
| 4 | `ret` | `0x7fffffffe190` | **+8** | `1` | **`0x40100a`** | 0xf |
| 5 | `nop` (`after_call`) | `0x7fffffffe190` | 0 | `1` | `0x40100b` | 0xf |
| 6 | `mov rax, 60` | `0x7fffffffe190` | 0 | `1` | `0x401010` | 0x3c |
| 7 | `xor rdi, rdi` | `0x7fffffffe190` | 0 | `1` | `0x401013` | 0x3c |
| 8 | `syscall` | n/a | n/a | n/a | n/a | exit(0) |

### Stack picture while inside `calculate`

```
 higher addresses
 0x7fffffffe198 | 0x00007fffffffe402 |  argv[0]
 0x7fffffffe190 | 0x0000000000000001 |  argc                         <- RSP before call / after ret
 0x7fffffffe188 | 0x000000000040100a |  return address = after_call  <- RSP inside calculate
 lower addresses
```

GDB output with a breakpoint on `calculate`:

```
(gdb) x/gx $rsp
0x7fffffffe188: 0x000000000040100a
(gdb) info symbol *(long*)$rsp
after_call in section .text
(gdb) p &after_call
$1 = 0x40100a <after_call>
```

## Observations from running it

1. **The pushed return address is exactly the `after_call` label.** `call` is 5 bytes long and starts at `0x401005`, so the next instruction is at `0x401005 + 5 = 0x40100a`. GDB confirmed that the value on the stack (`0x40100a`) is the address of `after_call`.
2. **`ret` blindly trusts the top of the stack.** It doesn't "remember" the caller; it just pops whatever RSP points at. If `calculate` had done a `push` without a matching `pop`, `ret` would have jumped to that pushed value and the program would most likely crash. That's why every function must leave RSP exactly where it found it.
3. **A leaf function needs no stack frame.** `calculate` calls nothing and uses no locals, so it skips `push rbp` and `mov rbp, rsp`. RBP stayed 0 for the whole run, and the only stack use was the 8-byte return address.
4. **RAX carried the data.** RAX went `0x0 → 0xa → 0xf`, and when execution reached `after_call` it held 15, as the comment in the source says.
5. **The stack is balanced.** RSP went `e190 → e188 → e190`.
6. The exit status is 0, not 15, because the exit code is taken from **RDI** (zeroed by `xor rdi, rdi`), while RAX is overwritten with 60 (`sys_exit`). Adding `mov rdi, rax` before the `mov rax, 60` would make `echo $?` print 15.

## Summary

The stack is used only to remember where to return. `call` pushes `0x40100a` (the `after_call` address) and lowers RSP by 8. `calculate` changes only RAX, and `ret` pops `0x40100a` back into RIP, which raises RSP by 8 and resumes execution right after the call with RAX = 15.
