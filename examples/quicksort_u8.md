# In-Place Quicksort: u8 Array

Sorts an array of 8-bit unsigned integers in ascending order using quicksort
(Lomuto partition, last-element pivot).  Recursive, no global state; all
work is done through registers and callee-saved stack frames.

### Register Allocation

| Register | Role (quicksort_u8)                          | Role (partition_u8)            |
|----------|----------------------------------------------|--------------------------------|
| BC       | Base pointer (callee-saved)                  | Base pointer (saved on stack)  |
| DE       | Scratch for address arithmetic               | Scan pointer                   |
| HL       | Return address                               | Swap pointer                   |
| T        | Count / pivot position / accumulator         | Scratch / accumulator          |
| B        | Pivot position (saved across calls)          | Pivot value                    |
| C        | —                                            | DJ loop counter                |
| F        | —                                            | Flags (not used as data)       |

The FT stack carries the original element count between the left and right
recursive calls in `quicksort_u8`, and saves the element count inside
`partition_u8` while the pivot address is computed.  The BC stack holds the
base pointer inside `partition_u8` and is used temporarily for address
arithmetic when computing the right-partition base pointer.

```
; --
; -- quicksort_u8
; --   Sorts an array of 8-bit unsigned integers in ascending order
; --   using the quicksort algorithm (Lomuto partition).  Sorts in place.
; --
; -- Inputs:
; --   T:  number of elements (0-255)
; --   BC: pointer to array of u8 values
; --
; -- Outputs:
; --   (none, array is sorted in place)
; --
        SECTION "quicksort_u8",CODE
quicksort_u8:
        push    bc/de/hl

        cmp     t,2
        j/leu   .done

        ; Save original count on FT stack for right-partition calculation
        ld      f,0
        push    ft

        ; Partition: BC = base, T = count  ->  T = pivot_position
        jal     partition_u8

        ld      d,t             ; D = pivot_position

        ; --- Left partition: base = BC, count = pivot_position ---
        cmp     d,2
        j/ltu   .no_left

        ld      t,d             ; T = pivot_position (count for left call)
        jal     quicksort_u8    ; BC already holds base pointer

.no_left
        ; --- Right partition: base = BC + pivot_pos + 1,
        ;                      count = original_count - pivot_pos - 1 ---
        pop     ft              ; T = original_count

        sub     t,d             ; T = original_count - pivot_pos
        sub     t,1             ; T = right_count
        cmp     t,2
        j/ltu   .done_right

        ; Save right_count on FT stack while we compute right_base
        ld      f,0
        push    ft

        ; right_base = BC + pivot_pos + 1
        ; Use BC stack as temporary to avoid clobbering registers
        push    bc              ; BC stack: [saved_base]
        ld      t,d             ; T = pivot_pos
        add     t,1             ; T = pivot_pos + 1
        ld      f,0             ; FT = 0:(pivot_pos+1)
        pop     bc              ; BC = saved_base
        add     ft,bc           ; FT = base + pivot_pos + 1
        ld      bc,ft           ; BC = right_base

        pop     ft              ; T = right_count
        jal     quicksort_u8

.done_right
.done
        pop     hl/de/bc
        j       (hl)

; --
; -- partition_u8
; --   Lomuto partition: places the last element in its sorted position
; --   and returns the index.  All elements <= pivot are to the left,
; --   all elements > pivot are to the right.
; --
; -- Inputs:
; --   T:  number of elements in the range (>= 2)
; --   BC: pointer to the first element of the range
; --
; -- Outputs:
; --   T:  pivot position (0-based index within the range)
; --
        SECTION "partition_u8",CODE
partition_u8:
        push    bc/de/hl

        ; Save count on FT stack (needed after pivot value is loaded)
        push    ft

        ; Compute pivot address = base + count - 1
        sub     t,1             ; T = count - 1
        ld      e,t             ; E = count - 1
        ld      d,0             ; DE = 0:(count-1)
        ld      ft,bc           ; FT = base
        add     ft,de           ; FT = pivot address

        ; Load pivot value into B (B is free once BC is restored from stack)
        ld      b,(ft)          ; B = pivot value (native LD R8,(FT); avoids clobbering T)

        ; Restore count and set up DJ loop counter
        pop     ft              ; FT = count
        sub     t,1             ; T = count - 1
        ld      c,t             ; C = loop counter

        ; Restore base pointer and set up scan/swap pointers
        pick    bc,1            ; BC = base (from BC stack)
        ld      ft,bc
        ld      de,ft           ; DE = scan pointer
        ld      hl,ft           ; HL = swap pointer

        ; Scan all elements except the pivot
.loop
        ld      t,(de)          ; T = arr[i]
        cmp     t,b             ; compare arr[i] with pivot
        j/gtu   .no_swap

        ; Swap arr[swap_idx] and arr[i]
        ld      t,(hl)          ; T = arr[swap_idx]
        push    ft              ; save arr[swap_idx] on FT stack
        ld      t,(de)          ; T = arr[i]
        ld      (hl),t          ; arr[swap_idx] = arr[i]
        pop     ft              ; T = saved arr[swap_idx]
        ld      (de),t          ; arr[i] = arr[swap_idx]
        add     hl,1            ; swap pointer advances

.no_swap
        add     de,1            ; scan pointer advances
        dj      c,.loop

.place_pivot
        ; Swap arr[swap_idx] with the pivot.  DE has walked to pivot address.
        ld      t,(hl)          ; T = old arr[swap_idx]
        push    ft              ; save it on FT stack
        ld      t,b             ; T = pivot value
        ld      (hl),t          ; arr[swap_idx] = pivot
        pop     ft              ; T = old arr[swap_idx]
        ld      (de),t          ; arr[pivot_addr] = old arr[swap_idx]

        ; Return pivot position = swap_idx = HL - base
        ld      ft,hl
        pick    bc,1            ; BC = base
        sub     ft,bc
        ; T = swap index

        pop     hl/de/bc
        j       (hl)
```
