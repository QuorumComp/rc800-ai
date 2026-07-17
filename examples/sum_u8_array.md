# 16-Bit Accumulator: Sum of u8 Array

Sums an array of 8-bit unsigned integers into a 16-bit result.

### Register Allocation

| Register | Role | Rationale |
|----------|------|-----------|
| BC | Array pointer | Callee-saved; used for memory access with post-increment `(bc+)` |
| T | Count input / byte scratch | Caller-saved; count moved to H before first iteration |
| H | Loop counter | Holds the element count for `DJ` |
| DE | 16-bit accumulator | Callee-saved; persists across iterations without stack traffic |
| FT | Scratch add register | Caller-saved; each byte is zero-extended into FT, added to DE via `ADD FT,DE`, then written back |

FT cannot serve as the accumulator directly because `LD T,(bc+)` clobbers its low byte. Using DE as the accumulator and FT as a temporary avoids the need for `push ft` / `pop ft` around each memory load. The pattern is: load byte into T, zero-extend F, `ADD FT,DE`, then `LD DE,FT` to store the result back.

```
; --
; -- sum_u8_array
; --   Sums an array of u8 values into a 16-bit result.
; --
; -- Inputs:
; --   BC: pointer to the array
; --   T: number of elements (0-255)
; --
; -- Outputs:
; --   FT: 16-bit sum
; --
        SECTION "sum_u8_array",CODE
sum_u8_array:
        push    bc/de/hl

        ld      h,t             ; save count in H
        ld      de,0            ; clear accumulator (also handles count=0)
        cmp     h,0             ; test for empty array
        j/eq    .done           ; if count=0, result is already 0
.loop
        ld      t,(bc+)         ; fetch next byte
        ld      f,0             ; zero-extend into FT
        add     ft,de           ; FT += accumulator
        ld      de,ft           ; store back into accumulator
        dj      h,.loop         ; decrement counter, branch if more

.done
        ld      ft,de           ; result to FT
        pop     hl/de/bc
        j       (hl)
```
