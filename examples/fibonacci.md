# Recursive Fibonacci: u8 Input, u16 Result

Computes the nth Fibonacci number recursively. Demonstrates recursion on the RC800: saving parameters in callee-saved registers, using the FT stack to preserve intermediate results across calls, and the `swap` idiom for retrieving stacked values.

### Register Allocation

| Register | Role | Rationale |
|----------|------|-----------|
| T | Input n / scratch | Caller-saved; 8-bit parameter per calling convention |
| B | Saved n | Callee-saved; preserves n across the first recursive call so we can compute both n-1 and n-2 |
| FT | Return value / accumulator | Caller-saved; result returned here, also used to stack fib(n-1) |
| BC | fib(n-2) during final add | Callee-saved; holds one operand while the other is in FT |

After `jal fibonacci` returns `fib(n-2)` in FT, the FT stack top holds `fib(n-1)`. Saving FT into BC then swapping retrieves `fib(n-1)` into FT — no `pick` needed, just one native `swap`.

### Limits

- **n <= 24**: `fib(24) = 46369`, `fib(25) = 75025` overflows u16.
- **Stack depth**: each call pushes FT once for the intermediate result. The 256-word FT stack is not the binding constraint — u16 overflow at n=25 is.

```
; --
; -- fibonacci
; --   Recursive Fibonacci. Computes fib(n).
; --
; -- Inputs:
; --   T: n (u8)
; --
; -- Outputs:
; --   FT: fib(n) (u16)
; --
; -- Limits:
; --   n <= 24. fib(24) = 46369, fib(25) = 75025 overflows u16.
; --   FT stack depth ~256, but u16 overflow is the binding constraint.
; --
        SECTION "fibonacci",CODE
fibonacci:
        cmp     t,2
        j/ltu   .return         ; if n < 2, return n
        push    bc/de/hl

        ld      b,t             ; save n (callee-saved, survives calls)
        sub     t,1
        jal     fibonacci       ; FT = fib(n-1)
        push    ft              ; stack: [fib(n-1)]
        ld      t,b             ; restore n
        sub     t,2
        jal     fibonacci       ; FT = fib(n-2), stack: [fib(n-1)]
        ld      bc,ft           ; BC = fib(n-2)
        pop     ft              ; FT = fib(n-1), cleans up stack
        add     ft,bc           ; FT = fib(n-1) + fib(n-2)

        pop     hl/de/bc
        j       (hl)            ; return (FT already correct, skip ext)
.return
        ext                     ; F = 0 (sign-extend T, n < 128)
        j       (hl)
```
