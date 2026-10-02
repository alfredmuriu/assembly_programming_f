# MUL and the EFLAGS register

This folder looks at unsigned multiplication with `mul`. Each program was
assembled, stepped through in GDB, and the result registers and EFLAGS were
recorded straight after the `mul` instruction.

## How I ran the programs

```bash
nasm -f elf32 -g -F dwarf mul1.asm -o mul1.o
ld -m elf_i386 mul1.o -o mul1
gdb ./mul1
```

```
(gdb) break _start
(gdb) run
(gdb) stepi
(gdb) info registers eax edx eflags
```

## How MUL works

`mul` uses an implicit operand (AL / AX / EAX) and produces a result **twice as
wide** as its operands, so the product can never be lost:

| Operand size | Multiplies | Result goes to |
|--------------|------------|----------------|
| 8-bit  | `AL * src`  | `AX` (high half in AH) |
| 16-bit | `AX * src`  | `DX:AX` (high half in DX) |
| 32-bit | `EAX * src` | `EDX:EAX` (high half in EDX) |

## Which flags MUL defines

Unlike `add`/`sub`, `mul` **only defines CF and OF**:

- **CF = OF = 0** if the **upper half** of the result (AH / DX / EDX) is **zero**.
  The product fits in the same size as the operands.
- **CF = OF = 1** if the upper half is **non-zero**. The product needed the
  extra register.

**SF, ZF, AF and PF are *undefined*** after `mul` (Intel SDM). The CPU may leave
them as anything, so a program must never branch on them after a multiply.

`IF` (0x200) and reserved bit 1 (0x2) are always on. The final
`xor ebx, ebx` in every program sets ZF/PF (`0x246`), which comes from the exit
code, not the multiply.

---

## Program 1: `mul1.asm` (8-bit, 25 × 10)

```asm
mov al, [num1]    ; AL = 25
mul byte [num2]   ; AX = AL * 10
```

25 × 10 = 250 = `0x00FA` → **AH = 0x00**, **AL = 0xFA**

**GDB:** `AX = 0x00fa (250)`, `eflags = 0x202  [ IF ]`

| Flag | Status | Why |
|------|--------|-----|
| CF | **cleared** | AH = 0. 250 fits in a single byte (≤ 255), so the upper half wasn't needed. |
| OF | **cleared** | Always equal to CF for `mul`, for the same reason. |
| SF, ZF, AF, PF | undefined | Not computed by `mul`. As evidence: `0xFA` = `1111 1010` has six 1 bits (even), so if PF were computed it would be 1, yet GDB shows PF = 0. |

---

## Program 2: `mul2.asm` (16-bit, 3000 × 200)

```asm
mov ax, [num1]    ; AX = 3000 (0x0BB8)
mul word [num2]   ; DX:AX = AX * 200
```

3000 × 200 = 600000 = `0x0009_27C0` → **DX = 0x0009**, **AX = 0x27C0**

**GDB:** `AX = 0x27c0 (10176)`, `DX = 0x9`, `eflags = 0xa03  [ CF IF OF ]`

`0xa03` = bits 0, 1, 9 and 11, which are CF, reserved, IF and OF.

| Flag | Status | Why |
|------|--------|-----|
| CF | **set** | 600000 is bigger than 65535, so it can't fit in AX alone. The high part (9) spilled into **DX**, which is non-zero. |
| OF | **set** | Set together with CF for the same reason. |
| SF, ZF, AF, PF | undefined | Not meaningful. For example, the low byte `0xC0` has two 1 bits (even) but PF is shown as 0. |

AX on its own (10176) is **not** the answer. CF/OF = 1 tells the programmer they
must combine DX:AX to get the full product: 9 × 65536 + 10176 = 600000.

---

## Program 3: `mul3.asm` (32-bit, 100000 × 300000)

```asm
mov eax, [num1]    ; EAX = 100000
mul dword [num2]   ; EDX:EAX = EAX * 300000
```

100000 × 300000 = 30 000 000 000 = `0x0000_0006_FC23_AC00`
→ **EDX = 0x00000006**, **EAX = 0xFC23AC00**

**GDB:** `EAX = 0xfc23ac00`, `EDX = 0x6`, `eflags = 0xa03  [ CF IF OF ]`

| Flag | Status | Why |
|------|--------|-----|
| CF | **set** | 30 billion is more than 2³² − 1 (about 4.29 billion), so the product overflowed into **EDX = 6**. |
| OF | **set** | Same as CF. The upper half is non-zero. |
| SF, ZF, AF, PF | undefined | Bit 31 of EAX is 1, but SF is shown as 0. That confirms SF isn't calculated from the result. |

GDB printed EAX as **-64771072**. That is only GDB showing `0xFC23AC00` as a
signed 32-bit number. `mul` is an **unsigned** multiply, and the real answer
is EDX:EAX = 6 × 2³² + 0xFC23AC00 = 30 000 000 000.

---

## Summary

| Program | Operation | Upper half | Lower half | CF | OF |
|---------|-----------|------------|------------|----|----|
| mul1 | 25 × 10 (8-bit) | AH = 0x00 | AL = 0xFA | 0 | 0 |
| mul2 | 3000 × 200 (16-bit) | DX = 0x0009 | AX = 0x27C0 | **1** | **1** |
| mul3 | 100000 × 300000 (32-bit) | EDX = 0x6 | EAX = 0xFC23AC00 | **1** | **1** |

- For `mul`, CF and OF always have the **same** value. They answer one
  question: *did the result need the upper half register?*
- Unlike `add`, the product is never lost, because the destination is double
  width. CF/OF don't mean "error" here. They tell you whether you can use only
  the lower half.
- The other status flags are undefined, and the GDB output (PF and SF not
  matching the result bits) shows this in practice.
