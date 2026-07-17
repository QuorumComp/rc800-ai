# PICK as O(1) Lookup Table

`PICK R16` uses the register's own low byte as a stack index, giving direct random access to previously pushed values. Pre-load a table onto any register stack, then index into it in constant time — no loop, no pointer arithmetic. The pattern pays off when the table replaces a genuinely expensive per-element computation inside a tight loop, and the one-time setup cost is amortized across enough iterations.

### Register Allocation

| Register | Role | Rationale |
|----------|------|-----------|
| BC | Array pointer | Callee-saved; `(bc+)` advances through the array |
| H | Loop counter | Part of callee-saved HL; preserved by the prologue push |
| L | Running total (popcount accumulator) | Part of callee-saved HL; persists across iterations |
| DE | PICK table (on DE stack) | Callee-saved; 16-entry nibble-popcount table built at entry |
| F | Byte stash | Holds the current byte while T extracts nibbles; avoids clobbering the pointer |
| T | Scratch / accumulator | Loads bytes, splits nibbles, indexes PICK, accumulates via `ADD T,R8` |

### Population Count of a Byte Array

Counts the total number of set bits across an array of bytes. The RC811 has no bit-test, no popcount, and shifts only operate on the full 16-bit `FT`. Without a table, each byte requires an 8-iteration inner loop that shifts and masks one bit at a time — expensive because every accumulator update must route through the T hub (`ADD` only targets `T`), and the byte must be stashed in `B` (forcing a `PUSH BC`/`POP BC` per byte since `B` is part of the pointer pair).

A 16-entry nibble-popcount table collapses the inner loop to two `PICK` lookups and two `ADD`s per byte. The byte is split into nibbles via `AND T,$0F` and `RS FT,4`; each nibble indexes the table, and the two popcount values are summed into the running total in `L`.

```
; --
; -- popcount_array
; --   Counts the total number of set bits in an array of bytes.
; --   Uses a 16-entry nibble-popcount table on the DE stack, looked
; --   up via PICK for O(1) per-byte bit counting.
; --
; -- Inputs:
; --   T:  number of bytes (1-255)
; --   BC: pointer to the array
; --
; -- Outputs:
; --   T:  total number of set bits
; --
; -- Notes:
; --   Without the table, each byte requires an 8-iteration shift-and-test
; --   inner loop (~352 cycles/byte).  The 16-entry table reduces per-byte
; --   cost to 18 native instructions (72 cycles/byte), a ~4.9x speedup at
; --   N=255.  Setup costs 128 cycles: the first LD DE sets D=0 so later
; --   entries only need LD E,xx, and consecutive equal entries share one
; --   LD (the second just pushes the unchanged register).  Break-even at
; --   N=1; see break-even analysis below.
; --
        SECTION "popcount_array",CODE
popcount_array:
        push    bc/de/hl        ; save callee-saved registers

        ld      h,t             ; H = loop counter = byte count
        ld      l,0             ; L = running total = 0

        ; Build 16-entry nibble-popcount table on the DE stack.
        ; Index i returns popcount(i) for i = 0..15.
        ;       i:  0  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15
        ;  popcnt: 0  1  1  2  1  2  2  3  1  2  2  3  2  3  3  4
        ; Push entries 15..1; register holds entry 0 (= 0).
        ; Two optimizations cut the build from 46 to 28 native instructions:
        ;   1. The first LD DE sets D=0; subsequent entries only need LD E,xx
        ;      since D stays 0 across every PUSH (push duplicates, register
        ;      unchanged).
        ;   2. Consecutive equal entries share one LD: the second push omits
        ;      the LD E,xx and pushes the unchanged register (entries 13, 9,
        ;      5, 1 — values 3, 2, 2, 1).
        ld      de,$0004        ; popcount(15) = 4  (also sets D = 0)
        push    de
        ld      e,$03           ; popcount(14) = 3
        push    de
        push    de              ; popcount(13) = 3 (E unchanged)
        ld      e,$02           ; popcount(12) = 2
        push    de
        ld      e,$03           ; popcount(11) = 3
        push    de
        ld      e,$02           ; popcount(10) = 2
        push    de
        push    de              ; popcount(9)  = 2 (E unchanged)
        ld      e,$01           ; popcount(8)  = 1
        push    de
        ld      e,$03           ; popcount(7)  = 3
        push    de
        ld      e,$02           ; popcount(6)  = 2
        push    de
        push    de              ; popcount(5)  = 2 (E unchanged)
        ld      e,$01           ; popcount(4)  = 1
        push    de
        ld      e,$02           ; popcount(3)  = 2
        push    de
        ld      e,$01           ; popcount(2)  = 1
        push    de
        push    de              ; popcount(1)  = 1 (E unchanged)
        ld      e,$00           ; E = 0 (register = index 0)

.loop
        ld      t,(bc+)         ; T = next byte, BC advanced (2 native: ld+add)
        ld      f,t             ; F = byte (stash for high nibble)
        and     t,$0F           ; T = low nibble
        ld      e,t             ; E = low nibble (PICK index)
        pick    de              ; E = popcount(low nibble)

        ld      t,l             ; T = running total
        add     t,e             ; T += popcount(low)
        ld      l,t             ; L = updated total

        ld      t,f             ; T = byte (recover from stash)
        ld      f,0             ; FT = $00:byte
        rs      ft,4            ; T = high nibble
        ld      e,t             ; E = high nibble (PICK index)
        pick    de              ; E = popcount(high nibble)

        ld      t,l             ; T = running total
        add     t,e             ; T += popcount(high)
        ld      l,t             ; L = updated total

        dj      h,.loop

        ; Discard 15 table entries from DE stack via LCR.
        ld      c,RC8_SP_DE     ; C = $02 (DE SP config register index)
        lcr     t,(c)           ; T = current DE stack pointer
        add     t,15            ; advance past 15 table entries
        lcr     (c),t           ; write back — table discarded

        ld      t,l             ; T = total bit count (result)
        pop     hl/de/bc        ; restore callee-saved registers
        j       (hl)
```

