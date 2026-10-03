# DIV — Arithmetic Operations and EFLAGS

Programs run: `div1.asm`, `div2.asm`, `div3.asm`.

## How to build and inspect

```bash
nasm -f elf32 div1.asm -o div1.o
ld -m elf_i386 div1.o -o div1
gdb ./div1
```

GDB sessions:

```
# div1: AX / BL -> AL = quotient, AH = remainder
break _start
run
stepi 3
info registers ax bx eflags
p/x $eflags

# div2: DX:AX / BX -> AX = quotient, DX = remainder (stepi 4)
# div3: EDX:EAX / EBX -> EAX = quotient, EDX = remainder (stepi 4)
```

Intel rule for `DIV`: **all status flags (`CF, PF, AF, ZF, SF, OF`) are undefined**
after the instruction. GDB will still print an `EFLAGS` value (here `0x202`, i.e.
only `IF` set), but it carries no information about the division result. The real
"result" of `DIV` is the quotient/remainder register pair, plus a
divide-error fault (`#DE`) if the quotient does not fit or the divisor is zero.

## 1. div1.asm — `100 / 7` (8-bit: `AX / BL -> AL, AH`)

```nasm
mov ax, [dividend]  ; AX = 100
mov bl, [divisor]   ; BL = 7
div bl              ; AL = 100/7 = 14 (0x0E), AH = 100%7 = 2
```

Observed after `div`: `AX = 0x020E` (`AH = 2`, `AL = 14`), `EFLAGS = 0x202` → `[ IF ]`.

| Flag | State | Why |
|------|-------|-----|
| CF/PF/AF/ZF/SF/OF | undefined (observed clear) | `DIV` does not define flags. The observed `0x202` just means GDB saw all status flags clear; do not read e.g. `ZF=0` as "quotient non-zero" — that reasoning only applies to `ADD`/`SUB`. |

Result check: `14 * 7 + 2 = 100`. Quotient fits in 8 bits, so no fault.

## 2. div2.asm — `50000 / 300` (16-bit: `DX:AX / BX -> AX, DX`)

```nasm
mov ax, [dividend]  ; AX = 50000
mov dx, [highpart]  ; DX = 0
mov bx, [divisor]   ; BX = 300
div bx              ; AX = 166, DX = 200
```

Observed after `div`: `AX = 0xA6 (166)`, `DX = 0xC8 (200)`, `EFLAGS = 0x202` → `[ IF ]`.

| Flag | State | Why |
|------|-------|-----|
| CF/PF/AF/ZF/SF/OF | undefined (observed clear) | Same as above — `DIV` leaves flags undefined. Remainder `200 != 0` does not "clear ZF" in any defined sense. |

Result check: `166 * 300 + 200 = 49800 + 200 = 50000`. Quotient `166 < 65536` fits in 16 bits.

## 3. div3.asm — `300000000 / 1000` (32-bit: `EDX:EAX / EBX -> EAX, EDX`)

```nasm
mov eax, [dividend] ; EAX = 300000000
mov edx, [highpart] ; EDX = 0
mov ebx, [divisor]  ; EBX = 1000
div ebx             ; EAX = 300000 (0x493E0), EDX = 0
```

Observed after `div`: `EAX = 0x493E0 (300000)`, `EDX = 0`, `EFLAGS = 0x202` → `[ IF ]`.

| Flag | State | Why |
|------|-------|-----|
| CF/PF/AF/ZF/SF/OF | undefined (observed clear) | `DIV` leaves flags undefined. `EDX = 0` (exact division) does not mean "ZF set" — `DIV` never sets `ZF` by definition. |

Result check: `300000 * 1000 + 0 = 300000000`. Quotient fits in 32 bits.
