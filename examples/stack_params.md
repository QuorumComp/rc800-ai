# Stack Parameter Passing

Demonstrates the HL stack parameter spillover calling convention. When a function has more parameters than available registers, the excess parameters are pushed onto the HL stack by the caller. The callee accesses them via either the `PICK` strategy (params stay on stack) or the swap-pop strategy (params consumed as accessed), and cleans up the stack in the epilogue.

### Signature

```
f(u8 a, u16 b, u16 c, s8 d, u8 e) → u16 result
```

Five parameters: three fit in registers (T, BC, DE), two spill to the HL stack (d as signed, e as unsigned). The 8-bit param `a` occupies T (making FT unavailable), so `b` and `c` take BC and DE. The return value is 16-bit in FT.

### Register Allocation

| Register | Role | Rationale |
|----------|------|-----------|
| T | Parameter a / scratch accumulator | First 8-bit param; caller-saved; reused for d+e accumulation |
| F | Saved a | High byte of FT; survives swap-pop (only touches HL) |
| B | Part of parameter b | Callee-saved, pushed; reused to save multiplicand for multiply |
| C | Part of parameter c | Callee-saved, pushed; reused to save multiplicand for multiply |
| D | Saved d+e / DJ counter | Callee-saved; holds intermediate sum, then serves as DJ loop counter |
| E | Part of parameter e | Callee-saved; not needed in multiply (FT is the accumulator) |
| FT | Partial sum (a+b+c) / accumulator / result | Caller-saved; holds multiplicand, then product accumulator, then final result |
| HL | Return address / stack access | swap-pop uses HL directly as operand — no register save needed |

### Caller

```
        ; call f(a, b, c, d, e)
        ; registers: T=a, BC=b, DE=c  |  stack: d(s8), e(u8)

        ; Push stack params in declaration order
        ld      t,d_val
        ext                    ; sign-extend d (s8)
        ld      hl,ft
        push    hl             ; d on HL stack

        ld      t,e_val
        ld      f,0            ; zero-extend e (u8)
        ld      hl,ft
        push    hl             ; e on HL stack

        push    hl             ; sacrificial slot for jal

        ; Load register params
        ld      t,a_val
        ld      bc,b_val
        ld      ft,c_val
        ld      de,ft

        jal     f
        ; FT = result
```

**Stack layout after `jal` (callee entry):**

```
  HL (index 0, SP)   = return address
  index 1             = e  (last stack param pushed, closest to ret)
  index 2             = d  (first stack param pushed)
```

### Callee — Swap-pop Variant

The swap-pop strategy extracts each parameter and discards it, incrementally cleaning the stack. No `PUSH HL` needed — HL=ret is naturally preserved between each swap-pop.

```
; --
; -- f
; --   f(u8 a, u16 b, u16 c, s8 d, u8 e) → u16
; --   Computes: (a + low(b) + low(c)) * (d + e)
; --
; -- Inputs:
; --   T: a, BC: b, DE: c
; --   HL stack[1]: e (u8, low byte), stack[2]: d (s8, low byte)
; --
; -- Outputs:
; --   FT: (a + low(b) + low(c)) * (d + e) (u16)
; --
        SECTION "f",CODE
f:
        ld      f,t            ; F=a (save before T is reused)
        push    bc/de          ; callee-saved (NOT hl)

        ; Compute (d + e) via swap-pop — use L directly as operand
        ld      t,0            ; T=0, accumulator
        swap    hl             ; HL=e (closest to ret)
        add     t,l            ; T += e (L used directly, no register save)
        pop     hl             ; HL=ret restored, e discarded
        swap    hl             ; HL=d
        add     t,l            ; T += d
        pop     hl             ; HL=ret restored, d discarded
        ld      d,t            ; D=d+e (save in callee-saved for multiply)

        ; Compute (a + b + c) in FT
        ld      t,f            ; T=a (restored from F)
        ld      f,0            ; FT=0:a
        add     t,b            ; T=a+b
        add     t,c            ; T=a+b+c; FT=0:(a+b+c)

        ; Multiply: FT * (d+e) → FT via repeated addition
        ld      bc,ft          ; BC=multiplicand (save a+b+c)
        ld      ft,0           ; FT=0 (product accumulator)
        cmp     d,0
        j/z     .mul_end
.mul
        add     ft,bc          ; FT += multiplicand
        dj      d,.mul
.mul_end
        ; FT = result

        ; Epilogue
        push    ft             ; dup result on FT stack
        ld      ft,hl          ; FT=ret (overwrite dup; result preserved below)
        pop     de/bc          ; restore callee-saved
        pop     hl             ; discard sacrificial slot
        ld      hl,ft          ; HL=ret
        pop     ft             ; FT=result
        j       (hl)
```

