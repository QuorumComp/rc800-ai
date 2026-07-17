# 4-Bit to ASCII Hex Conversion

Converts a 4-bit unsigned value (0-15) in T to its ASCII hex character. Values 0-9 produce '0'-'9'; values 10-15 produce 'A'-'F'.

### Algorithm

Adds `'0'` (0x30) unconditionally, then conditionally adds 7 more when the original value is >= 10. The difference between `'0'` (0x30) and `'A'` (0x41) is exactly 7, so a single conditional add bridges the gap between the digit and letter ranges.

### Register Allocation

| Register | Role | Rationale |
|----------|------|-----------|
| T | Input/output | Caller-scratch; holds the 4-bit value on entry, the ASCII character on return |
| F | Flags | Set by `CMP`; overwritten as a side effect but no value is needed after the comparison |

No callee-saved registers are used — no prologue or epilogue required.

```
; --
; -- nibble_to_hex
; --   Converts a 4-bit value (0-15) to its ASCII hex character.
; --
; -- Inputs:
; --   T: 4-bit unsigned value (0-15), upper 4 bits must be 0
; --
; -- Outputs:
; --   T: ASCII hex character ('0'-'9' for 0-9, 'A'-'F' for 10-15)
; --
        SECTION "nibble_to_hex",CODE
nibble_to_hex:
        cmp     t,10              ; F <- Flags[T - 10]
        add     t,'0'             ; T += 0x30, all nibbles shifted into '0' range
        add/geu t,7               ; T += 7 only if original nibble >= 10
                                  ; (synthesized: j/c,@+2 / add t,7)
        j       (hl)              ; return
```
