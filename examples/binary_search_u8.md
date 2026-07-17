# Binary Search: Sorted u8 Array

Binary searches a sorted array of 8-bit unsigned integers for a target element.
Returns the index on success with F set to EQ, or sets F to NE if not found.

### Register Allocation

| Register | Role | Rationale |
|----------|------|-----------|
| BC | Array base pointer | Callee-saved; moved from FT at entry |
| D | Search key | Callee-saved (part of DE); persists across loop |
| H | Low index | Callee-saved; bounds check and update |
| L | High index | Callee-saved; paired with H in HL |
| E | Mid index | Callee-saved (part of DE); survives address arithmetic |
| T | Scratch / return | Caller-saved; memory loads, mid computation, return index |

Address computation routes through the FT hub: base is saved to DE, mid is
zero-extended into FT, then `ADD FT,DE` produces the element address. E holds
mid across the clobber so it remains available for bounds updates and the
return value.

```
; --
; -- binary_search_u8
; --   Binary searches a sorted array of u8 values for a target element.
; --
; -- Inputs:
; --   FT: pointer to the sorted array
; --   B:  number of elements (0-255)
; --   C:  element to search for
; --
; -- Outputs:
; --   T:  index of the element if found (undefined if not found)
; --   F:  EQ if found, NE if not found
; --
; -- Precondition:
; --   The array must be sorted in ascending order.
; --
        SECTION "binary_search_u8",CODE
binary_search_u8:
        push    bc/de/hl

        ld      bc,ft            ; BC = base pointer
        ld      t,c              ; T = key
        ld      d,t              ; D = key (callee-saved)
        cmp     b,0              ; empty array?
        j/eq    .not_found

        ld      h,0              ; H = low = 0
        ld      t,b              ; T = len
        sub     t,1              ; T = len - 1
        ld      l,t              ; L = high = len - 1

.loop
        ld      t,h              ; T = low
        cmp     t,l              ; F = Flags[low - high]
        j/gtu   .not_found       ; low > high, exhausted

        ; mid = (low + high) / 2 — T already holds low
        ld      f,0              ; FT = 0:low
        add     t,l              ; FT = 0:(low + high)
        rs      ft,1             ; FT = (low + high) / 2
        ld      e,t              ; E = mid

        ; Load arr[base + mid] into T
        ld      ft,bc            ; FT = base
        ld      de,ft            ; DE = base (save)
        ld      t,e              ; T = mid (clobbers FT)
        ld      f,0              ; FT = 0:mid
        add     ft,de            ; FT = base + mid
        ld      t,(ft)           ; T = arr[mid]

        cmp     t,d              ; F = Flags[arr[mid] - key]
        j/eq    .found           ; match!

        j/ltu   .go_high         ; arr[mid] < key, search upper half

        ; arr[mid] > key, high = mid - 1
        cmp     e,0              ; mid is 0, can't go lower
        j/eq    .not_found
        ld      t,e              ; T = mid
        sub     t,1              ; T = mid - 1
        ld      l,t              ; L = mid - 1
        j       .loop

.go_high
        ld      t,e              ; T = mid
        add     t,1              ; T = mid + 1
        ld      h,t              ; H = mid + 1
        j       .loop

.found
        ld      t,e              ; T = index (return value)
        ld      f,2              ; F = FLAGS_EQ
        pop     hl/de/bc
        j       (hl)

.not_found
        ld      f,0              ; F = FLAGS_NE
        pop     hl/de/bc
        j       (hl)
```