### Callee — `PICK` Variant

The `PICK` strategy reads parameters at fixed indices without discarding them. `PUSH HL` saves the return address, shifting params to index 2+. Params remain on the stack for repeated or conditional access. Cleanup happens in the epilogue.

```
f_pick:
        push    hl             ; dup ret; params shift to index 2+
        push    bc/de          ; callee-saved

        pick    hl,3           ; HL=d (first stack param, deepest)
        ld      t,l
        ld      d,t            ; D=d

        pick    hl,2           ; HL=e (second stack param, closer to ret)
        ld      t,l
        ld      e,t            ; E=e
        ; ... params remain on stack for later access ...

        ; ... body: compute result in FT ...

        ; Epilogue — clean up HL stack
        push    ft             ; save result
        ld      ft,hl          ; FT=ret
        pop     de/bc          ; restore callee-saved
        pop     hl             ; discard ret dup
        pop     hl             ; discard ret
        pop     hl             ; discard e
        pop     hl             ; discard d
        ld      hl,ft          ; HL=ret
        pop     ft             ; FT=result
        j       (hl)
```

For many stack parameters, bulk cleanup via `LCR` is more compact than individual pops:

```
        push    ft             ; save result
        ld      ft,hl          ; FT=ret
        pop     de/bc          ; restore callee-saved
        ld      de,ft          ; DE=ret (save; LCR clobbers FT)
        ld      c,RC8_SP_HL    ; C = SP_HL index
        lcr     t,(c)          ; T = SP_HL
        add     t,4            ; advance past ret dup, ret, e, d
        lcr     (c),t          ; SP_HL = new value
        ld      hl,de          ; HL=ret (restore from DE)
        pop     ft             ; FT=result
        j       (hl)
```

### Combined Strategy

A function might `PICK` a frequently-accessed parameter and swap-pop the rest:

```
f_combined:
        push    hl             ; dup ret for PICK
        push    bc/de          ; callee-saved

        pick    hl,3           ; HL=d (accessed multiple times in loop)
        ld      t,l
        ld      d,t            ; D=d

        pop     hl             ; discard ret dup, back to swap-pop mode
        swap    hl             ; HL=e
        ld      t,l
        ld      e,t            ; E=e
        pop     hl             ; HL=ret restored, e discarded
        ; ... body uses D repeatedly, E once ...
```

### Notes

**Using HL directly as operand**: between `SWAP HL` and `POP HL`, the parameter sits in HL and can be used directly: `ADD T,L`, `ADD FT,HL`, `LD (BC),L`, `AND T,H`, etc. No register save is needed when the parameter is consumed immediately. This is the primary benefit of swap-pop — it eliminates the intermediate save step. When a parameter must be reused across multiple operations, saving to a callee-saved register byte is still necessary.

**Parameter `a` survival**: `T` is clobbered by swap-pop. Save `a` in F (`LD F,T`) before param extraction — F survives because swap/pop only touches HL. For the `PICK` variant, the same applies.

**Swap-pop mechanics**: `SWAP HL` exchanges HL with `Stack[SP+1]`. The old HL value (ret_addr) is written to `Stack[SP+1]`. `POP HL` advances SP and reloads HL from `Stack[SP]` — which now holds ret_addr. The parameter is available in HL between swap and pop.

**Extension discipline**: `d` is declared `s8` — the caller sign-extends with `EXT`. `e` is declared `u8` — the caller zero-extends with `LD F,0`. The callee reads only the low byte (`L`) for 8-bit params; the high byte is available if sign-extension is needed for arithmetic.

