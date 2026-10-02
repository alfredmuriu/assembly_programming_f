# ADD / ADC and the EFLAGS register

This folder looks at how `add` (and `adc`) change the status flags. I assembled
and stepped through each program in GDB, then checked every flag against the
arithmetic done by hand.

## How I ran the programs

```bash
nasm -f elf32 -g -F dwarf add1.asm -o add1.o
ld -m elf_i386 add1.o -o add1
gdb ./add1
```

Inside GDB:

```
(gdb) break _start
(gdb) run
(gdb) stepi                     # execute one instruction
(gdb) info registers eax eflags # inspect result and flags
```

## Flags reference

| Flag | Bit | Meaning after ADD |
|------|-----|-------------------|
| CF – Carry | 0 | 1 if there was a carry **out of the most significant bit** (unsigned result too large for the register) |
| PF – Parity | 2 | 1 if the **low byte** of the result has an **even** number of 1 bits |
| AF – Auxiliary carry | 4 | 1 if there was a carry out of **bit 3** (low nibble) into bit 4 |
| ZF – Zero | 6 | 1 if the result is exactly 0 |
| SF – Sign | 7 | Copy of the most significant bit of the result (1 = negative if read as signed) |
| OF – Overflow | 11 | 1 if the **signed** result does not fit, e.g. positive + positive gives a negative |

`IF` (bit 9) shows up in every GDB output. It is the interrupt-enable flag set
by the operating system, and arithmetic never changes it. Bit 1 is reserved and
is always 1, so the "empty" EFLAGS value is `0x202`.

> **Note:** every program ends with `xor ebx, ebx`, which sets ZF and PF
> (`0x246`). Those flags come from the exit code, **not** from the addition, so
> I read the flags straight after the arithmetic instruction.

---

## Program 1: `add1.asm` (8-bit, signed overflow)

```asm
mov al, [num1]   ; AL = 120 (0x78)
add al, [num2]   ; AL = 120 + 10
```

```
  0111 1000   (120)
+ 0000 1010   ( 10)
-----------
  1000 0010   (0x82 = 130 unsigned / -126 signed)
```

**GDB:** `AL = 0x82`, `eflags = 0xa96  [ PF AF SF IF OF ]`

Decoding `0xa96` = `1010 1001 0110`b gives bits 1, 2, 4, 7, 9 and 11:
reserved, PF, AF, SF, IF, OF.

| Flag | Status | Why |
|------|--------|-----|
| CF | **cleared** | 130 fits in 8 bits (max 255), so nothing carried out of bit 7. |
| OF | **set** | Both operands are positive in signed terms (+120, +10), but the result has bit 7 = 1, i.e. **-126**. The true answer +130 is outside the signed 8-bit range (-128 to +127), so signed overflow happened. |
| SF | **set** | Bit 7 of `1000 0010` is 1. |
| ZF | **cleared** | The result is not zero. |
| AF | **set** | Low nibbles: `1000` (8) + `1010` (10) = 18, which is more than 15, so a carry went from bit 3 into bit 4. |
| PF | **set** | Low byte `1000 0010` has two 1 bits, which is even. |

The key point is that **CF and OF disagree**. Read as unsigned, the answer (130)
is correct, so CF = 0. Read as signed, it is wrong (-126), so OF = 1. The CPU
works out both because it doesn't know which interpretation the programmer
intends.

---

## Program 2: `add2.asm` (16-bit, no flags set)

```asm
mov ax, [num1]   ; AX = 32000 (0x7D00)
add ax, [num2]   ; AX = 32000 + 500
```

```
  0x7D00  (32000)
+ 0x01F4  (  500)
--------
  0x7EF4  (32500)
```

**GDB:** `AX = 0x7ef4 (32500)`, `eflags = 0x202  [ IF ]`

