# setjmp / longjmp via LCR

Saves and restores the full execution context (four register stack pointers plus the return address) into a 5-byte `jmp_buf` in memory. `setjmp` returns 0 on direct call; `longjmp(buf, val)` restores the context and returns `val` to the original `setjmp` caller.

The mechanism uses `LCR` to read and write the four stack-pointer configuration registers directly, bypassing `PUSH`/`POP` entirely for the restore. This is the only way to recover a caller's register-stack state after the stacks have been modified by intervening code.

### jmp_buf Layout (5 bytes)

```
buf[0] = SP_FT   (pre-prologue; compensated for setjmp's own PUSH FT)
buf[1] = SP_DE   (pre-prologue; uncontaminated by prologue)
buf[2] = SP_BC   (pre-prologue; compensated for prologue's PUSH BC)
buf[3] = SP_HL   (pre-prologue; compensated for prologue's PUSH HL; Stack[SP_HL+1] = return address)
buf[4] = val     (written by longjmp before restores)
```

### Register Allocation

| Register | Role | Rationale |
|----------|------|-----------|
| BC | Buffer pointer | Callee-saved; setjmp-era BC = buf (the parameter). In longjmp, `ADD BC,1` after each `POP BC` advances the pointer, but each such `ADD` also modifies `Stack[SP_BC]` (the surviving duplicate), so `Stack[SP_BC_setjmp]` no longer holds buf by the time the BC restore runs. The BC restore therefore saves the pointer on the FT stack before the `LCR` and recovers it afterward — see below |
| D | `val` parameter (longjmp) | Held across Phase 0; written to `buf[4]` before any SP restore |
| FT | Store pointer (setjmp) / scratch + save area (longjmp) | Caller-saved; the store pointer in setjmp; freely clobbered by `LCR T,(C)` reads in longjmp; also used to save the buffer pointer on the FT stack during the BC restore |
| C | LCR config-register index | Set by `LD C,RC8_SP_xx`; each `LD C,i8` clobbers BC's low byte (modifies `Stack[SP_BC]`), but saved duplicates on the BC stack survive — see below |
| T | SP value hub | Hub register; carries the SP value to/from `LCR` |
| HL | Return address | Set by `JAL` at setjmp entry; pushed onto the HL stack; longjmp's `LCR (C),T` restore of SP_HL makes `HL = Stack[SP_HL] = return addr` |

### The PUSH-Duplicate Survival Rule

`PUSH R16` duplicates the current top: both `Stack[new_SP]` and `Stack[new_SP + 1]` hold the same value after the push. `LD C,i8` writes the low byte of `Stack[SP_BC]` (the current top), corrupting it — but the duplicate at `Stack[SP_BC + 1]` is untouched. A subsequent `POP BC` advances past the corrupted top and loads the surviving duplicate. This is why setjmp and longjmp can freely `LD C,i8` between `PUSH BC` / `POP BC` pairs: the saved buffer pointer survives in the duplicate.

**Caveat — `ADD BC,s8` after a `POP BC` modifies the surviving duplicate.** Because BC is a live view into `Stack[SP_BC]`, any write to B or C — including `ADD BC,1` used to advance the buffer pointer — overwrites `Stack[SP_BC]`, the very slot that the next `PUSH BC` will duplicate. The pointer advancement is correct (each `POP BC` recovers the advanced value), but `Stack[SP_BC_setjmp]` drifts away from buf. The BC restore therefore cannot rely on `Stack[SP_BC_setjmp] = buf`; it saves the pointer on the FT stack instead (see longjmp Phase 1).

### setjmp

