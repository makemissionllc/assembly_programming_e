# Logic and Conditions — Results

- **Class dates:** 15–16 October 2026

The source programs do not print their computed values. All five files assembled successfully with NASM as ELF32 objects and linked with `ld -m elf_i386`. Running those original binaries in the current Linux runtime ended with signal 31 (`SIGSYS`, shell status 159) at the `int 0x80` exit call, so their original runtime behavior could not complete here. Temporary 64-bit compatibility probes retained each logic instruction and wrote the resulting byte (or `TEST`'s nonzero predicate) to stdout; the repository sources were left unchanged.

| Program | Computation | Observed result | Notes |
| --- | --- | --- | --- |
| `and.asm` | `10101010b AND 11001100b` | `10001000b` (`0x88`, 136) | Result held in `AL`; program itself prints nothing. |
| `not.asm` | `NOT 10101010b` | `01010101b` (`0x55`, 85) | `NOT` does not change the flags. |
| `or.asm` | `10101010b OR 11001100b` | `11101110b` (`0xEE`, 238) | `AL` result has six set bits, so parity is even. |
| `test.asm` | `10101010b TEST 00001111b` | AND condition is `00001010b` (`0x0A`); `ZF=0` | `TEST` updates flags but does not store the AND result. The probe confirmed the nonzero condition. |
| `xor.asm` | `10101010b XOR 11001100b` | `01100110b` (`0x66`, 102) | Result has four set bits, so parity is even. |

For the arithmetic and logic instructions that update flags, parity is even for the listed results. `AND` and `OR` also set the sign flag because their result's high bit is 1; `XOR` and the `TEST` condition have a clear sign flag. `NOT` leaves flags unchanged.
