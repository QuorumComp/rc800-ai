# In-Place Bubble Sort: u8 Array

Sorts an array of 8-bit unsigned integers in ascending order using bubble sort.

### Register Allocation

| Register | Role | Rationale |
|----------|------|-----------|
| BC | Array base pointer | Callee-saved; pushed/popped at function boundaries; also saved/restored on BC stack around each inner pass |
| DE | Inner loop scan pointer | Callee-saved; reset from BC at the start of each outer pass |
| H | Outer loop counter | Holds remaining pass count (n-1, decremented each pass) |
| L | Inner loop counter | Holds comparisons remaining in current pass |
| C | Left element of pair | Part of callee-saved BC; holds arr[j] during comparison (BC saved around inner loop) |
| T | Right element / scratch | Caller-saved; holds arr[j+1] during comparison |

C holds arr[j] and T holds arr[j+1] during comparison. Because `LD C,T` modifies C (part of BC), the base pointer is saved on the BC stack at the start of each outer pass and restored after the inner loop.

```
; --
; -- bubble_sort_u8
; --   Sorts an array of 8-bit unsigned integers in ascending order
; --   using the bubble sort algorithm. Sorts in place.
; --
; -- Inputs:
; --   T:  number of elements (0-255)
; --   BC: pointer to array of u8 values
; --
; -- Outputs:
; --   (none, array is sorted in place)
; --
        SECTION "bubble_sort_u8",CODE
bubble_sort_u8:
        push    bc/de/hl

        cmp     t,1
        j/leu   .done

        ld      h,t
        sub     h,1

.outer_loop
        ld      ft,bc
        ld      de,ft
        ld      t,h
        ld      l,t

        push    bc              ; save base on BC stack (inner loop clobbers C)

.inner_loop
        ld      t,(de+)         ; T = arr[j], DE++
        ld      c,t             ; C = arr[j]
        ld      t,(de)          ; T = arr[j+1]

        cmp     t,c
        j/geu   .inner_next

.swap
        sub     de,1            ; DE = j
        ld      (de+),t         ; arr[j] = old arr[j+1], DE++
        ld      t,c             ; T = old arr[j]
        ld      (de),t          ; arr[j+1] = old arr[j]

.inner_next
        dj      l,.inner_loop

        pop     bc              ; restore base from BC stack
        dj      h,.outer_loop

.done
        pop     hl/de/bc
        j       (hl)
```
