# strchr: Character Search in String

Finds the first occurrence of a character in a null-terminated string.

### Register Allocation

| Register | Role | Rationale |
|----------|------|-----------|
| BC | String pointer | Callee-saved; post-increment `(bc+)` advances through the string |
| D | Search character | Part of callee-saved DE; holds the character to find |
| T | Byte scratch | Caller-saved; used for memory loads and comparisons |

The search character from T is saved to D at the start. D is part of DE (callee-saved),
so it persists across the loop without additional stack traffic. T is free scratch for
loading bytes and comparing against D.

```
; --
; -- strchr
; --   Finds the first occurrence of a character in a null-terminated string.
; --
; -- Inputs:
; --   BC: pointer to the null-terminated string
; --   T:  character to search for
; --
; -- Outputs:
; --   FT: pointer to the first occurrence of the character, or 0 if not found
; --
        SECTION "strchr",CODE
strchr:
        push    bc/de/hl

        ld      d,t             ; D = search character

.loop
        ld      t,(bc+)         ; T = *str++, BC advanced
        cmp     t,d             ; match?
        j/eq    .found          ; found the character
        cmp     t,0             ; null terminator?
        j/ne    .loop           ; not null, continue scanning

.found
        sub     bc,1            ; BC = pointer to matched byte (undo post-increment)
        ld      ft,bc           ; result to FT
        pop     hl/de/bc
        j       (hl)

.not_found
        ld      ft,0            ; not found
        pop     hl/de/bc
        j       (hl)
```
