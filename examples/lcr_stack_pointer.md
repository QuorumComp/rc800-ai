# LCR Stack Pointer Manipulation

The configuration registers `RC8_SP_FT`, `RC8_SP_BC`, `RC8_SP_DE`, and `RC8_SP_HL` hold the current stack pointer for each register stack. Accessing them via `LCR` lets you read and write the stack pointer directly, bypassing `PUSH`/`POP`/`SWAP` entirely.

### Why It Matters

`POP` removes one entry per instruction (4 cycles). Discarding 15 entries takes 15 `POP`s — 60 cycles. Reading the SP, adding an offset, and writing it back takes 3 instructions — 12 cycles. The same technique works for truncating deeply nested stack frames.

### The Register Is a Live View of the Stack Top

The register pair is not separate storage from its stack — it is a window into the entry at the current SP. `PUSH` decrements SP and stores the value at the new position; `POP` increments SP and loads the new top into the register. Writing the SP via `LCR` is no different: **the register content changes to reflect whatever entry the new SP points to.** After any SP adjustment, the register holds `Stack[new_SP]`, not its previous value. Treat the register as clobbered by every `LCR (C),T` that writes the SP.

The stack grows downward: SP starts at `$FF` (empty) and each `PUSH` decrements it toward `$00`. So `$FF` means 0 entries, `$FE` means 1 entry, and `$00` means 255 entries.

### Discard N Entries

```
        ; Discard 15 entries from the DE stack
        ld      c,RC8_SP_DE     ; C = $02 (DE SP config register index)
        lcr     t,(c)           ; T = current DE stack pointer
        add     t,15            ; advance pointer past 15 entries
        lcr     (c),t           ; write back — entries gone, DE = Stack[new_SP]
```

After the write, `DE` reflects the entry 15 positions below the old top. The discarded entries remain in stack memory but are no longer reachable through `PICK`/`SWAP`/`POP` (which index relative to the new SP). This is safe to use with interrupts enabled: an interrupt handler that pushes and pops balanced entries on this stack writes into the freed region temporarily but restores SP on return, so the reachable entries below the new SP are unaffected.

### Restore N Entries (Rewind)

```
        ; Restore 3 previously discarded entries on the BC stack.
        ; DANGEROUS with interrupts enabled — see below.
        di                      ; required: no interrupt may touch the BC stack
        ld      c,RC8_SP_BC     ; C = $01 (BC SP config register index)
        lcr     t,(c)           ; T = current BC stack pointer
        sub     t,3             ; rewind pointer 3 entries
        lcr     (c),t           ; write back — old entries reachable again
        ei
```

Rewinding decrements SP, making previously discarded entries reachable again. **This is only safe if no interrupt (or any code it calls) can push onto this register stack between the discard and the rewind.** An interrupt handler's `PUSH` writes into the freed region and overwrites the "saved" entries; even a balanced `PUSH`/`POP` leaves the handler's value in memory at the freed address, so the rewind exposes garbage, not the original data. Disable interrupts (`DI`) around the discard+rewind window, or use a stack that no interrupt handler touches.

### Empty a Stack

```
        ; Completely empty the FT stack (discard everything)
        ld      c,RC8_SP_FT     ; C = $00 (FT SP config register index)
        ld      t,$FF           ; $FF = empty stack boundary
        lcr     (c),t           ; FT stack cleared; FT = Stack[$FF] (undefined)
```

After writing `$FF`, `FT` reflects `Stack[$FF]`, which is whatever junk occupies the empty boundary — do not rely on the register value until it is reloaded with a known value.

### Stack Depth Query

```
        ; Measure how many entries are on the HL stack
        ld      c,RC8_SP_HL     ; C = $03 (HL SP config register index)
        lcr     t,(c)           ; T = current HL stack pointer
        ; T = SP: $FF = empty (0 entries), $00 = full (255 entries).
        ; Number of entries = $FF - T:
        not     t               ; T = ~SP = $FF - SP = stack depth
```

`PUSH` decrements SP from `$FF` downward, so the raw SP value is inverted relative to depth: `$FF` means empty, `$00` means 255 entries. `NOT T` (synthesized as `XOR T,$FF`) converts the SP value to the entry count in one instruction.

### Safety

- **Bounds**: each stack is 256 words deep. The SP values `$00`–`$FF` are valid. Writing a value outside this range is undefined behavior.
- **Lower/upper bounds**: `RC8_SP_LOW` and `RC8_SP_HIGH` define the stack boundaries. Writing an SP value outside these bounds may trigger a stack overflow/underflow interrupt.
- **Register coherence**: the register is a live view into `Stack[SP]`. Writing the SP via `LCR (C),T` changes the register content to reflect the new top — the register does not retain its previous value. Treat any SP write as clobbering the register pair.
- **Interrupt safety**: discarding entries (advancing SP) is safe with interrupts enabled. Rewinding (decreasing SP) to restore discarded entries is **not** safe with interrupts enabled — an interrupt handler's `PUSH` overwrites the freed entries. Disable interrupts around any discard+rewind window.
- **Clobber considerations**: `LCR T,(C)` writes `T` (clobbers `FT`). `LCR (C),T` reads `T` and `C` but writes no registers (though it changes the target register stack's top). If `FT` is live, save it before reading the SP. Load `C` with `LD C,i8` before any `LCR` that targets `T`.
