# Computed Branch via FLAGS Table

Dispatches an input byte to one of several **code handlers** using a 256-entry lookup table in code space. Each table entry holds a precomputed `FLAGS_*` byte; `LCO T,(FT)` fetches the byte for the input value, `LD F,T` injects it directly into the flags register, and a `J/CC` chain jumps to the matching handler.

This is a *computed branch*, not a value lookup. The point of the table is to pick which **routine** runs next — each category executes qualitatively different code. A plain category table that just stored `0,1,2,3` could return a number but could not, by itself, transfer control to a handler; it would still need a post-fetch dispatch (a `CMP`/`J/CC` cascade or a jump table) to reach the routine. The FLAGS table bridges that gap in one fetch + a short condition chain, with the dispatch cost independent of how irregular the byte→category mapping is.

### Use Case: Serial-Protocol Opcode Dispatcher

A serial-protocol byte is dispatched to one of four handlers — control/NOP, read, write, error — each of which sets up a different transfer. The opcode allocation is deliberately disjoint (legacy extended ranges live at `$80+`):

| Category | Byte ranges | Why distinct code |
|----------|--------------|-------------------|
| Control/NOP | `$00`, `$20-$2F` | send ACK; sub-command in low nibble |
| Read | `$01-$0F`, `$80-$8F` | set up read DMA; channel = `C & $0F` |
| Write | `$30-$3F`, `$90-$9F` | set up write DMA; channel = `C & $0F` |
| Error | everything else | raise protocol error; log opcode |

Read and Write each span **two disjoint ranges**, so a naive `CMP`/`J/CC` cascade needs at least two comparisons per category (bound-check both ranges) — eight or more range checks total. The table absorbs all of that irregularity into 256 bytes for one fetch plus the same three-branch chain regardless.

### FLAGS Encoding

