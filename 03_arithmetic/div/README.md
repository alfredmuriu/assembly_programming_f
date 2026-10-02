# DIV and the EFLAGS register

This folder looks at unsigned division with `div`. Each program was assembled,
stepped through in GDB, and the quotient, remainder and EFLAGS were recorded
straight after the `div` instruction.

## How I ran the programs

```bash
nasm -f elf32 -g -F dwarf div1.asm -o div1.o
ld -m elf_i386 div1.o -o div1
gdb ./div1
```

```
(gdb) break _start
(gdb) run
(gdb) stepi
(gdb) info registers eax ebx edx eflags
```

## How DIV works

The dividend is implicit and **twice the width** of the divisor:

| Divisor size | Dividend | Quotient | Remainder |
|--------------|----------|----------|-----------|
| 8-bit  | `AX`      | `AL`  | `AH`  |
| 16-bit | `DX:AX`   | `AX`  | `DX`  |
| 32-bit | `EDX:EAX` | `EAX` | `EDX` |

## Which flags DIV defines

**None.** According to the Intel manual, **CF, OF, SF, ZF, AF and PF are all
undefined after `div`**. Division doesn't report problems through flags.
Instead:

- If the divisor is **0**, or
- if the quotient is **too big** for the destination (e.g. `AX = 1000`,
  `div bl` with `BL = 2` gives 500, which doesn't fit in AL),

the CPU raises a **divide error exception (#DE)**, and on Linux the program is
killed with `SIGFPE` ("Floating point exception"). This is why the high half
(AH / DX / EDX) has to be set up correctly **before** a `div`.

The GDB results below show that whatever flags appear afterwards carry no
meaning about the result.

`IF` (0x200) and reserved bit 1 (0x2) are always on. The final
`xor ebx, ebx` sets ZF/PF (`0x246`) as part of the exit code, not the division.

---

## Program 1: `div1.asm` (8-bit divisor, 100 ÷ 7)

```asm
mov ax, [dividend]  ; AX = 100
mov bl, [divisor]   ; BL = 7
div bl              ; AL = quotient, AH = remainder
```

100 ÷ 7 = 14 remainder 2 → **AL = 0x0E (14)**, **AH = 0x02 (2)**

**GDB:** `EAX = 0x20e (526)`, `eflags = 0x212  [ AF IF ]`

GDB shows 526 because it reads the whole of AX = `0x020E` as one number. Split
into bytes, it is **AH = 0x02** (remainder) and **AL = 0x0E** (quotient).

| Flag | Status | Why |
|------|--------|-----|
| CF, OF | cleared (undefined) | `div` doesn't report overflow through flags. An oversized quotient raises #DE instead. |
| AF | set (undefined) | There is no nibble carry in a division. AF = 1 is just left over from how the CPU computes the divide, not a meaningful result. |
| ZF | cleared (undefined) | Not based on the quotient or the remainder. |
| SF, PF | cleared (undefined) | Not computed. AL = 0x0E has three 1 bits, so PF would be 0 anyway, but that is a coincidence (see div3). |

---

## Program 2: `div2.asm` (16-bit divisor, 50000 ÷ 300)

```asm
mov ax, [dividend]  ; AX = 50000 (0xC350)
mov dx, [highpart]  ; DX = 0  -> dividend DX:AX = 50000
mov bx, [divisor]   ; BX = 300
div bx              ; AX = quotient, DX = remainder
```

50000 ÷ 300 = 166 remainder 200 → **AX = 0x00A6 (166)**, **DX = 0x00C8 (200)**

**GDB:** `AX = 0xa6 (166)`, `DX = 0xc8 (200)`, `eflags = 0x212  [ AF IF ]`

| Flag | Status | Why |
|------|--------|-----|
| CF, OF | cleared (undefined) | No flag-based overflow reporting. The quotient 166 fits in AX, so no #DE occurred. |
| AF | set (undefined) | Same leftover value as div1, unrelated to the numbers involved. |
| ZF, SF, PF | cleared (undefined) | Not computed. The low byte of AX, `0xA6` = `1010 0110`, has four 1 bits (even), so a real PF would be **1**, but GDB shows PF = 0. |

`DX` had to be zeroed first (`mov dx, [highpart]`). If DX had held garbage, the
CPU would have divided a much bigger 32-bit number and could have raised #DE.

---

## Program 3: `div3.asm` (32-bit divisor, 300 000 000 ÷ 1000)

```asm
mov eax, [dividend]  ; EAX = 300000000
mov edx, [highpart]  ; EDX = 0
mov ebx, [divisor]   ; EBX = 1000
div ebx              ; EAX = quotient, EDX = remainder
```

300 000 000 ÷ 1000 = 300 000 remainder 0 → **EAX = 0x000493E0**, **EDX = 0**

**GDB:** `EAX = 0x493e0 (300000)`, `EDX = 0x0`, `eflags = 0x212  [ AF IF ]`

| Flag | Status | Why |
|------|--------|-----|
| ZF | **cleared** (undefined) | **This shows clearly that the flags are undefined.** The remainder is exactly **0**, but ZF is **not** set. `div` doesn't test either result for zero, so you have to use `test edx, edx` or `cmp edx, 0` yourself. |
| CF, OF | cleared (undefined) | The quotient fits in EAX, so no #DE. Overflow is never reported in CF/OF. |
| AF | set (undefined) | Same leftover value as the other two programs. |
| SF, PF | cleared (undefined) | Not computed. The low byte `0xE0` has three 1 bits, so PF happens to be correct here, but only by coincidence. |

---

## Summary

| Program | Dividend ÷ Divisor | Quotient | Remainder | EFLAGS after `div` |
|---------|--------------------|----------|-----------|--------------------|
| div1 | 100 ÷ 7 (AX / BL) | AL = 14 | AH = 2 | `0x212 [AF IF]` |
| div2 | 50000 ÷ 300 (DX:AX / BX) | AX = 166 | DX = 200 | `0x212 [AF IF]` |
| div3 | 300000000 ÷ 1000 (EDX:EAX / EBX) | EAX = 300000 | EDX = 0 | `0x212 [AF IF]` |

- All three programs produced the **same** EFLAGS (`0x212`), even though the
  results were very different (one remainder was 0, one quotient was small,
  one was large). The flags clearly don't depend on the result.
- After `div`, **every arithmetic flag is undefined**. Errors (divide by zero,
  quotient too large) are reported with a **#DE exception**, not a flag.
- To act on the result, test the registers directly, e.g.
  `test edx, edx` / `jz` to check for a zero remainder.
