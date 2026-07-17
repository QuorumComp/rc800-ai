# DJ Jump-to-End Trick: Max of an Array

Finds the maximum value in an array of 8-bit unsigned integers. Demonstrates the DJ jump-to-end trick, which seeds the running value on the first iteration then jumps straight to the DJ at the bottom — naturally handling arrays of length 1.

### Register Allocation

| Register | Role | Rationale |
|----------|------|-----------|
| H | Loop counter | Part of callee-saved HL; pushed/popped with HL |
| BC | Array pointer | Callee-saved; post-increment `(bc+)` advances through the array |
| L | Running max | Part of callee-saved HL; holds the maximum value found so far |
| T | Scratch | Caller-saved; used for memory loads and comparisons |

H is used as the DJ counter because HL is callee-saved and already pushed/popped — the counter is preserved at no extra cost. BC holds the pointer (callee-saved), and L is the running max. T is free scratch since it is caller-saved and destroyed by the callee anyway.

```
; --
; -- u8_max
; --   Finds the largest u8 value in an array.
; --
; -- Inputs:
; --   T: length of the array (must be >= 1)
; --   BC: pointer to the first element of the array
; --
; -- Outputs:
; --   T: largest u8 value found in the array
; --
        SECTION "u8_max",CODE
u8_max:
        push    bc/hl

        ld      h,t             ; H = loop counter = length
        ld      t,(bc+)         ; T = arr[0], BC -> &arr[1]
        ld      l,t             ; L = running max = arr[0]
        j       .loop_end       ; DJ handles length=1 naturally

.loop
        ld      t,(bc+)         ; T = arr[i]
        cmp     t,l             ; F <- Flags[T - L]
        ld/gtu  l,t             ; L = T if T > L (unsigned)

.loop_end
        dj      h,.loop

        ld      t,l             ; T = return value = max

        pop     hl/bc
        j       (hl)
```