### Break-Even Analysis

The table approach pays a one-time setup cost (building the 15-entry table) that the naive per-bit loop does not. Comparing total cycle counts:

| Approach | Setup | Per-byte | Total for N bytes |
|----------|------:|---------:|-------------------|
| Naive (8-iteration shift-and-test per byte) | 0 | ~352 | 352N |
| Table (two PICK lookups per byte) | 128 | 72 | 128 + 72N |

Setup is 28 native instructions (112 cycles) for the table build plus 4 instructions (16 cycles) for `LCR` cleanup. Two optimizations cut the build from 46 to 28 instructions: the first `LD DE,$0004` establishes `D=0` so later entries only need `LD E,xx` (1 instruction, not 2), and consecutive equal popcount values share one `LD` — the second entry just pushes the unchanged register (entries 13, 9, 5, 1).

Break-even at 128 + 72N = 352N, i.e. **N = 1**. The table wins for any valid input (the function requires N >= 1). The speedup grows with N:

| N | Naive | Table | Speedup |
|---|------:|------:|--------:|
| 1 | 352 | 200 | 1.8x |
| 10 | 3520 | 848 | 4.2x |
| 100 | 35200 | 7328 | 4.8x |
| 255 | 89760 | 18488 | 4.9x |

The naive per-byte cost is dominated by the 8-iteration inner loop (10 instructions/bit: recover byte, mask bit 0, stash result, route accumulator through T, shift byte, update stash, `DJ`). The table approach replaces all 80 inner-loop instructions with 16 (two lookups + two accumulator round-trips), at the cost of 128 cycles of one-time setup.

### When to Use

- **Expensive per-element computation in a tight loop**: the table replaces a computation that cannot be done in one or two native instructions (popcount, bit-reversal, non-trivial bit permutation). Do not use it for values obtainable by a single shift, mask, or `ADD` — the table setup costs more than the computation it saves.
- **Small tables (up to 256 entries per stack)**: each `PUSH` costs 4 cycles; 256 entries is 1024 cycles of setup. Amortize across enough iterations to break even.
- **Multiple independent tables**: each register stack (BC, DE, HL, FT) can hold its own table simultaneously.

### Tradeoffs

- **Setup cost**: building the table requires one `PUSH` per entry (4 cycles each). Amortize by building once and reusing across calls, or by ensuring the loop runs enough iterations to recover the cost.
- **Cleanup is cheap**: table entries are discarded in 3 instructions via `LCR` stack pointer adjustment, not N individual `POP`s. See [LCR Stack Pointer Manipulation](lcr_stack_pointer.md).
- **Index range**: limited to 0-255 per stack (the register's low byte). For larger tables, use memory-based lookup with pointer arithmetic instead.
- **Read-only**: `PICK` reads the stack but does not modify it. To update an entry, `PICK` it into the register, modify, then `PUSH` the new value and adjust depth.
