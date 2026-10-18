# Loops — Results

- **Class dates:** 15–16 October 2026
- **Runtime validation:** 18 October 2026

All programs assembled with NASM as ELF64 objects and linked with `ld`. Finite examples exited with status 0. The two unbounded runs were limited to 0.1 seconds and stopped by `timeout` (status 124).

| Program | Result |
| --- | --- |
| `01_loop_instruction.asm` | Printed `Hello, Assembly!` repeatedly and did not finish within the time limit. `syscall` overwrites `RCX`, so `loop` does not see the original count of 5. |
| `02_dec_jnz.asm` | Printed `Hello, Assembly!` five times; exited with status 0. |
| `04_while_loop.asm` | Printed the numbers 1 through 5, each on its own line; exited with status 0. |
| `07_sum_loop.asm` | Printed `Sum = 15`; exited with status 0. |
| `08_infinite_loop.asm` | Printed `Running...` repeatedly until the time limit stopped it, as expected for an infinite loop. |
| `09_loop_syscall.asm` | Printed `Hello, Assembly!` five times; exited with status 0. Saving and restoring `RCX` around `syscall` preserves the loop counter. |

## Captured finite output

`02_dec_jnz.asm` and `09_loop_syscall.asm` each printed:

```text
Hello, Assembly!
Hello, Assembly!
Hello, Assembly!
Hello, Assembly!
Hello, Assembly!
```

`04_while_loop.asm` printed:

```text
1
2
3
4
5
```

`07_sum_loop.asm` printed:

```text
Sum = 15
```