| Flag | Status | Why |
|------|--------|-----|
| CF | **cleared** | 32500 is less than 65535, so there was no carry out of bit 15. |
| OF | **cleared** | Positive + positive = positive. 32500 is still at most +32767 (the signed 16-bit maximum), so the sign bit did not flip. This one is close: adding 268 more would have set OF. |
| SF | **cleared** | Bit 15 of `0x7EF4` (`0111 1110 ...`) is 0. |
| ZF | **cleared** | The result is not zero. |
| AF | **cleared** | Low nibbles: 0x0 + 0x4 = 0x4, no carry out of bit 3. |
| PF | **cleared** | PF only looks at the low byte, `0xF4` = `1111 0100`, which has five 1 bits (odd). |

Every arithmetic flag is 0 here. That is the normal result of an addition that
fits comfortably in both signed and unsigned ranges.

---

## Program 3: `add3.asm` (16-bit wrap-around + ADC)

```asm
mov ax, [num1]   ; AX = 0xFFFF (65535)
add ax, [num2]   ; AX = 0xFFFF + 1
adc ax, 0        ; AX = AX + 0 + CF
```

### After `add ax, [num2]`

```
  1111 1111 1111 1111  (0xFFFF)
+ 0000 0000 0000 0001  (0x0001)
---------------------
1 0000 0000 0000 0000   -> 17th bit is lost, AX = 0x0000
```

**GDB:** `AX = 0x0000`, `eflags = 0x257  [ CF PF AF ZF IF ]`

| Flag | Status | Why |
|------|--------|-----|
| CF | **set** | The true sum is 65536, which needs 17 bits. The carry out of bit 15 goes into CF. |
| ZF | **set** | The 16 bits left in AX are all 0. |
| OF | **cleared** | Read as signed, this is -1 + 1 = 0, which is correct, so there is no signed overflow. This is the opposite of add1: here CF = 1 but OF = 0. |
| SF | **cleared** | Bit 15 of `0x0000` is 0. |
| AF | **set** | Low nibble 0xF + 0x1 = 0x10, so a carry went out of bit 3. |
| PF | **set** | Low byte `0x00` has zero 1 bits, and zero counts as even. |

### After `adc ax, 0`

`adc` adds the operand **plus the current carry flag**: AX = 0 + 0 + CF(1) = **1**.

**GDB:** `AX = 0x0001`, `eflags = 0x202  [ IF ]`

| Flag | Status | Why |
|------|--------|-----|
| CF | **cleared** | 0 + 0 + 1 = 1, no carry out of bit 15. The carry from the previous `add` has been "used up". |
| ZF | **cleared** | The result is 1, not 0. |
| OF / SF | **cleared** | 1 is a small positive number and its bit 15 is 0. |
| AF | **cleared** | Low nibble 0 + 0 + 1 = 1, no carry out of bit 3. |
| PF | **cleared** | `0x01` has one 1 bit (odd). |

This is why `adc` exists: the carry that fell off the top of AX was brought
back in. This is how multi-word numbers wider than a register are added, by
using `add` on the low words and then `adc` on the high words.

---

## `adc4.asm`

This file is empty in the original repository, so there was nothing to
assemble or run.

## Summary

| Program | Operation | Result | CF | OF | SF | ZF | AF | PF |
|---------|-----------|--------|----|----|----|----|----|----|
| add1 | 120 + 10 (8-bit) | 0x82 | 0 | **1** | **1** | 0 | **1** | **1** |
| add2 | 32000 + 500 (16-bit) | 0x7EF4 | 0 | 0 | 0 | 0 | 0 | 0 |
| add3 | 0xFFFF + 1 | 0x0000 | **1** | 0 | 0 | **1** | **1** | **1** |
| add3 | `adc ax, 0` | 0x0001 | 0 | 0 | 0 | 0 | 0 | 0 |

- **CF** is about the **unsigned** result not fitting (add3).
- **OF** is about the **signed** result not fitting (add1).
- They are independent, and these programs show each one set without the other.
