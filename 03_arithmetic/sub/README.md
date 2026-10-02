# SUB / SBB and the EFLAGS register

This folder looks at how `sub` (and `sbb`) change the status flags. Each program
was assembled, stepped through one instruction at a time in GDB, and checked
against the binary subtraction done by hand.

## How I ran the programs

```bash
nasm -f elf32 -g -F dwarf sub1.asm -o sub1.o
ld -m elf_i386 sub1.o -o sub1
gdb ./sub1
```

```
(gdb) break _start
(gdb) run
(gdb) stepi
(gdb) info registers eax eflags
```

## Flags reference for SUB (`dest - src`)

| Flag | Bit | Meaning after SUB |
|------|-----|-------------------|
| CF – Carry | 0 | 1 if a **borrow** was needed, i.e. `dest < src` when both are read as **unsigned** |
| PF – Parity | 2 | 1 if the **low byte** of the result has an even number of 1 bits |
| AF – Auxiliary | 4 | 1 if the low nibble needed a borrow from bit 4 |
| ZF – Zero | 6 | 1 if the result is 0 (i.e. `dest == src`) |
| SF – Sign | 7 | Copy of the result's most significant bit |
| OF – Overflow | 11 | 1 if the **signed** result is out of range. This can only happen when the operands have **different** signs (e.g. positive − negative giving a negative). |

`IF` (0x200) and reserved bit 1 (0x2) are always on, so `0x202` means "no
arithmetic flags set". The final `xor ebx, ebx` in every program sets ZF and PF
(`0x246`), which belongs to the exit code, not to the subtraction.

---

## Program 1: `sub1.asm` (8-bit, 50 − 80)

```asm
mov al, [num1]   ; AL = 50 (0x32)
sub al, [num2]   ; AL = 50 - 80
```

```
  0011 0010   (50)
- 0101 0000   (80)
-----------
  1110 0010   (0xE2 = 226 unsigned / -30 signed)
```

**GDB:** `AL = 0xe2`, `eflags = 0x287  [ CF PF SF IF ]`

| Flag | Status | Why |
|------|--------|-----|
| CF | **set** | 50 < 80 as unsigned numbers, so a borrow was needed past bit 7. The result 226 is not the real unsigned answer, and CF reports that. |
| OF | **cleared** | Both operands are positive. Positive − positive always lands in −127…+127, so it can't overflow. The signed answer −30 is correct. |
| SF | **set** | Bit 7 of `1110 0010` is 1. Read as signed, the result is negative (−30). |
| ZF | **cleared** | The result is not 0 (50 ≠ 80). |
| AF | **cleared** | Low nibbles: `0010` − `0000` = `0010`, no borrow needed from bit 4. |
| PF | **set** | `1110 0010` has four 1 bits (even). |

---

## Program 2: `sub2.asm` (16-bit, 1000 − 2000)

```asm
mov ax, [num1]   ; AX = 1000 (0x03E8)
sub ax, [num2]   ; AX = 1000 - 2000
```

```
  0x03E8   (1000)
- 0x07D0   (2000)
--------
  0xFC18   (64536 unsigned / -1000 signed)
```

**GDB:** `AX = 0xfc18 (64536)`, `eflags = 0x287  [ CF PF SF IF ]`

| Flag | Status | Why |
|------|--------|-----|
| CF | **set** | 1000 < 2000 unsigned, so a borrow came out of bit 15. |
| OF | **cleared** | Positive − positive, and −1000 fits easily in the signed 16-bit range (−32768…+32767). |
| SF | **set** | Bit 15 of `0xFC18` (`1111 1100 ...`) is 1, so the result is negative. |
| ZF | **cleared** | The result is not 0. |
| AF | **cleared** | Low nibbles 0x8 − 0x0 = 0x8, no borrow. |
| PF | **set** | Only the low byte counts: `0x18` = `0001 1000` has two 1 bits (even). |

GDB prints 64536 because it shows the register as unsigned. The same bit
pattern `0xFC18` is −1000 in two's complement. The CPU doesn't store which
meaning is intended, which is why it sets **CF for the unsigned view** and **SF/OF
for the signed view** at the same time.

---

## Program 3: `sub3.asm` (0 − 1, then SBB)

```asm
mov ax, [num1]   ; AX = 0
sub ax, [num2]   ; AX = 0 - 1
sbb ax, 0        ; AX = AX - 0 - CF
```

### After `sub ax, [num2]`

```
  0000 0000 0000 0000   (0)
- 0000 0000 0000 0001   (1)
---------------------
  1111 1111 1111 1111   (0xFFFF = 65535 unsigned / -1 signed)
```

**GDB:** `AX = 0xffff`, `eflags = 0x297  [ CF PF AF SF IF ]`

| Flag | Status | Why |
|------|--------|-----|
| CF | **set** | 0 < 1 unsigned, so the subtraction borrowed past bit 15 and the result wrapped around to 0xFFFF. |
| OF | **cleared** | 0 − 1 = −1 is a valid signed 16-bit value. |
| SF | **set** | Bit 15 is 1 (the result is −1). |
| ZF | **cleared** | The result is not 0. |
| AF | **set** | Low nibble 0x0 − 0x1 needs a borrow from bit 4. |
| PF | **set** | Low byte `0xFF` has eight 1 bits (even). |

### After `sbb ax, 0`

`sbb` subtracts the operand **and the current carry (borrow) flag**:
AX = 0xFFFF − 0 − CF(1) = **0xFFFE**.

**GDB:** `AX = 0xfffe (65534)`, `eflags = 0x282  [ SF IF ]`

| Flag | Status | Why |
|------|--------|-----|
| CF | **cleared** | 65535 − 1 needs no borrow because the minuend is larger. The borrow from the first `sub` has been consumed. |
| OF | **cleared** | −1 − 1 = −2, which is in range. |
| SF | **set** | `0xFFFE` still has bit 15 = 1 (−2 signed). |
| ZF | **cleared** | The result is not 0. |
| AF | **cleared** | Low nibble 0xF − 0 − 1 = 0xE, no borrow. |
| PF | **cleared** | Low byte `0xFE` = `1111 1110` has seven 1 bits (odd). |

`sbb` is the subtraction counterpart of `adc`. It passes a borrow from the low
word to the high word when subtracting numbers wider than one register.

---

## Summary

| Program | Operation | Result | CF | OF | SF | ZF | AF | PF |
|---------|-----------|--------|----|----|----|----|----|----|
| sub1 | 50 − 80 (8-bit) | 0xE2 | **1** | 0 | **1** | 0 | 0 | **1** |
| sub2 | 1000 − 2000 (16-bit) | 0xFC18 | **1** | 0 | **1** | 0 | 0 | **1** |
| sub3 | 0 − 1 | 0xFFFF | **1** | 0 | **1** | 0 | **1** | **1** |
| sub3 | `sbb ax, 0` | 0xFFFE | 0 | 0 | **1** | 0 | 0 | 0 |

- In subtraction, **CF = 1 means a borrow**, i.e. the first operand was smaller
  than the second when read as unsigned.
- **OF stayed 0** in every program because all operands were non-negative.
  Signed overflow in subtraction needs operands with different signs (for
  example `-128 - 1` in 8 bits).
