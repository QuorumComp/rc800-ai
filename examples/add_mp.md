# Multiprecision Add: Source to Destination In-Place

Adds an N-byte little-endian source integer to an N-byte little-endian
destination integer in place, producing an N-byte result with carry-out.
Demonstrates the F-as-carry-lane pattern: zero-extending each byte into FT, then
using 16-bit `ADD FT` to propagate the carry through F.

### Register Allocation

| Register | Role | Rationale |
|----------|------|-----------|
| BC | Pointer to destination | In-out; `LD (BC),T` stores the sum, `ADD BC,1` advances it |
| DE | Pointer to source | In-out; post-increment `(de+)` advances through source |
| FT | 16-bit accumulator | Caller-saved; reused for each byte's carry/sum pair |
| H | Loop counter | Register value; HL is pushed/popped for the return address |
| L | Carry between bytes | Register value; persists carry-in across loop iterations |
| F | Carry lane / stash | High byte of FT holds the byte being accumulated or the carry-out |
| T | Accumulator low byte | Holds the sum byte that is stored back to the destination |

Because `JAL` stores the return address in HL, the function must save HL before
using H and L as loop counter and carry. The return address is pushed onto the
HL stack and restored at the end.

The destination pointer arrives in BC and is the same as the result pointer. The
function reads each destination byte through `LD T,(BC+)`, computes the sum with
the corresponding source byte and the running carry, restores the original
destination pointer, and writes the sum back through `LD (BC),T` followed by
`ADD BC,1`. The sum byte is stored directly from T, so it never needs to be
parked in another register.

Inside the loop, BC and DE are temporarily repurposed as the zero-extended
scratch operands `$00:src_i` and `$00:carry_in`. The destination pointer is saved
on the BC stack before it is advanced by the read and before BC is reused as
scratch; the source pointer is saved on the DE stack after it is advanced by the
read and before DE is reused as scratch. Both pointers are restored before the
sum is stored.

The byte kernel uses two 16-bit adds:

1. `ADD FT,DE` with DE = `$00:carry_in` adds the incoming carry to the low byte.
2. `ADD FT,BC` with BC = `$00:src_i` adds the next byte.

Because `T` is the low byte of `FT`, the carry-out and the sum byte cannot both
be kept in `FT` after the store. The sum byte is stored directly from `T` before
the carry-out is extracted from `F` into `L`, where it serves as the next
iteration's carry-in.

```
; --
; -- add_mp
; --   Adds an N-byte little-endian source integer to an N-byte little-endian
; --   destination integer in place.  Returns the carry-out.
; --   Demonstrates the F-as-carry-lane pattern.
; --
; -- Inputs:
; --   T:  number of bytes to add (1-255)
; --   BC: pointer to the destination integer (little-endian, in-out)
; --   DE: pointer to the source integer (little-endian, in-out)
; --
; -- Outputs:
; --   T:  carry-out (0 or 1)
; --   F:  flags set by CMP T,0 (NZ if carry, Z if no carry)
; --   BC: advanced past the destination integer
; --   DE: advanced past the source integer
; --   Memory[dest..dest+N-1]: += Memory[src..src+N-1]
; --
; -- Notes:
; --   The destination is modified in place.  The carry-out is returned in T;
; --   F is set to the comparison flags.  The routine is safe when dest <= src;
; --   overlapping ranges with dest > src may read modified source bytes.
; --

        SECTION "add_mp",CODE
add_mp:
        push    hl              ; save return address

        ld      h,t             ; H = loop counter = byte count
        ld      l,0             ; L = running carry = 0

        ; BC = destination pointer (in-out), DE = source pointer (in-out).
        ; H = loop counter, L = running carry.  Inside the loop BC and DE are
        ; reused as $00:src_i and $00:carry_in, so the advanced pointers are
        ; saved on their own stacks across the scratch reuse.

.loop
        push    bc              ; save current dest pointer (read will advance it)
        ld      t,(bc+)         ; T = dest_i, BC = dest_{i+1}
        ld      f,t             ; F = dest_i (stash; T free for src_i load)

        ld      t,(de+)         ; T = src_i, DE = src_{i+1}
        push    de              ; save advanced source pointer
        ld      c,t             ; C = src_i
        ld      b,0             ; BC = $00:src_i

        ld      t,l             ; T = carry_in
        ld      e,t             ; E = carry_in
        ld      d,0             ; DE = $00:carry_in

        ld      t,f             ; T = dest_i (recover from F stash)
        ld      f,0             ; FT = $00:dest_i

        add     ft,de           ; FT = $00:dest_i + $00:carry_in
        add     ft,bc           ; FT = carry_out:sum_i

        ; T already holds sum_i.  Restore the pointers, write the sum back,
        ; and route the carry-out (in F) into L for the next byte.
        pop     de              ; restore advanced source pointer
        pop     bc              ; restore original destination pointer
        ld      (bc),t          ; store sum_i back to destination
        add     bc,1            ; advance destination pointer

        ld      t,f             ; T = carry_out
        ld      l,t             ; L = carry_out  ->  carry-in for next byte

        dj      h,.loop

        ; Loop exit: T = final carry_out (left in place by the last iteration).
        cmp     t,0             ; F = flags (NZ if carry, Z if no carry)

        pop     hl              ; restore return address
        j       (hl)
```
