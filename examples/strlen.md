# strlen: Null-Terminated String Length

Returns the number of bytes in a null-terminated string (excludes the null terminator).
Uses pointer arithmetic at the end instead of a running counter, saving two instructions
per loop iteration (break-even at strings of length 2 or more).

### Register Allocation

| Register | Role | Rationale |
|----------|------|-----------|
| BC | String pointer | Callee-saved; post-increment `(bc+)` advances through the string |
| DE | Start pointer | Callee-saved; holds the original pointer for final subtraction |
| T | Byte scratch | Caller-saved; used for memory loads and null checks |
| FT | Length / scratch | Caller-saved; computes `BC - DE` via `SUB FT,DE` at the end |

Length is computed as `BC - DE` after the loop via `LD FT,BC` / `SUB FT,DE`. FT holds
the result across the epilogue since `POP HL/DE/BC` only writes H,L,D,E — not F or T.

```
; --
; -- strlen
; --   Returns the number of bytes in a null-terminated string.
; --
; -- Inputs:
; --   BC: pointer to the null-terminated string
; --
; -- Outputs:
; --   FT: length of the string (excludes null terminator)
; --
        SECTION "strlen",CODE
strlen:
        push    bc/de/hl

        ld      ft,bc           ; FT = BC (start pointer)
        ld      de,ft           ; DE = start pointer (preserved across loop)

.loop
        ld      t,(bc+)         ; T = *str++, BC advanced
        cmp     t,0             ; null terminator?
        j/ne    .loop           ; not null, continue

        sub     bc,1            ; BC = address of null terminator (undo post-increment)
        ld      ft,bc           ; FT = BC
        sub     ft,de           ; FT = BC - DE = length
        pop     hl/de/bc        ; restore callee-saved registers
        j       (hl)
```