Each category is assigned a `FLAGS_*` value whose set bits are distinct enough to be tested by a short `J/CC` chain ordered most-specific-first. Three "primary" composite `J/CC` conditions (`J/EQ` Z, `J/LTU` C, `J/LT` N⊕V) plus a fallthrough bound the robust chain to four categories; `$08` (`FLAGS_OVERFLOW`) is not a fourth distinguishable slot since it sets N⊕V=1 just like `$04` and would collide under `J/LT` (see [Limit of the Technique](#limit-of-the-technique)):

| Category | FLAGS value | Bits (C,Z,N,V) | J/CC that takes it |
|----------|------------|-----------------|---------------------|
| Control/NOP | `$02` (`FLAGS_EQ`) | 0,1,0,0 | `J/EQ` (Z=1) |
| Read | `$01` (`FLAGS_LTU`) | 1,0,0,0 | `J/LTU` (C=1, Z=0) |
| Write | `$04` (`FLAGS_NEGATIVE`) | 0,0,1,0 | `J/LT` (N=1, Z=0, C=0) |
| Error | `$00` (`FLAGS_NE`) | 0,0,0,0 | none — falls through to the catch-all |

Because one `F` value can satisfy multiple conditions (e.g., `$03` satisfies both `EQ` and `LTU`), the chain tests the most-specific flag first. No `FLAGS_*` value in this table triggers more than one branch:

- `$02` → only `J/EQ` (Z=1; C=0 so `J/LTU` skips)
- `$01` → only `J/LTU` (Z=0 so `J/EQ` skips; C=1)
- `$04` → only `J/LT` (Z=0, C=0 so the first two skip; N=1)
- `$00` → all skip → fallthrough

### Register Allocation

| Register | Role | Rationale |
|----------|------|-----------|
| BC | Input value / table index | Callee-saved; `LD B,0` zero-extends `C` so callers may leave `B` garbage; `ADD FT,BC` indexes the table; `C` is preserved so handlers can read the sub-opcode nibble |
| FT | Table pointer | Caller-saved; `LD FT,i16` loads the base, `ADD FT,BC` offsets by the input, `LCO T,(FT)` fetches |
| T | Fetched flags byte / return status | Hub register; loaded by `LCO`, reused as the return value |
| F | Injected flags | Set by `LD F,T`; tested by the `J/CC` chain |

### Table Layout (256 bytes)

Counts sum to 256: 1 + 15 + 16 + 16 + 16 + 64 + 16 + 16 + 96.

```
OPCODE_TABLE:
        db      FLAGS_EQ          ; $00       : control/NOP (1)
        db      FLAGS_LTU         ; $01-$0F   : read (15)
        db      FLAGS_NE          ; $10-$1F   : error (16)
        db      FLAGS_EQ          ; $20-$2F   : control family (16)
        db      FLAGS_NEGATIVE    ; $30-$3F   : write (16)
        db      FLAGS_NE          ; $40-$7F   : error (64)
        db      FLAGS_LTU         ; $80-$8F   : read extended (16)
        db      FLAGS_NEGATIVE    ; $90-$9F   : write extended (16)
        db      FLAGS_NE          ; $A0-$FF   : error (96)
```

### Full Assembly

```
; --
; -- dispatch_opcode
; --   Dispatches a serial-protocol opcode byte to one of four handlers
; --   (control/NOP, read, write, error) using a 256-entry FLAGS_* table in
; --   code space, looked up via LCO and injected into F for a J/CC chain.
; --
; --   Each category maps to distinct transfer-setup code, not a constant;
; --   the FLAGS table is the bridge between a per-byte code-space entry and
; --   a per-category branch — a plain category table cannot branch by itself.
; --
; -- Inputs:
; --   C:  opcode byte (0-255); B is ignored (zeroed internally)
; --
; -- Outputs:
; --   T:  handler status ($00 = control/ack, $81 = read pending,
; --        $82 = write pending, $FF = error)
; --
        SECTION "dispatch_opcode",CODE
dispatch_opcode:
        push    bc              ; save BC (callee-saved)
        ld      b,0             ; B = 0 -> BC = opcode (zero-extend C)

        ld      ft,OPCODE_TABLE ; FT = table base (synthesized: ld f,hi / ld t,lo)
        add     ft,bc           ; FT = table + opcode
        lco     t,(ft)          ; T = FLAGS_* byte from code space
        ld      f,t             ; inject flags into F

        j/eq    .control        ; Z=1: control/NOP
        j/ltu   .read           ; C=1, Z=0: read
        j/lt    .write          ; N!=V (N=1): write
        j       .error          ; fallthrough: error

.control
        ; (real driver: send ACK; sub-command = C & $0F)
        ld      t,0             ; status = control/ack
        j       .done

.read
        ; (real driver: tail-call dma_read with channel = C & $0F)
        ld      t,$81           ; status = read pending
        j       .done

.write
        ; (real driver: tail-call dma_write with channel = C & $0F)
        ld      t,$82           ; status = write pending
        j       .done

.error
        ; (real driver: raise protocol error, log opcode in C)
        ld      t,$FF           ; status = error

.done
        pop     bc              ; restore BC (C preserved throughout)
        j       (hl)            ; return
```

### Edge-Case Traces

| Input | Table entry | F value | Chain path | Result |
|-------|------------|---------|------------|--------|
| `$00` | `FLAGS_EQ` (`$02`) | Z=1 | `J/EQ` taken → `.control` | `T = $00` |
| `$05` | `FLAGS_LTU` (`$01`) | C=1 | `J/EQ` skip, `J/LTU` taken → `.read` | `T = $81` |
| `$20` | `FLAGS_EQ` (`$02`) | Z=1 | `J/EQ` taken → `.control` | `T = $00` |
| `$35` | `FLAGS_NEGATIVE` (`$04`) | N=1 | skip, skip, `J/LT` taken → `.write` | `T = $82` |
| `$10` | `FLAGS_NE` (`$00`) | all clear | all skip → `.error` | `T = $FF` |
| `$80` | `FLAGS_LTU` (`$01`) | C=1 | `J/EQ` skip, `J/LTU` taken → `.read` | `T = $81` |
| `$95` | `FLAGS_NEGATIVE` (`$04`) | N=1 | skip, skip, `J/LT` taken → `.write` | `T = $82` |
| `$A0` | `FLAGS_NE` (`$00`) | all clear | all skip → `.error` | `T = $FF` |
| `$FF` | `FLAGS_NE` (`$00`) | all clear | all skip → `.error` | `T = $FF` |

### Why This Beats the Alternatives

- **Plain category table + `LCO T,(FT)` → return `T`.** Insufficient when each category must run distinct routine code (call `dma_read`, call `dma_write`, send ACK, raise error). A byte can hold a number, not a call; after the fetch you'd still need a nested `CMP`/`J/CC` cascade or a jump table to reach the code. The FLAGS table collapses the fetch and the dispatch into one mechanism: the looked-up byte *is* the dispatch decision.
- **`CMP`/`J/CC` cascade.** Disjoint ranges force ≥2 comparisons per category: read needs both `≥$01 & ≤$0F` and `≥$80 & ≤$8F`; write needs both `$30-$3F` and `$90-$9F`. Eight or more range checks plus the carry-save gymnastics around mixing 8-bit compares with 16-bit pointer housekeeping. The table collapses all disjointness into 256 bytes.
- **FLAGS table.** One code-space fetch plus three conditional branches; complexity is independent of the number of disjoint ranges per category.

### Contrast: Jump Table

A 16-bit jump table in code space (`DW` entries) is the natural alternative when categories exceed the 4-slot FLAGS ceiling. The two share the spirit (O(1) dispatch via code-space lookup, scale-invariant to mapping irregularity) but the trade-offs differ:

| Dimension | FLAGS table | Jump table |
|---|---|---|
| Table size | 256 bytes (1 byte/entry) | 512 bytes (2 bytes/entry) |
| Step count | `LCO` + `LD F,T` + `J/CC` chain (3 + chain) | `LS FT,1` (double index) + `LD HL,(FT+)` + `J (HL)` |
| Index doubling | none (1 byte/entry → `ADD FT,BC`) | required (no `ADD BC,BC`; double via `LD FT,BC`/`LS FT,1`/`LD BC,FT`) |
| Categories | ≤4 (three primary `J/CC` + fallthrough) | unlimited |
| Handler addressing | indirect via a `J/CC` label in the dispatcher | each entry stores the handler's own address |
| HL clobber | none (`J (HL)` is only the final return) | `LD HL,(FT+)` clobbers HL; the dispatcher's own `PUSH HL` return path must be preserved separately |
| Assemble-time irregular ranges | one `db FLAGS_x` per byte (cheap to fill holes) | one `dw handlerN` per byte (still cheap with `DW`, but twice the byte cost of FLAGS) |

**Pick the jump table when:** handlers number >4, or per-byte distinctness is essential (each of 256 bytes can route to its very own routine, with no shared category). Discard the FLAGS bit-budgeting constraint entirely.

**Pick the FLAGS table when:** handlers are 4 or fewer, and the byte→handler grouping is irregular. The 256-byte table is half the jump-table's size, the index needs no doubling, and `J/CC` is the dispatch primitive (no HL round-trip).

**Practical hybrid:** if a fifth category is a future possibility, design from day one with a jump table indexed by a plain category byte — the byte→category mapping moves into a 1-byte table (`LCO T,(FT)`) and the 4-category FLAGS ceiling disappears, at the cost of 256 bytes (category) + 4-8 bytes of jump table + the HL clobber bookkeeping.

### When the Table Wins

The table approach pays off when:
- The byte→category mapping is **disjoint or irregular** (each disjoint range adds cost to a `CMP` cascade; the table absorbs it for free).
- Up to four dispatch targets (the FLAGS-table ceiling; see [Limit of the Technique](#limit-of-the-technique) below). Beyond that, use a [jump table](#contrast-jump-table).
- Each category runs **distinct code**, so a constant-return table is not enough — the table's job is to *branch*, not to *classify*.
- The same table serves multiple dispatch sites (amortized 256-byte cost).

### Register Safety Audit

Traced against the Native Instruction Table:

- `LD B,0` writes `B` only; `C` is preserved (handlers may read `C & $0F`).
- Zeroing `B` before `ADD FT,BC` prevents any high-byte contamination of the table index — see the carry-propagation warning in [AGENTS.md](../AGENTS.md): `ADD FT,BC` reads `B/C`; with `B=0` only the low byte of `BC` contributes and no high-byte carry leaks into `FT`.
- `LD FT,i16` (synthesized `LD F,hi` / `LD T,lo`) writes `F/T`; `BC` untouched.
- `ADD FT,BC` reads `F/T/B/C`, writes `F/T`; `BC` untouched.
- `LCO T,(FT)` reads `F/T`, writes `T`; `F` is not clobbered, but we overwrite it next anyway.
- `LD F,T` reads `T`, writes `F`; leaves `T` intact (the `Writes` column lists only `F`).
- `J/CC` / `J` read `F`, write no registers — the injected flags survive the entire chain.
- `LD T,N` in each handler writes `T` (clobbering the 16-bit `FT` view) but does not modify `F` — see [Flags survival across T/F writes](../REFERENCE.md#flags-survival-across-tf-writes).
- `POP BC` restores the original `BC`; `J (HL)` returns.

`PUSH BC` / `POP BC` are balanced (one save, one restore); no callee-saved register leaks.

### Flags Survival Note

`LD F,T` writes `F` and leaves `T` intact (the `Writes` column of `LD F,T` lists only `F`). The subsequent `J/CC` instructions read `F` and write no registers, so the injected flags survive the entire chain. The `LD T,N` in each handler writes `T` (clobbering the 16-bit `FT` view) but does not modify `F` — see [Flags survival across T/F writes](../REFERENCE.md#flags-survival-across-tf-writes). The flags are therefore still valid if a handler needs to re-test them, though this routine does not.

### Limit of the Technique

The robust construction distinguishes four categories via three "primary" composite `J/CC` conditions plus a fallthrough:

| Slot | Condition | FLAGS value |
|------|-----------|-------------|
| 1 | `J/EQ` (Z=1) | `$02` `FLAGS_EQ` |
| 2 | `J/LTU` (C=1, Z=0) | `$01` `FLAGS_LTU` |
| 3 | `J/LT` (N⊕V=1, Z=0, C=0) | `$04` `FLAGS_NEGATIVE` (V=0) |
| 4 | fallthrough | `$00` `FLAGS_NE` |

Note that `J/LT` tests **N⊕V**, not N alone — so `$08` (`FLAGS_OVERFLOW`, V=1 N=0) also makes N⊕V=1 and would collide with `$04` under `J/LT`. It is not a fresh distinguishable slot.

Going beyond four categories is possible in principle — the ISA provides composite `J/CC` conditions (`J/LE`, `J/LEU`, `J/GT`, `J/GTU`, `J/GE`, `J/NE`) — but each FLAGS value must satisfy exactly one condition in the ordered chain, and bit budgeting becomes fragile fast. The practical fallback is a 16-bit jump table in code space — see [Contrast: Jump Table](#contrast-jump-table) above.