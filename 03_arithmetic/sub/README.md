# SUB — Arithmetic Operations and EFLAGS

Programs run: `sub1.asm`, `sub2.asm`, `sub3.asm`.

## How to build and inspect

```bash
nasm -f elf32 sub1.asm -o sub1.o
ld -m elf_i386 sub1.o -o sub1
gdb ./sub1
```

GDB session:

```
break _start
run
stepi 2        # execute mov + sub, stop before storing result
info registers al eflags   # or ax for 16-bit
p/x $eflags
```

For `sub3.asm` a second stop was taken after the `sbb` (`stepi 3`).

For `SUB`, `CF` is a borrow flag (set when the subtrahend > minuend, i.e. unsigned underflow).

## 1. sub1.asm — `50 - 80` (8-bit)

```nasm
mov al, [num1]     ; AL = 50 (0x32)
sub al, [num2]     ; AL -= 80 (0x50) -> -30 (0xE2)
```

Observed after `sub`: `AL = 0xE2 (-30)`, `EFLAGS = 0x287` → `[ CF PF SF IF ]`.

| Flag | State | Why |
|------|-------|-----|
| CF | set (1) | Borrow: `50 < 80` unsigned, so subtraction wraps (`0x32 - 0x50 = 0xE2` with borrow). |
| PF | set (1) | Low byte `0xE2 = 11100010b` has 4 one-bits (even) → PF=1. |
| AF | cleared (0) | No borrow out of bit 3: low nibbles `0x2 - 0x0 = 0x2`, no nibble borrow. |
| ZF | cleared (0) | Result `0xE2 != 0`. |
| SF | set (1) | Bit 7 = 1 (`0xE2`). |
| OF | cleared (0) | No signed overflow: `50 - 80 = -30` fits in int8 (-128..127). |

## 2. sub2.asm — `1000 - 2000` (16-bit)

```nasm
mov ax, [num1]     ; AX = 1000
sub ax, [num2]     ; AX -= 2000 -> -1000 (0xFC18)
```

Observed after `sub`: `AX = 0xFC18 (-1000)`, `EFLAGS = 0x287` → `[ CF PF SF IF ]`.

| Flag | State | Why |
|------|-------|-----|
| CF | set | Borrow: `1000 < 2000` unsigned. |
| PF | set | Low byte `0x18 = 00011000b` has 2 one-bits (even) → PF=1 (PF only looks at low byte). |
| AF | cleared | Low nibble `0x8 - 0x0` needs no borrow. |
| ZF | cleared | Result `0xFC18 != 0`. |
| SF | set | Bit 15 = 1 (result is negative as int16). |
| OF | cleared | `-1000` fits in int16, so no signed overflow. |

## 3. sub3.asm — `0 - 1`, then `sbb ax, 0` (16-bit)

```nasm
mov ax, [num1]     ; AX = 0x0000
sub ax, [num2]     ; AX -= 1 -> 0xFFFF, CF=1
sbb ax, 0          ; AX -= 0 + CF -> 0xFFFE (demonstrates SBB)
```

Observed after `sub`: `AX = 0xFFFF (-1)`, `EFLAGS = 0x297` → `[ CF PF AF SF IF ]`.
Observed after `sbb`: `AX = 0xFFFE (-2)`, `EFLAGS = 0x282` → `[ SF IF ]`.

After `sub` (result `0xFFFF`):

| Flag | State | Why |
|------|-------|-----|
| CF | set | Borrow: `0 < 1`. |
| PF | set | Low byte `0xFF` has 8 one-bits (even) → PF=1. |
| AF | set | Borrow out of bit 3: low nibble `0x0 - 0x1` needs a nibble borrow. |
| ZF | cleared | Result != 0. |
| SF | set | Bit 15 = 1. |
| OF | cleared | `0 - 1 = -1` fits in int16, no signed overflow. |

After `sbb ax, 0`: `0xFFFF - 0 - CF(1) = 0xFFFE`. Now `0xFFFF >= 1` so no borrow-out → `CF=0`.
Low byte `0xFE = 11111110b` has 7 one-bits (odd) → `PF=0`. Low nibble `0xF - 0 - 1 = 0xE`
needs no borrow → `AF=0`. Result non-zero → `ZF=0`; bit 15 = 1 → `SF=1`; value `-2` fits → `OF=0`.
This proves `SBB` subtracted the previous borrow; without it the result would have stayed `0xFFFF`.