```
; --
; -- setjmp
; --   Saves the current execution context into a jmp_buf.
; --   Returns 0 on direct call, non-zero after longjmp.
; --
; -- Inputs:
; --   BC: pointer to 5-byte jmp_buf
; --   HL: return address (set by JAL at the call site)
; --
; -- Outputs:
; --   T:  0 on direct call, val after longjmp
; --   BC: restored to original value (callee-saved)
; --   HL: restored to original value (callee-saved)
; --
        SECTION "setjmp",CODE
setjmp:
        push    hl              ; save return address on HL stack (SP_HL -= 1)
        push    bc              ; save buf pointer on BC stack (SP_BC -= 1)
        ld      ft,bc           ; FT = buf (store pointer; BC preserved)

        ; Capture SP_FT -> buf[0].
        ; The PUSH FT used to save the buf pointer contaminates SP_FT by -1;
        ; ADD T,1 compensates so the stored value is the pre-push SP_FT.
        push    ft              ; save buf pointer on FT stack (SP_FT -= 1)
        ld      c,RC8_SP_FT     ; C = $00 (clobbers BC, but BC saved)
        lcr     t,(c)           ; T = SP_FT (post-push value)
        add     t,1             ; T = pre-push SP_FT (compensate for the PUSH FT)
        pop     ft              ; restore buf pointer (SP_FT += 1)
        ld      (ft),t          ; buf[0] = SP_FT
        add     ft,1            ; FT = buf+1

        ; Capture SP_DE -> buf[1].
        ; PUSH FT lands on the FT stack, so SP_DE is uncontaminated.
        push    ft              ; save buf pointer
        ld      c,RC8_SP_DE     ; C = $02
        lcr     t,(c)           ; T = SP_DE
        pop     ft              ; restore buf pointer
        ld      (ft),t          ; buf[1] = SP_DE
        add     ft,1            ; FT = buf+2

        ; Capture SP_BC -> buf[2].
        ; The prologue's PUSH BC contaminates SP_BC by -1; ADD T,1
        ; compensates so the stored value is the pre-prologue SP_BC.
        push    ft              ; save buf pointer
        ld      c,RC8_SP_BC     ; C = $01
        lcr     t,(c)           ; T = SP_BC (post-prologue value)
        add     t,1             ; T = pre-prologue SP_BC (compensate for PUSH BC)
        pop     ft              ; restore buf pointer
        ld      (ft),t          ; buf[2] = SP_BC
        add     ft,1            ; FT = buf+3

        ; Capture SP_HL -> buf[3].
        ; The prologue's PUSH HL contaminates SP_HL by -1; ADD T,1
        ; compensates so the stored value is the pre-prologue SP_HL.
        push    ft              ; save buf pointer
        ld      c,RC8_SP_HL     ; C = $03
        lcr     t,(c)           ; T = SP_HL (post-prologue value)
        add     t,1             ; T = pre-prologue SP_HL (compensate for PUSH HL)
        pop     ft              ; restore buf pointer
        ld      (ft),t          ; buf[3] = SP_HL
        ; no ADD FT,1 — last slot

        pop     bc              ; restore buf pointer (SP_BC += 1)
        pop     hl              ; restore SP_HL (SP_HL += 1); HL = return address
        ld      t,0             ; return 0
        j       (hl)            ; return to caller
```

### longjmp

```
; --
; -- longjmp
; --   Restores the execution context saved by setjmp.
; --   After return, T = val (the return value seen by setjmp).
; --
; -- Inputs:
; --   BC: pointer to 5-byte jmp_buf
; --   D:  return value (0-255, conventionally non-zero)
; --
; -- Outputs:
; --   T:  val (returned to setjmp caller)
; --   All register stacks restored to setjmp-era state
; --
        SECTION "longjmp",CODE
longjmp:
        ; Phase 0: write val into buf[4] before any SP restore.
        ld      t,d             ; T = val (clobbers FT)
        push    bc              ; save buf pointer on BC stack
        add     bc,4            ; BC = buf+4
        ld      (bc),t          ; buf[4] = val
        pop     bc              ; restore buf pointer

        ; Phase 1: restore all 4 SPs via LCR. Order: FT, DE, BC, HL.
        ; BC is restored third so that the BC-stack duplicate recovery
        ; (used by the FT and DE restores) is still valid.

        ; Restore SP_FT from buf[0].
        ld      t,(bc)          ; T = buf[0] = SP_FT (clobbers FT)
        push    bc              ; save buf pointer
        ld      c,RC8_SP_FT     ; C = $00
        lcr     (c),t           ; SP_FT = T (FT = Stack[new_SP_FT])
        pop     bc              ; restore buf pointer
        add     bc,1            ; BC = buf+1

        ; Restore SP_DE from buf[1].
        ld      t,(bc)          ; T = buf[1] = SP_DE
        push    bc              ; save buf pointer
        ld      c,RC8_SP_DE     ; C = $02
        lcr     (c),t           ; SP_DE = T (DE = Stack[new_SP_DE])
        pop     bc              ; restore buf pointer
        add     bc,1            ; BC = buf+2

        ; Restore SP_BC from buf[2] — special: cannot use PUSH/POP BC around
        ; the LCR because it moves SP_BC itself. LD C,i8 clobbers Stack[SP_BC]
        ; (the current top), and the restored SP equals the current SP, so
        ; BC would read the corrupted slot. Save the pointer on the FT stack
        ; (already restored) and recover it after the LCR.
        ld      ft,bc           ; FT = current pointer (buf+2)
        push    ft              ; save on FT stack (SP_FT -= 1)
        ld      t,(bc)          ; T = buf[2] = SP_BC
        ld      c,RC8_SP_BC     ; C = $01 (clobbers BC)
        lcr     (c),t           ; SP_BC = T (BC = Stack[SP_BC] = garbage)
        pop     ft              ; FT = saved pointer (SP_FT += 1)
        ld      bc,ft           ; BC = pointer (restored)
        add     bc,1            ; BC = buf+3 (advance to SP_HL slot)

        ; Restore SP_HL from buf[3] — must be last: after it HL = return addr.
        ld      t,(bc)          ; T = buf[3] = SP_HL
        push    bc              ; save buf+3 on BC stack
        ld      c,RC8_SP_HL     ; C = $03
        lcr     (c),t           ; SP_HL = T (HL = return address)
        pop     bc              ; restore buf+3 (BC = buf+3)

        ; Phase 2: load return value and jump.
        add     bc,1            ; BC = buf+4 (val slot)
        ld      t,(bc)          ; T = val
        j       (hl)            ; return to setjmp caller
```

