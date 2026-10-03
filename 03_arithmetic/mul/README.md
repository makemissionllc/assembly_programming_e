# MUL — Arithmetic Operations and EFLAGS

Programs run: `mul1.asm`, `mul2.asm`, `mul3.asm`.

## How to build and inspect

```bash
nasm -f elf32 mul1.asm -o mul1.o
ld -m elf_i386 mul1.o -o mul1
gdb ./mul1
```

GDB session:

```
break _start
run
stepi 2        # execute mov + mul
info registers ax eflags       # mul1 (AX)
info registers ax dx eflags    # mul2 (DX:AX)
info registers eax edx eflags  # mul3 (EDX:EAX)
p/x $eflags
```

Intel rule for `MUL`: only `CF` and `OF` are defined — set if the upper half of the
product is non-zero (result does not fit in the low half), cleared otherwise.
`SF`, `ZF`, `AF`, `PF` are **undefined** after `MUL`; whatever GDB prints for them
must not be interpreted as a result property.

## 1. mul1.asm — `25 * 10` (8-bit: `AL * r/m8 -> AX`)

```nasm
mov al, [num1]      ; AL = 25
mul byte [num2]     ; AX = 25 * 10 = 250 (0x00FA)
```

Observed after `mul`: `AX = 0xFA (250)`, `AH = 0`, `EFLAGS = 0x286` → `[ PF SF IF ]`.

| Flag | State | Why |
|------|-------|-----|
| CF | cleared (0) | Upper half `AH = 0`, product fits in `AL` → no overflow. |
| OF | cleared (0) | Same condition as CF for `MUL`. |
| SF/ZF/AF/PF | undefined | GDB happens to show `PF=1, SF=1` here, but per the manual these are meaningless after `MUL`. `PF=1` only reflects that `0xFA` has 6 one-bits, not a documented outcome. |

Why `CF=OF=0`: `250 < 256`, so the 16-bit result `0x00FA` has a zero upper byte.

## 2. mul2.asm — `3000 * 200` (16-bit: `AX * r/m16 -> DX:AX`)

```nasm
mov ax, [num1]      ; AX = 3000
mul word [num2]     ; DX:AX = 3000 * 200 = 600000 (0x000927C0)
```

Observed after `mul`: `AX = 0x27C0`, `DX = 0x9`, `EFLAGS = 0xA07` → `[ CF PF IF OF ]`.

| Flag | State | Why |
|------|-------|-----|
| CF | set (1) | Upper half `DX = 0x0009 != 0`, product does not fit in 16 bits. |
| OF | set (1) | Same condition as CF for `MUL`. |
| SF/ZF/AF/PF | undefined | Ignore; e.g. displayed `PF=1` is not architecturally meaningful. |

Math check: `3000 * 200 = 600000 = 0x927C0`, so `DX = 0x09`, `AX = 0x27C0`. Stored as
`[result] = AX`, `[result+2] = DX`.

## 3. mul3.asm — `100000 * 300000` (32-bit: `EAX * r/m32 -> EDX:EAX`)

```nasm
mov eax, [num1]     ; EAX = 100000
mul dword [num2]    ; EDX:EAX = 100000 * 300000 = 30000000000 (0x6FC23AC00)
```

Observed after `mul`: `EAX = 0xFC23AC00`, `EDX = 0x6`, `EFLAGS = 0xA87` → `[ CF PF SF IF OF ]`.

| Flag | State | Why |
|------|-------|-----|
| CF | set (1) | Upper half `EDX = 0x6 != 0`, product needs 64 bits. |
| OF | set (1) | Same condition as CF for `MUL`. |
| SF/ZF/AF/PF | undefined | Ignore the displayed `PF/SF` values. |

Math check: `100000 * 300000 = 30,000,000,000 = 0x6_FC23AC00`, so `EDX = 6`,
`EAX = 0xFC23AC00`. Stored as `[result] = EAX`, `[result+4] = EDX`.