**Signed params and full-word access**: for signed arithmetic that depends on the sign extension, use the full HL word:

```
        swap    hl             ; HL=d=$FFFF (sign-extended)
        add     ft,hl          ; FT += d (full 16-bit, directly)
        pop     hl             ; HL=ret
```

For 8-bit params where only the low byte is needed (e.g., array indexing), `ADD T,L` is sufficient.

**Stack balance**: caller pushes 3 entries (d, e, sacrificial). Callee pops 3 total for swap-pop (2 swap-pops + 1 final `POP HL`). PICK variant pushes 1 extra (ret dup) and pops 4 in the epilogue. LCR advances by 4. Net: zero. All four register stacks are balanced on return.

### Worked Trace (swap-pop variant)

Assume `a=1, b=$0002, c=$0003, d=-1 (s8, $FF), e=4`. Expected: `(1+low($0002)+low($0003)) * (-1+4) = 6 * 3 = 18`.

**Caller:**
```
   push d:  HL=$FFFF (sign-extended), SP_HL decremented
  push e:  HL=$0004 (zero-extended), SP_HL decremented
  push sacrificial: SP_HL decremented
  jal f:   HL=ret_addr (overwrites sacrificial)
```

**Callee prologue:**
```
  ld f,t:   F=1 (save a)
  push bc/de:  BC=$0002, DE=$0003 saved on stacks
```

**Swap-pop e (L used directly):**
```
  ld t,0:   T=0 (accumulator for d+e)
  swap hl:  HL=$0004 (e), Stack[SP+1]=ret
  add t,l:  T=0+4=4 (L used directly as operand)
  pop hl:   HL=ret, SP_HL incremented
```

**Swap-pop d (L used directly):**
```
  swap hl:  HL=$FFFF (d, sign-extended), Stack[SP+1]=ret
  add t,l:  T=4+$FF=$03 (low byte; signed wrap: -1+4=3)
  pop hl:   HL=ret, SP_HL incremented
  ld d,t:   D=$03 (save d+e for multiply)
```

**Body:**
```
  a+b+c: T=F(1), LD F,0, T+B(2)+C(3) = 6, FT=$0006
  d+e:   D=$03 (signed: -1+4=3; low-byte wrap gives correct result)
  mul:   ld bc,ft → BC=$0006, ld ft,0 → FT=$0000
         cmp d,0 → not zero, fall through
         iter1: add ft,bc → FT=$0006, dj d → D=2, jump
         iter2: add ft,bc → FT=$000C, dj d → D=1, jump
         iter3: add ft,bc → FT=$0012, dj d → D=0, exit
         FT = 18 ($0012)
```

**Note on signed params**: `ADD T,L` reads only the low byte. For d=-1 ($FF), L=$FF, so T=4+$FF=$03 — the low-byte wrap happens to produce the correct signed result (-1+4=3). This works whenever the signed sum fits in 8 bits. When the result may exceed 8 bits, use `ADD FT,HL` to incorporate the full sign-extended value.

**Epilogue:**
```
  push ft:  FT stack dup = result ($0012)
  ld ft,hl: FT=ret, result preserved below
  pop de/bc: BC=$0002, DE=$0003 restored
  pop hl:   SP_HL incremented, sacrificial discarded
  ld hl,ft: HL=ret
  pop ft:   FT=$0012 (result)
  j (hl):   return
```

### Nested Call Pattern

When `f` calls another function that uses stack parameters, the inner callee's HL cleanup will discard outer stack entries. Save the outer return address before the inner call:

```
        push    ft             ; save FT
        ld      ft,hl          ; FT=ret (save outer return address)
        ; ... set up inner call params, jal inner ...
        ; Inner callee cleans its own HL stack.
        ; If inner call used stack params, advance SP past them:
        ld      c,RC8_SP_HL    ; C = SP_HL index
        lcr     t,(c)          ; T = SP_HL
        add     t,N+1          ; N = inner stack param count
        lcr     (c),t          ; SP_HL = new value
        ld      hl,ft          ; restore outer ret
        pop     ft             ; restore FT
```