### Worked Trace

Assume `BC = $2004` (buf), `HL = $1050` (return address), and initial stack pointers `SP_FT = $FE`, `SP_BC = $FC`, `SP_DE = $FA`, `SP_HL = $FB`.

**setjmp** — after the prologue (`PUSH HL`, `PUSH BC`), the pointers are `SP_HL = $FA`, `SP_BC = $FB`. The four captures store into `buf` without disturbing those SPs (each `PUSH FT`/`POP FT` pair is balanced on the FT stack). The `ADD T,1` in the SP_FT capture cancels the contamination from the capture's own `PUSH FT`; the `ADD T,1` in the SP_BC and SP_HL captures cancels the contamination from the prologue's `PUSH BC` and `PUSH HL`. The `POP BC` and `POP HL` at the end restore `SP_BC` and `SP_HL` to their pre-call values.

Result: `buf = [$FE, $FA, $FC, $FB]`, `T = 0`, `BC = $2004` restored, `HL = $1050` restored, stack pointers `SP_FT = $FE`, `SP_BC = $FC`, `SP_DE = $FA`, `SP_HL = $FB`.

**longjmp** — with `BC = $2004`, `D = $05`, `buf = [$FE, $FA, $FC, $FB, —]`:

- Phase 0 writes `buf[4] = $05`; `POP BC` recovers `BC = $2004`.
- FT restore: `LD T,(BC)` loads `$FE`; `PUSH BC`/`LD C,$00`/`LCR`/`POP BC` sets `SP_FT = $FE`, recovers `BC = $2004`; `ADD BC,1` → `$2005`.
- DE restore: `LD T,(BC)` loads `$FA`; `PUSH BC`/`LD C,$02`/`LCR`/`POP BC` sets `SP_DE = $FA`, recovers `BC = $2005`; `ADD BC,1` → `$2006`.
- BC restore: `LD FT,BC` saves `$2006` on the FT stack; `LD T,(BC)` loads `$FC`; `LD C,$01` clobbers BC's low byte; `LCR (C),T` sets `SP_BC = $FC` (BC = `Stack[$FC]` = garbage); `POP FT` recovers `FT = $2006`; `LD BC,FT` sets `BC = $2006`; `ADD BC,1` → `$2007`.
- HL restore: `LD T,(BC)` loads `$FB`; `PUSH BC` saves `$2007`; `LD C,$03` clobbers that slot's low byte; `LCR (C),T` sets `SP_HL = $FB`, `HL = Stack[$FB] = $1050`; `POP BC` recovers `$2007` from the surviving `Stack[$FC]` duplicate.
- Phase 2: `ADD BC,1` → `$2008`, `LD T,(BC)` loads `T = $05`, `J (HL)` returns to `$1050`.

Result: `T = $05`, `HL = $1050`, all four SPs restored to their pre-call values (`SP_FT = $FE`, `SP_DE = $FA`, `SP_BC = $FC`, `SP_HL = $FB`).

### Edge Cases

- **`val = 0`**: `buf[4] = 0`, so `T = 0` after longjmp — indistinguishable from a direct `setjmp` call. This matches the C convention; callers that need to distinguish the two paths must pass a non-zero `val`.
- **`val = 255`**: `buf[4] = $FF`, `T = $FF`; no overflow (8-bit).
- **Nested `setjmp`**: each `setjmp` captures its own SPs into its own `jmp_buf`; `longjmp` restores only the buffer it is passed. The two do not conflict because each `jmp_buf` is self-contained.
- **Stack boundary (`SP = $FF`, empty)**: writing `$FF` via `LCR` is valid; the register pair reflects `Stack[$FF]` (undefined, do not rely on it).
- **Stack boundary (`SP = $00`, 255 entries)**: `PUSH` decrements to `$FF` (wraps). `LCR` writes to `$00` are valid.

### Instruction Count

- **setjmp**: 33 native instructions = 132 cycles (30 base + 3 compensation/restoration: `ADD T,1` for SP_BC, `ADD T,1` for SP_HL, `POP HL` in epilogue).
- **longjmp**: 33 native instructions = 132 cycles (29 base + 4 for the FT-stack save/restore around the BC restore).
- Round-trip: 66 instructions = 264 cycles.

### Why the Order Is FT, DE, BC, HL

`LCR (C),T` writing `SP_BC` changes where the BC stack lives. Any `PUSH BC` or `POP BC` around the BC restore would operate on the wrong stack — the SP has moved. But the BC restore also needs `LD C,RC8_SP_BC`, which clobbers `Stack[SP_BC]` (the current top). If the restored SP equals the current SP, `BC = Stack[SP_BC]` reads the corrupted slot. The restore therefore saves the buffer pointer on the FT stack (already restored) before the `LCR` and recovers it afterward, avoiding any reliance on `Stack[SP_BC_setjmp]`.

HL must be restored last because `J (HL)` returns through `HL`. Restoring HL earlier would leave `HL = return address` live across subsequent `LD C` / `LCR` operations that clobber BC — harmless to HL, but ordering HL last keeps Phase 1 monotonic (no restore is ever undone by a later one).
