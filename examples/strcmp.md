# strcmp: String Comparison

Compares two null-terminated strings byte by byte. Returns the signed difference
of the first differing bytes with the corresponding condition code in F.

### Register Allocation

| Register | Role | Rationale |
|----------|------|-----------|
| BC | Pointer to s1 | Callee-saved; post-increment `(bc+)` advances through s1 |
| DE | Pointer to s2 | Callee-saved; post-increment `(de+)` advances through s2 |
| T | Byte from s2 / scratch | Caller-saved; primary comparison operand |
| H | Byte from s1 | Part of callee-saved HL; holds first byte across CMP (which overwrites F) |

```
; --
; -- strcmp
; --   Compares two null-terminated strings byte by byte.
; --
; -- Inputs:
; --   BC: pointer to the first string (s1)
; --   DE: pointer to the second string (s2)
; --
; -- Outputs:
; --   T:  signed difference of first differing bytes (0 if equal)
; --   F:  condition code: EQ if equal, LTU if s1 < s2, GTU if s1 > s2
; --
        SECTION "strcmp",CODE
strcmp:
        push    bc/de/hl

.loop
        ld      t,(bc+)         ; T = *s1++
        ld      h,t             ; H = byte from s1 (temp hold, HL is callee-saved)
        ld      t,(de+)         ; T = *s2++
        cmp     t,h             ; F = Flags[T - H]
        j/ne    .diff           ; different bytes → return difference
        cmp     t,0             ; both equal — null terminator?
        j/ne    .loop           ; not null, continue scanning
        ld      t,0             ; strings are equal

        pop     hl/de/bc
        j       (hl)

.diff
        sub     t,h             ; T = s1_byte - s2_byte (F preserved)
        pop     hl/de/bc
        j       (hl)
```
