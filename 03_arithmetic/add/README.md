# ADD — Arithmetic Operations and EFLAGS

Programs run: `add1.asm`, `add2.asm`, `add3.asm` (`adc4.asm` is empty, 0 bytes, nothing to run).

## How to build and inspect

```bash
nasm -f elf32 add1.asm -o add1.o
ld -m elf_i386 add1.o -o add1
gdb ./add1
```

GDB session used for each program:

```
break _start
run
stepi 2        # execute mov + add, stop before storing result
info registers al eflags   # or ax for 16-bit
p/x $eflags
```

For `add3.asm` a second stop was also taken after the `adc` (`stepi 3`).

Status flags referenced below: `CF` (carry), `PF` (parity of low byte, 1 = even),
`AF` (carry out of bit 3), `ZF` (result == 0), `SF` (top bit of result),
`OF` (signed overflow). `IF` is the interrupt flag, always on here, not a result flag.

## 1. add1.asm — `120 + 10` (8-bit)

```nasm
mov al, [num1]     ; AL = 120 (0x78, 01111000b)
add al, [num2]     ; AL += 10 (0x0A) -> 130 (0x82, 10000010b)
```

Observed after `add`: `AL = 0x82`, `EFLAGS = 0xA96` → `[ PF AF SF IF OF ]`.

| Flag | State | Why |
|------|-------|-----|
| CF | cleared (0) | No carry out of bit 7. `0x78 + 0x0A = 0x82` fits in 8 bits. |
| PF | set (1) | Low byte `0x82 = 10000010b` has 2 one-bits (even) → PF=1. |
| AF | set (1) | Carry out of bit 3: low nibbles `0x8 + 0xA = 0x12`, nibble overflowed → AF=1. |
| ZF | cleared (0) | Result `0x82 != 0`. |
| SF | set (1) | Bit 7 of result is 1 (`0x82`). |
| OF | set (1) | Signed overflow: two positives (`120`, `10`) produced a "negative" (`0x82` = -126 as int8). 130 > 127, so OF=1. |

This is the classic case where `CF=0` but `OF=1`: no unsigned overflow, but signed overflow.

## 2. add2.asm — `32000 + 500` (16-bit)

```nasm
mov ax, [num1]     ; AX = 32000 (0x7D00)
add ax, [num2]     ; AX += 500 (0x01F4) -> 32500 (0x7EF4)
```

Observed after `add`: `AX = 0x7EF4 (32500)`, `EFLAGS = 0x202` → `[ IF ]` (all status flags clear).

| Flag | State | Why |
|------|-------|-----|
| CF | cleared | `32000 + 500 = 32500 < 65536`, no carry out of bit 15. |
| PF | cleared | Low byte `0xF4 = 11110100b` has 5 one-bits (odd) → PF=0. |
| AF | cleared | Low nibbles `0x0 + 0x4 = 0x4`, no carry out of bit 3. |
| ZF | cleared | Result `0x7EF4 != 0`. |
| SF | cleared | Bit 15 = 0 (`0x7EF4` is positive as int16). |
| OF | cleared | No signed overflow: `32500 < 32767`, result still positive, sign bit did not wrongly flip. |

## 3. add3.asm — `0xFFFF + 1`, then `adc ax, 0` (16-bit)

```nasm
mov ax, [num1]     ; AX = 0xFFFF (65535)
add ax, [num2]     ; AX += 1 -> 0x0000, CF=1
adc ax, 0          ; AX += 0 + CF -> 0x0001 (demonstrates ADC)
```

Observed after `add`: `AX = 0x0`, `EFLAGS = 0x257` → `[ CF PF AF ZF IF ]`.
Observed after `adc`: `AX = 0x1`, `EFLAGS = 0x202` → `[ IF ]`.

After `add` (result 0):

| Flag | State | Why |
|------|-------|-----|
| CF | set | Carry out of bit 15: `0xFFFF + 1 = 0x10000`, truncated to `0x0000`. |
| PF | set | Result low byte `0x00` has 0 one-bits (even) → PF=1. |
| AF | set | Low nibble `0xF + 0x1` carries out of bit 3. |
| ZF | set | Result is zero. |
| SF | cleared | Bit 15 = 0. |
| OF | cleared | Signed: `-1 + 1 = 0`, no overflow (signs differ in effect, result correct). |

After `adc ax, 0`: `0 + 0 + CF(1) = 1`. No carry is generated, result is non-zero with bit 15 = 0,
low byte `0x01` has odd parity, so all status flags clear (`0x202`). This proves `ADC`
added the previous `CF`; without it the result would have stayed `0`.
