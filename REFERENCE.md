# RC800 Architecture Reference

## RC800 Architecture Background

The RC800 architecture is an experiment that explores what a hypothetical 8-bit CPU introduced in the mid to late 1980s might look like, if certain dogmas such as FWIW and RISC techniques had been applied.

It is inspired by several existing architectures, such as the Z80, MIPS, and 68000. The ISA has been designed to fit into a four-stage RISC pipeline. Internally, it has eight 8-bit registers. These are named `F` (for `F`lag), `T` (`T`emporary), `B`, `C`, `D`, `E`, `H`, and `L`. They can also be combined to form four 16-bit registers: `FT`, `BC`, `DE`, and `HL`. There is an extensive set of instructions that perform 16-bit operations, such as comparisons, shifts, and addition. There is also a program counter, interrupt enable flag, stack pointers, and a set of configuration registers.

The architecture features on-chip stacks, one per register pair. All registers can be pushed at once, enabling very fast interrupt response.

The first implementation is called RC811.

## Introduction

The RC811 (first implementation of the RC800 architecture) is an 8-bit CPU where every instruction completes in exactly 4 clock cycles. No pipelining, no variable latency — just predictable, deterministic execution.

What makes it unusual:

- **Four independent register stacks** — BC, DE, HL, and FT each have their own 256-word on-chip stack. Push and pop any register in any order; there is no shared stack to corrupt.
- **T/FT hub architecture** — all data flows through T (8-bit) or FT (16-bit). This sounds restrictive but simplifies the instruction set and makes register clobbering obvious.
- **Synthesized instructions** — the assembler fills gaps in the native instruction set (16-bit immediates, post-increment addressing, conditional execution) with safe, side-effect-free expansions.

This document covers everything from writing your first instruction to understanding the full instruction set. Start with [Quick Start](#quick-start) to get coding, then refer to later sections as needed.

## Quick Start

### Register Names
- 8-bit: `F`, `T`, `B`, `C`, `D`, `E`, `H`, `L`
- 16-bit: `FT`, `BC`, `DE`, `HL`

### Register Aliasing

16-bit pairs are views into adjacent 8-bit registers — not separate storage. Writing an 8-bit register silently modifies the containing pair. Z80 and x86 programmers will be familiar with this (for example, x86's `AX`/`AH`/`AL`). See [Register Aliasing](#register-aliasing-1) in the Register File section for the exact clobber-checking rules and a guide to reading the Native Instruction Table.

### The T Hub

All data flows through T (8-bit) or FT (16-bit). There are no register-to-register moves between any other pair — T/FT is the hub:

```
        ld      t,b             ; T <- B  (move to hub)
        ld      c,t             ; C <- T  (move from hub)
        ; effectively: C = B

        ld      ft,bc           ; FT <- BC  (16-bit move to hub)
        ld      de,ft           ; DE <- FT  (16-bit move from hub)
        ; effectively: DE = BC
```

Every arithmetic, logic, and comparison instruction targets T or FT. Think of T/FT as the accumulator: data goes in, gets processed, comes out.

A few common operations break this rule: `ADD R8, i8`, `CMP R8, i8`, and `LD R8, i8` can target any 8-bit register directly. Similarly, `ADD R16, s8` and `ADD R16, i16` work on any 16-bit register. These are the exceptions; most operations require T or FT. See the [Register Transfer](#register-transfer) section for the full constraint and the exact allowed transfer patterns.

### Instruction Categories

**T-centric operations** — most instructions operate on T:

| Category | Example | Effect |
|----------|---------|--------|
| Arithmetic | `add t,b` / `sub t,5` | T = result |
| Bitwise | `and t,b` / `or t,$ff` | T = result |
| Comparison | `cmp t,b` | F = flags (T unchanged) |
| Load | `ld t,(bc)` / `ld t,42` | T = value |
| Store | `ld (bc),t` | memory = T |

**16-bit operations** — operate on FT or use FT as a pointer:

| Category | Example | Effect |
|----------|---------|--------|
| Arithmetic | `add ft,bc` / `add de,1` | result in left operand |
| Load | `ld ft,bc` / `ld ft,$1234` | FT = value |
| Pointer math | `add ft,bc` / `add bc,1` | FT/BC = new address |
| Memory via FT | `ld t,(ft)` / `ld (ft),t` | T ↔ memory[FT] |

**Flags:** Only `CMP` and `TST` set the F register. Arithmetic, bitwise, load, store, and move instructions do not update flags. There is no implicit flag setting on this architecture.

**Flow control:**

| Instruction | Effect |
|-------------|--------|
| `j label` | Unconditional jump |
| `j/cc label` | Conditional jump (10 conditions) |
| `dj r8,label` | Decrement and jump if non-zero |
| `jal label` | Call; return address stored in HL |
| `j (hl)` | Return; jump to address in HL |

### Using Registers as Pointers

Any 16-bit pair can hold an address. Load a pointer, then use indirect addressing:

```
        ld      bc,$1000        ; BC = pointer to memory
        ld      t,(bc)          ; T = first byte
        add     bc,1            ; advance pointer
        ld      t,(bc)          ; T = second byte
```

Post-increment makes this more compact:

```
        ld      bc,$1000
        ld      t,(bc+)         ; T = first byte, BC points to second
        ld      t,(bc+)         ; T = second byte, BC points to third
```

FT is also a pointer — and unlike BC/DE/HL, it can do 16-bit arithmetic directly:

```
        ld      ft,$1000        ; FT = pointer
        add     ft,de           ; FT += offset
        ld      t,(ft)          ; T = base[offset]
```

### Minimal Program

```
        SECTION "main",CODE
main:
        ld      bc,$1234        ; pointer
        ld      t,42            ; value
        lio     (bc),t          ; write to I/O
        j       main            ; loop forever
```

## Assembly

### Source Style

- The assembler is case-insensitive for both mnemonics and directives.
- Mnemonics and register names are commonly written in **lowercase** (e.g., `ld t,(bc+)`).
- Assembler directives are commonly written in **UPPERCASE** (e.g., `SECTION`, `GLOBAL`, `DB`).
- No space after the comma separator, for both instructions and directives (e.g., `ld t,(bc+)`, `SECTION "reset",CODE[0]`).
- Indentation uses **8 spaces** (not tab characters). A single 8-space indent separates the mnemonic from its operands, aligning operands at column 9. Labels and directives follow the same alignment.
- Blank lines delineate logical blocks: place a blank line after the function prologue, before the epilogue, and after `DJ` instructions that terminate a loop. This visually separates setup, main logic, and teardown.

### Literals

- Hexadecimal: `$1234` or `0x1234`
- Decimal: `42`
- Signed: `-87`

### Labels

- Global labels: `Label` (starts at column 0)
- Local labels: `.local` (scoped to enclosing global label)
- PC-relative: `@+67` (address relative to current PC)

### Register List Syntax

- **Range syntax** (`-`): `push bc-hl` pushes bc, de, hl (expands to `pusha` / `pop ft`). `pop bc-hl` pops them (expands to `popa` / `push ft`).
- **List syntax** (`/`): `push bc/de` pushes only bc and de individually (two `push` instructions). `pop bc/de` pops them in reverse order.
- Any combination is accepted: `push ft/hl`, `push bc-de`, etc.

### Addressing Forms

- Indirect: `(FT)`, `(BC)`, `(DE)`, `(HL)` — any 16-bit register pair
- Post-increment: `(FT+)`, `(BC+)`, `(DE+)`, `(HL+)` — increments the register after the access (synthesized)
- Pre-decrement: `(-FT)`, `(-BC)`, `(-DE)`, `(-HL)` — decrements the register before the access (synthesized)
- Store: `LD (R16), T` or `LD (R16+), T`
- Load: `LD T, (R16)` or `LD T, (R16+)`

### I/O and Code Access

- Store: `LIO (R16), T` — any 16-bit pair (FT, BC, DE, HL)
- Load: `LIO T, (R16)` — any 16-bit pair
- Synthesized addressing: `(R16+)`, `(-R16)` also supported for LIO and LCO

### Directives

- `SECTION "name",CODE` — define a code section (use `CODE` without an address for user-land code; use `CODE[x]` only for interrupt handlers and BIOS at fixed addresses)
- `EQU` — equate a symbol to a value
- `DB` — define byte(s)
- `DW` — define word(s) (16-bit)
- `DS` — define (reserve) storage
- `MACRO` / `ENDM` — macro definition
- `IMPORT` — reference a symbol defined in another source file
- `GLOBAL` — export a symbol for use by other files

## Architecture Overview

### Pipeline Stages

1. **Decode/Register Read** — Opcode decoded, registers read
2. **Memory** — Memory/I/O access, trailing byte fetch at PC+1
3. **ALU/PC** — Arithmetic, logical operations, PC calculation
4. **Fetch/Register Write** — Write results, fetch next instruction

Memory stage precedes ALU (unusual for RISC) because 8-bit opcodes cannot carry immediates. A trailing byte is fetched at PC+1 in the memory stage and passed to the ALU.

### Opcodes

All opcodes are 8 bits. Instructions that appear to be 2 bytes wide actually fetch the second byte from memory at PC+1 during the memory stage. Any opcode with a second byte has the form `10------`, enabling trivial instruction prefetch.

### Addressing Modes

The 8-bit opcode space limits the number of available addressing modes. An instruction can take at most a single 8-bit immediate value as an optional operand, fetched at PC+1 during the memory stage. There is no native support for 16-bit immediates inline with an instruction.

To work with 16-bit values, the programmer must either:
- Load the value into a register first using separate byte loads, then use the register in the target instruction.
- Use a **synthesized instruction**, which the assembler expands into a sequence of native instructions that produces the intended result.

Synthesized instructions have no side effects beyond their documented result — they do not modify flags, alter registers, or change state in any way other than the intended outcome.

### Memory and I/O

- 16-bit address bus, 8-bit data bus: 64 KiB address space
- Three access-type signals can expand to 384 KiB:
  - **CODE** — asserted for instruction fetch or `LCO` instruction
  - **IO** — asserted for `LIO` instruction
- **SYS** — asserted for interrupt handling or `SYS` instruction
- Endianness is not defined by the CPU; the system designer chooses

### Section Alignment

Sections are not aligned to any particular multiple.

## Register File

### General-Purpose Registers

| Name | Role |
|------|------|
| F | Flag register (holds comparison results), otherwise usable as a normal register. **Writable**: can be loaded directly with `LD F, i8` to set condition codes artificially (e.g., `LD F, FLAGS_EQ` to force "equal"). |
| T | Temporary/accumulator — most operations target T |
| B, C, D, E, H, L | General-purpose |

### F Register Tricks

The F register is fully writable, not just set by the ALU. This enables several patterns:

- **Force condition codes**: `LD F, 0` sets EQ/Z, `LD F, $FF` sets NE/NZ.
- **F as carry lane for byte math**: There is no native "add with carry" instruction and no arithmetic instruction that updates the F register as a flag side effect. Only `CMP` and `TST` set F, and they do not produce an arithmetic result. To obtain a carry-out from an addition, use a 16-bit `ADD` on zero-extended 8-bit operands.

  Treat `FT` as a 16-bit temporary where `T` is bits 7:0 and `F` is bits 15:8. To add two unsigned 8-bit numbers in `T` and `C`:

  ```
          ; Inputs: T=a, C=b, B=0
          ; Outputs: T=(a+b) mod 256, F=carry-out (0 or 1)
          ld      f,0             ; FT = $00:a
          add     ft,bc           ; FT = $00:a + $00:b
                                  ; T = low byte of sum, F = carry
  ```

  `ADD FT, BC` is therefore the architecture's "add with carry-out" instruction. The carry is written into `F` as a normal register value, not as a flag side effect, but it can be used exactly like a carry flag. Read it back with `LD T,F` (the standard 8-bit T-hub path, 1 instruction) — do not use `LD BC,FT` (a 16-bit transfer that clobbers `BC`) just to reach `F`.

  This extends to add-with-carry. To add `a_i` (in `T`), `b_i` (in `C`), and an incoming carry, add the carry into the low byte first with a second 16-bit add:

  ```
          ; Inputs: T=a_i, C=b_i, E=carry_in (0 or 1), D=0, B=0
          ; Outputs: T=(a_i+b_i+carry_in) mod 256, F=carry_out
          ld      f,0             ; FT = $00:a_i
          add     ft,de           ; FT = $00:a_i + $00:carry_in
          add     ft,bc           ; FT += $00:b_i
  ```

  Because `F` is overwritten by `CMP` and `TST`, the carry is most useful when consumed immediately or saved to a callee-saved register before any comparison.

  For a complete multi-precision addition loop, see [add_mp](examples/add_mp.md).

- **Computed branch via FLAGS table**: `LD F,T` can inject a precomputed flags byte fetched from a code-space table via `LCO T,(FT)`, then a `J/CC` chain dispatches to a handler. See [computed_branch](examples/computed_branch.md). The robust construction distinguishes four categories using three "primary" composite conditions plus a fallthrough:

  | Slot | Condition | FLAGS value |
  |------|-----------|-------------|
  | 1 | `j/eq` (Z=1) | `$02` `FLAGS_EQ` |
  | 2 | `j/ltu` (C=1, Z=0) | `$01` `FLAGS_LTU` |
  | 3 | `j/lt` (N⊕V=1, Z=0, C=0) | `$04` `FLAGS_NEGATIVE` (V=0) |
  | 4 | fallthrough | `$00` `FLAGS_NE` |

  Order the chain most-specific-first: a multi-bit FLAGS value satisfies several conditions (e.g. `$03` takes both `j/eq` and `j/ltu`), so the first matching condition must be the most exclusive. `$08` (`FLAGS_OVERFLOW`) is not a new distinguishable slot — it sets N⊕V=1 just like `$04`, so it collides under `j/lt`. Going beyond four categories requires the composite `J/CC` conditions (`j/le`, `j/leu`, `j/gt`, `j/gtu`, `j/ge`, `j/ne`), at the cost of fragile bit budgeting, or a 16-bit jump table in code space.

**Note**: F is primarily a flags register. Although it can be used as a normal register, its contents are overwritten by `CMP` and `TST`, so any value stored in F is only reliably available for the brief window between writes and the next comparison. In practice, F-as-data is most useful inside tight loops where no comparison occurs between the value being produced and consumed.

### 16-Bit Register Pairs

Formed by combining adjacent 8-bit registers:

| Pair | Bits 15:8 | Bits 7:0 | Notes |
|------|-----------|----------|-------|
| FT | F | T | Used for 16-bit arithmetic, shifts, comparisons. **Primary multi-word register**: the FT stack is the standard location for 32-bit and larger values. |
| BC | B | C | Commonly used for I/O port addressing |
| DE | D | E | |
| HL | H | L | Holds return address for `JAL` |

Each pair is a 16-bit view into the same physical storage as its two 8-bit halves.

### Register Aliasing

The 16-bit pairs `FT`, `BC`, `DE`, and `HL` are **not separate storage** from the 8-bit registers `F`, `T`, `B`, `C`, `D`, `E`, `H`, and `L`. Each pair is a 16-bit view of the same physical 16-bit register. The first-named register holds bits 15:8, and the second-named register holds bits 7:0:

```
        16-bit register FT
   ┌─────────┬─────────┐
   │    F    │    T    │
   │  15:8   │   7:0   │
   └─────────┴─────────┘
```

The same layout applies to `BC`, `DE`, and `HL`.

Because the pair and its constituent bytes are the same hardware register, any write to an 8-bit register immediately changes the 16-bit view, and any write to a 16-bit pair immediately changes the two 8-bit registers it overlaps.

#### Clobber Rule

> **Writing any 8-bit register changes the value of its parent 16-bit pair (the other byte keeps its old value, but the pair as a whole is no longer meaningful). Writing any 16-bit pair changes both of its 8-bit halves.**

Use this rule to check every instruction that writes a register. If the parent pair is currently live, the instruction clobbers it.

> **WARNING — `FT`, `T`, and `F` are the same physical register.**
>
> Any instruction that writes `FT` also writes `T` and `F`. Any instruction that writes `T` or `F` changes the corresponding byte of `FT`. There is no "abstract 16-bit value" separate from its two bytes. Operations such as `SWAP FT`, `LD BC,FT`, `ADD FT,BC`, and `LD FT,BC` all modify the physical `T` and `F` registers.

A useful mental model: whenever you see `FT`, read it as `F:T`. `ADD FT,BC` is really `ADD F:T, B:C`. `SWAP FT` really swaps the two bytes `F` and `T` with the stack word. Thinking this way makes it impossible to forget that a 16-bit operation on `FT` destroys whatever was in `T` or `F`.

#### Register Membership Lookup

| 8-bit register | Parent pair |
|---|---|
| `F` | `FT` |
| `T` | `FT` |
| `B` | `BC` |
| `C` | `BC` |
| `D` | `DE` |
| `E` | `DE` |
| `H` | `HL` |
| `L` | `HL` |

Conversely:

| 16-bit pair | High byte | Low byte |
|---|---|---|
| `FT` | `F` | `T` |
| `BC` | `B` | `C` |
| `DE` | `D` | `E` |
| `HL` | `H` | `L` |

#### What this means in practice

```
        ; BC = $1234
        ld      c,0             ; C changes, so BC is now $1200
        ; B is still $12, but the full 16-bit value of BC is now $1200
```

On a machine with independent registers, writing `C` would not affect `BC`. On the RC800 architecture, it does.

#### Examples of aliasing bugs

- `LD T, (BC)` reads from memory into `T`. Because `T` is the low byte of `FT`, the `FT` pair is no longer meaningful as a 16-bit value. `F` itself is not modified, but any live 16-bit value in `FT` is destroyed.
- `ADD B, i8` modifies `B`, which is the high byte of `BC`. The `BC` pair is now partially corrupted.
- `PUSH BC` saves both `B` and `C`, but may also use `FT` internally if the assembler synthesizes the instruction.

**Rule of thumb**: every instruction that writes an 8-bit register clobbers the containing 16-bit pair as a whole. The other byte of the pair keeps its old value, but the 16-bit value is no longer meaningful. Check the [Native Instruction Table](#native-instruction-table) to see which 16-bit pairs are affected by each instruction; the Clobbers column lists the parent pair for every written 8-bit register.

#### Common aliasing traps

- **Any instruction that writes `FT` also writes `T` and `F`.** `SWAP FT`, `LD BC,FT`, `ADD FT,BC`, `ADD FT,DE`, `LS FT,R8`, etc. all modify the physical `T` and `F` registers.
- **Any instruction that writes `T` or `F` clobbers `FT`.** `LD T,R8`, `LD F,i8`, `CMP T,R8`, `CMP R8,i8`, and `DJ T` all write one byte of `FT`, so the old 16-bit value of `FT` is no longer valid.
- **16-bit transfers clobber both bytes of the destination.** `LD BC,FT` writes both `B` and `C`.
- **The `SWAP FT` instruction is not a logical pointer swap.** It exchanges the full 16-bit value `F:T` with the top of the FT stack, so `T` and `F` are overwritten.
- **The `Clobbers` column in the Native Instruction Table means the parent pair is no longer meaningful, not that the other byte is modified.** `LD T,(BC)` writes only `T`; `F` keeps its old value, but the combined value of `FT` is invalid.

#### How to read the Native Instruction Table

The table has three columns: **Reads**, **Writes**, and **Clobbers**.

- **Reads** uses the notation `r8/r8/... - r16/r16/...`: direct byte operands on the left of `-`, and their parent 16-bit pairs on the right.
- **Writes** lists only the 8-bit registers that the instruction actually modifies.
- **Clobbers** lists the 16-bit parent pairs of any written byte. The other byte of a clobbered pair is **not** changed — only the old 16-bit value is no longer valid.

Separating **Writes** (actual byte modifications) from **Clobbers** (parent-pair validity loss) makes it clear that writing `T` does not write `F` — a common misinterpretation when a single column mixes the two.

| Instruction | Reads | Writes | Clobbers |
|---|---|---|---|
| `ld t,(bc)` | `B/C - BC` | T | FT |

**Reading this row:**
- `Reads B/C - BC`: the instruction reads `B` and `C` (the pointer), and therefore the parent pair `BC`.
- `Writes T`: the instruction writes only `T`.
- `Clobbers FT`: because `T` is the low byte of `FT`, the 16-bit value of `FT` is no longer valid. The `F` register itself is **not** modified.

Another non-obvious example:

| Instruction | Reads | Writes | Clobbers |
|---|---|---|---|
| `cmp b,i8` | `B - BC` | F | FT |

**Reading this row:**
- `Reads B - BC`: the instruction reads `B` and its parent pair `BC`.
- `Writes F`: the instruction writes only `F`.
- `Clobbers FT`: because `F` is the high byte of `FT`, the 16-bit value of `FT` is no longer valid. The `T` register itself is **not** modified.

#### Non-obvious clobbers

| Instruction | Writes | Clobbers |
|---|---|---|
| `cmp t,r8` | `F` | `FT` |
| `cmp r8,i8` | `F` | `FT` |
| `tst r16` | `F` | `FT` |
| `ld t,(r16)` | `T` | `FT` |
| `ld r8,(ft)` | `r8` | parent pair of `r8` |
| `add r8,i8` | `r8` | parent pair of `r8` |
| `ld f,i8` | `F` | `FT` |

This is the most common source of bugs on the RC800 architecture. When mixing 8-bit and 16-bit operations, trace the containing pair of every 8-bit register to verify no live 16-bit value is silently clobbered.

**F as data is destroyed by `CMP` and `TST`.** Although F can hold user data between writes (see [F Register Tricks](#f-register-tricks)), any `CMP` or `TST` instruction overwrites F with comparison flags. Do not use F as a temporary hold across a comparison — save the value in H or L (callee-saved, already pushed/popped at function boundaries) instead:

```
        ld      t,(bc+)         ; T = byte to save
        ld      h,t             ; H = byte (safe across comparisons)
        ld      t,(de+)         ; T = another byte
        cmp     t,h             ; F = Flags[T - H] — H survives, F overwritten
```

#### Flags survival across T/F writes

Because the `Clobbers` column reports parent-pair validity loss — not byte modification (see the [Clobbers column note](#how-to-read-the-native-instruction-table) above) — the flag byte `F` is preserved by any instruction whose **Writes** column lists only `T`, and `T` is preserved by any instruction whose **Writes** column lists only `F`.

Practical consequence: after `cmp t,i8`, `cmp r8,i8`, or `tst r16` sets flags in `F`, a following instruction that writes only `T` — such as `add t,i8`, `sub t,i8`, `ld t,r8`, or `ld t,(r16)` — leaves those flags intact. An immediately subsequent conditional instruction (`INST/CC`, `J/CC`) therefore reads the original comparison flags, not a stale or destroyed copy.

```
        cmp     t,1             ; F <- Flags[T - 1]
        add     t,2             ; T += 2 ; writes T only, F preserved
        j/ne    @+2             ; skips the add iff original T == 1 (reads F)
        add     t,4             ; executes only on the EQ path
```

Without the `T`-write-only property of `add t,i8`, computing a value from `T` between a `cmp` and the conditional that consumes its flags would require a separate register to hold the comparison result. The [`nibble_to_hex`](examples/nibble_to_hex.md) example relies on this idiom to convert a 4-bit value to ASCII in a single pass.

**Caveat:** the inverse does not hold — an instruction that writes `F` (e.g., another `cmp`, `tst`, or `ld f,i8`) destroys the flags even if `T` survives. Re-derive flag liveness from the table whenever a new instruction is inserted between the `cmp` and the conditional consumer.

#### How to check synthesized instructions

The [Native Instruction Table](#native-instruction-table) only lists native instructions. For synthesized instructions, determine clobbers by expanding the instruction and checking each native step:

1. Find the expansion in the [Synthesized Instructions](#synthesized-instructions) section.
2. Look up each native instruction in the [Native Instruction Table](#native-instruction-table).
3. Apply the Clobber Rule: if any 8-bit register in a live pair is written, the whole pair is clobbered.

**Example:** `LD FT, (BC)` is synthesized as:

```
        ld      t,(bc)
        add     bc,1
        exg     f,t
        ld      t,(bc)
        add     bc,-1
        exg     f,t
```

During this sequence, `T`, `F`, `BC`, and `FT` are all modified at various points. The final result is in `FT`, but the intermediate steps are only safe if `BC` and `FT` are not needed simultaneously.

### 32-Bit Value Convention (`ft:ft'`)

The FT stack is the standard mechanism for multi-word values. A 32-bit value is stored as:
- **FT** (register) = low word
- **FT'** (top of FT stack, accessed via `SWAP FT`) = high word

Little-endian on the stack: the low word is always in the register, the high word is one `SWAP` away. This convention is used by the calling convention for multi-word parameters and return values.

### Register Stacks

Each 16-bit register pair has its own 256-word (16-bit word) on-chip stack. The register acts as a window into the top of its stack — it is **not** separate storage from the stack. Firmware convention defines `RC8_STACK_SIZE` as `256`.

- **PUSH r** — duplicates top value, decrements stack pointer
- **POP r** — increments stack pointer, loads new top into register
- **SWAP r** — exchanges register with next stack entry
- **PICK r** — replaces register with entry at index (index = current register value & 0xFF). Index 0 = the register itself (current top), index 1 = the last pushed value, index 2 = the one before that, and so on.

#### Stack Pointer Direction and Depth

The stack grows **downward**: SP starts at `$FF` (empty) and each `PUSH` decrements it toward `$00`. So `$FF` means 0 entries, `$FE` means 1 entry, and `$00` means 255 entries. The raw SP value is inverted relative to depth; number of entries = `$FF - SP`, computable in one instruction via `NOT T` after `LCR T,(C)`. The `RC8_SP_*` configuration registers (see [Configuration Registers](#configuration-registers)) expose these SP values for read and write via `LCR`.

#### Writing the SP Clobbers the Register

Because the register is a live view into `Stack[SP]`, writing the SP via `LCR (C),T` changes the register content to reflect `Stack[new_SP]` — the register does not retain its previous value. Treat any SP write as clobbering the register pair. Load a known value into the register after the SP write if its content is needed.

#### Interrupt Safety of SP Manipulation

Advancing the SP (discarding entries) is safe with interrupts enabled — an interrupt handler's balanced `PUSH`/`POP` temporarily uses the freed region but restores SP on return, leaving reachable entries below the new SP intact. **Rewinding** the SP (decreasing it to restore previously discarded entries) is **not** safe with interrupts enabled: an interrupt handler's `PUSH` overwrites the freed entries, and even a balanced `PUSH`/`POP` leaves the handler's value in memory at the freed address, so the rewind exposes garbage. Disable interrupts (`DI`/`EI`) around any discard-and-rewind window, or use a register stack that no interrupt handler touches.

See [`examples/lcr_stack_pointer.md`](examples/lcr_stack_pointer.md) for worked examples of LCR-based SP manipulation (discard, rewind, empty, depth query).

**Unique architecture feature:** Because each register pair has its own independent stack, there is no shared memory stack. This means you do not need to push and pop register pairs in a specific order — `push bc; push de; pop bc; pop de` is perfectly valid, unlike architectures with a single shared stack where LIFO order is mandatory.

#### Register-Stack Architecture

The fundamental model: **the register IS the stack top**. Each register (FT, BC, DE, HL) is a window into `Stack[SP]` — the register value and the top stack entry are the same physical storage. This differs from traditional architectures where the register and stack are separate entities.

**Traditional architecture model:**
```
AX register ──> $1234
Stack top   ──> $5678  (separate storage)

PUSH AX:  copies $1234 to stack; AX still holds $1234
POP AX:   loads stack top into AX; stack top unchanged
```

**RC800 register-stack model:**
```
FT register ──┐
              ├──> Stack[SP] = $1234  (same physical storage)
Stack[SP]  ───┘

Before PUSH FT:
  FT = $1234, Stack[SP] = $1234

After PUSH FT:
  FT = $1234, Stack[SP] = $1234, Stack[SP+1] = $1234
  (SP decremented; both register and stack top still hold $1234)

After POP FT:
  FT = Stack[SP+1], SP incremented
  (loads next stack entry into register)
```

**Key implications:**

- **Writing the register writes the stack top.** `LD FT, $5678` sets both FT and `Stack[SP]` to `$5678`. The register is not a separate copy.

- **PUSH duplicates the current value.** After `PUSH FT`, both `Stack[SP]` (the register) and `Stack[SP+1]` hold the same value. The register still holds the original value because it IS `Stack[SP]`.

- **POP loads the next entry.** `POP FT` increments SP and loads `Stack[SP]` into FT. The register now holds whatever was at the next stack position.

- **The register is not "above" the stack.** It is the top entry. There is no separate register value that gets pushed onto the stack; the register value and stack top are identical.

This model applies to all four register stacks (FT, BC, DE, HL). Each register is a window into its own stack's top entry.

**Common Mistakes:**

- **`push ft` followed by `swap ft` is a no-op.** After `PUSH FT`, both FT and `Stack[SP+1]` contain the same value. Swapping them exchanges identical values, leaving the stack unchanged. This pattern cannot be used to copy or duplicate elements.

- **`push ft` duplicates, it does not "push the register."** The instruction creates a copy of the current TOS on the stack below. It does not move FT to a new stack position while leaving the old position empty. After push, FT and `Stack[SP+1]` are identical.

#### Using the FT Stack for RPN Calculations

The RC800 has a limited register set (only BC, DE, HL as general-purpose pairs). When complex calculations require more temporary storage than registers provide, the FT stack serves as an unlimited data stack for RPN (Reverse Polish Notation) calculations. Using the register-stack model described above, FT serves as TOS (top-of-stack), with stack entries below at `Stack[SP+1]`, `Stack[SP+2]`, etc.

```
        FT register          = TOS      (index 0)
        Stack[SP + 1]        = 2nd      (index 1, last pushed value)
        Stack[SP + 2]        = 3rd      (index 2)
        Stack[SP + 3]        = 4th      (index 3)
        ...
```

This model maps naturally to RPN calculators and stack-based languages. Common stack operations:

| Operation | Stack effect | RC800 instruction | Meaning |
|---|---|---|---|
| dup | `( a -- a a )` | `push ft` | Duplicate TOS |
| drop | `( a -- )` | `pop ft` | Drop TOS, load next |
| swap | `( a b -- b a )` | `swap ft` | Exchange TOS with 2nd |
| over | `( a b -- a b a )` | `push ft` / `swap ft` | Copy 2nd to TOS |

**Pushing a value** follows a two-step pattern. The first value is loaded directly into FT (FT becomes TOS). Each subsequent value requires `push ft` first to save the current TOS, then `ld ft,value` to load the new TOS:

```
        ld      ft,3         ; FT=3, stack: [3]        (first value, just load)
        push    ft           ; save 3 below FT
        ld      ft,4         ; FT=4, stack: [3, 4]     (second value)
        push    ft           ; save 4 below FT
        ld      ft,5         ; FT=5, stack: [3, 4, 5]  (third value)
```

**Binary operations** `( a b -- result )` consume two stack entries and produce one. The stack depth decreases by 1. The pattern is: save TOS to a scratch register, pop to get the second operand, and operate. The result lands in FT which is already TOS — **no `push ft` needed**:

```
        ; Stack: [a, b]  (FT = b, Stack[SP+1] = a)
        ld      bc,ft        ; BC = b (save TOS)
        pop     ft           ; FT = a (2nd becomes TOS)
        add     ft,bc        ; FT = a + b (result is TOS)
        ; Stack: [a+b]      (depth decreased by 1)
```

**Unary operations** `( a -- result )` replace TOS in place. The stack depth is unchanged. Just modify FT directly:

```
        neg     ft           ; FT = -a (TOS replaced, depth unchanged)
```

**Stack depth changes** for common operations:

| Operation | Depth change | Example |
|---|---|---|
| Binary op (add, sub, mul, ...) | -1 | `( a b -- a+b )` |
| Unary op (neg, abs, invert) | 0 | `( a -- -a )` |
| `push ft` (dup) | +1 | `( a -- a a )` |
| `pop ft` (drop) | -1 | `( a -- )` |
| `swap ft` | 0 | `( a b -- b a )` |

This RPN approach is useful whenever register pressure is high — for example, in expression evaluators, interpreters, or complex mathematical calculations. The FT stack provides 256 entries of temporary storage without consuming any registers.

See [`rpn_stack_machine.md`](examples/rpn_stack_machine.md) for a complete RPN calculator implementation.


### Register Stack Patterns

**`SWAP FT` as 32-bit accessor**: `SWAP FT` exchanges FT with the top of the FT stack. This is the primary way to access the high word of a 32-bit value:

```
        ld ft,$1234        ; low word
        push ft
        ld ft,$5678        ; high word
        swap ft            ; ft = $5678 (high word), $1234 on stack
        ; ... work with high word ...
        swap ft            ; ft = $1234 (low word), $5678 on stack
```

**`PICK R16, imm` assembler shorthand**: The assembler accepts an immediate index: `PICK FT, 2` loads stack entry 2 directly into FT, without setting FT to the index value first. Index 0 = the register itself (current top), index 1 = the last pushed value, index 2 = the one before that.

**Byte-by-byte operations using `EXG F,T`**: When performing 16-bit operations byte-by-byte on FT, use `EXG F,T` to swap between bytes without needing to save/restore through other registers:

```
        ; Example: 16-bit AND with value in BC
        and     t,c          ; operate on low byte (T & C)
        exg     f,t          ; F = low result, T = old F (high byte)
        and     t,b          ; operate on high byte (T & B)
        exg     f,t          ; F = high result, T = low result
        ; FT now contains the 16-bit result
```

This pattern works because `EXG F,T` swaps the F and T registers, allowing you to operate on one byte, swap to expose the other byte in T, operate on it, then swap back to restore the first byte to T.

All registers can be pushed/popped simultaneously (`PUSHA`/`POPA`), enabling fast interrupt response.

### Configuration Registers

Accessed via `LCR` instruction using C register as index:

| Index | Constant | Content |
|-------|----------|---------|
| $00 | `RC8_SP_FT` | FT stack pointer |
| $01 | `RC8_SP_BC` | BC stack pointer |
| $02 | `RC8_SP_DE` | DE stack pointer |
| $03 | `RC8_SP_HL` | HL stack pointer |
| $04 | `RC8_SP_LOW` | Stack lower bound |
| $05 | `RC8_SP_HIGH` | Stack upper bound |

### Special Registers

| Name | Description |
|------|-------------|
| PC | 16-bit program counter, starts at $0000 on reset |
| IEF | Interrupt enable flag, controlled by `EI`/`DI` |

## Condition Codes

**Only `CMP` and `TST` set the F register.** Arithmetic, bitwise, load, store, and move instructions do not update flags; there is no implicit flag setting on this architecture. F is tested by `J/CC` and `DJ`. The F register is fully writable, so condition codes can also be set directly by writing to F:

| Bit | Value | Name |
|-----|-------|------|
| 0 | $01 | Carry (C) |
| 1 | $02 | Zero (Z) |
| 2 | $04 | Negative (N) |
| 3 | $08 | Overflow (V) |

### Flag Constants

Firmware convention defines these named constants (from `rc800.i`):

| Constant | Value | Meaning |
|----------|-------|---------|
| `FLAGS_CARRY` | $01 | Carry flag |
| `FLAGS_ZERO` | $02 | Zero flag |
| `FLAGS_NEGATIVE` | $04 | Negative flag |
| `FLAGS_OVERFLOW` | $08 | Overflow flag |
| `FLAGS_Z` | $02 | Alias for FLAGS_ZERO |
| `FLAGS_NZ` | $00 | Not zero (clears all flags) |
| `FLAGS_NE` | $00 | Alias for FLAGS_NZ (not equal) |
| `FLAGS_EQ` | $02 | Alias for FLAGS_Z (equal) |
| `FLAGS_LTU` | $01 | Alias for FLAGS_CARRY (unsigned less than) |

Common values: `LD F, 0` = EQ/Z, `LD F, $FF` = NE/NZ, `LD F, $04` = LT/N.

| Code | Meaning | Alt |
|------|---------|-----|
| LE | Signed less than or equal | Z or (N xor V) |
| GT | Signed greater than | not (Z or (N xor V)) |
| LT | Signed less than | N xor V |
| GE | Signed greater than or equal | not (N xor V) |
| LEU | Unsigned less than or equal | Z or C |
| GTU | Unsigned greater than | not (Z or C) |
| LTU | Unsigned less than | C (carry) |
| GEU | Unsigned greater than or equal | NC (not carry) |
| EQ | Equal | Z (zero) |
| NE | Not equal | NZ (not zero) |

## Interrupts

### Interrupt Modes (ascending priority)

1. **User** — normal execution
2. **System** — entered via `SYS` instruction
3. **Int** — external interrupt (maskable via `DI`/`EI`)
4. **NMI** — non-maskable interrupt

Higher modes can interrupt lower modes. A mode cannot interrupt itself. `RETI` returns to the previously active mode.

### Interrupt Handling

When an interrupt fires:
1. HL is pushed onto its stack
2. Return PC is stored in HL
3. PC is set to the entry address (code at this address may then jump to the handler)

### Entry Addresses

These are not jump vectors — the CPU sets PC directly to the address, and the code placed there decides what to do (typically jumping to the actual handler).

| Address | Constant | Purpose |
|---------|----------|---------|
| $0000 | `RC8_VECTOR_RESET` | Reset (power-up) |
| $0008 | `RC8_VECTOR_NMI` | Non-maskable external interrupt |
| $0010 | `RC8_VECTOR_ILLEGAL_IRQ` | Illegal interrupt (`SYS` used in SYS handler) |
| $0018 | `RC8_VECTOR_ILLEGAL_OP` | Illegal instruction |
| $0020 | `RC8_VECTOR_STACK_OVFL` | Stack overflow |
| $0028 | `RC8_VECTOR_EXT_INT` | External interrupt |
| $0038–$003F | | Reserved |
| $0040–$07FF | SYS entry points (vector byte × 8) |

## Calling Convention

### Parameter Passing

Parameters are assigned left to right. The first parameters use registers from the sequence **T, B, C, D, E** (8-bit) or **FT, BC, DE** (16-bit). When the register sequence is exhausted, remaining parameters spill onto the HL stack. `HL` is never used for register parameters because `jal` overwrites it with the return address.

- **8-bit param** — occupies one register from the sequence, or one 16-bit word on the HL stack (value in low byte, high byte sign- or zero-extended per declared type)
- **16-bit param** — occupies FT, BC, or DE in order, or one 16-bit word on the HL stack
- **Multi-word (≥32-bit)** — passed on a register stack; MSW in R, next word in R'

Mixed 8-bit and 16-bit parameters are allowed. An 8-bit parameter may not use a register that is part of an already-claimed 16-bit pair. Organize parameter order to avoid wasting registers or leaving "holes" in the assignment sequence.

**Examples:**

| Signature | Assignment |
|-----------|-----------|
| `f(u8 a, u8 b)` | T, B |
| `f(u16 a, u16 b)` | FT, BC |
| `f(u16 a, u8 b)` | FT, B |
| `f(u8 a, u16 b)` | T, BC |
| `f(u8 a, u16 b, u8 c)` | T, BC, D |
| `f(u8 a, u8 b, u16 c)` | T, B, —, DE (hole at C — avoid) |
| `f(u8 a, u8 b, u8 c, s8 d, u8 e)` | T, B, C, D, E |
| `f(u16 a, u16 b, u16 c)` | FT, BC, DE |

When the first parameter is 8-bit (occupying T), FT is unavailable because T is the low byte of FT, so the first 16-bit parameter falls back to BC. Put 16-bit parameters first in the signature to give them access to FT: prefer `f(u16, u8)` → FT, B over `f(u8, u16)` → T, BC.

### Stack Parameters

Parameters that do not fit in registers are passed on the HL stack. Each parameter occupies one 16-bit word regardless of size. For 8-bit parameters, the value is in the low byte; the high byte is sign-extended for signed types (`s8`) and zero-extended for unsigned types (`u8`).

**Caller responsibilities:**

1. Push stack parameters onto the HL stack in declaration order (first stack param pushed first).
2. Push a sacrificial slot after the last parameter — `jal` overwrites the HL stack top with the return address.
3. Load register parameters into their assigned registers.

```
        ; call f(u8 a, u16 b, u16 c, s8 d, u8 e)
        ; registers: T=a, BC=b, DE=c  |  stack: d(s8), e(u8)
        ld      t,arg_d
        ext                    ; sign-extend (s8)
        ld      hl,ft
        push    hl             ; d on HL stack
        ld      t,arg_e
        ld      f,0            ; zero-extend (u8)
        ld      hl,ft
        push    hl             ; e on HL stack
        push    hl             ; sacrificial slot for jal
        ld      t,arg_a
        ld      bc,arg_b
        ld      ft,arg_c
        ld      de,ft
        jal     f
```

**Stack layout after `jal`:**

```
  HL (index 0, SP)   = return address
  index 1             = e (last stack param pushed)
  index 2             = d (first stack param pushed)
```

The last stack parameter pushed is closest to the return address (index 1).

**Callee responsibilities:**

There are two strategies for accessing stack parameters. The callee should employ whichever fits best, or combine them.

**Strategy 1 — `PICK` (params stay on stack):**

`PUSH HL` duplicates the return address on the HL stack, shifting params to higher indices. `PICK HL,$nn` reads a parameter at a fixed index without disturbing the return address. Parameters remain on the stack and can be re-read at any time — useful when a param is accessed multiple times or conditionally.

```
        push    hl             ; dup ret; params shift to index 2+
        pick    hl,2           ; HL=arg1 (first stack param)
        ld      t,l            ; T=arg1 low byte
        pick    hl,3           ; HL=arg2
        ld      t,l
        ; ... params remain on stack for later access ...
```

**Strategy 2 — swap-pop (params consumed as accessed):**

No `PUSH HL` needed. `SWAP HL` exchanges HL with `Stack[SP+1]`, placing the parameter in HL and the return address into `Stack[SP+1]`. The parameter in HL can be used directly as an operand (`ADD T,L`, `ADD FT,HL`, `LD (BC),L`, etc.) — no register save needed when the value is consumed immediately. `POP HL` advances SP and reloads HL from `Stack[SP]` (which now holds the return address). Params are discarded as they are consumed — the stack is cleaned up incrementally.

```
        swap    hl             ; HL=arg1 (closest to ret)
        add     ft,hl          ; use HL directly as operand
        pop     hl             ; HL=ret restored, arg1 discarded

        swap    hl             ; HL=arg2
        add     t,l            ; use L directly
        pop     hl             ; HL=ret restored, arg2 discarded
        ; ... stack is clean, no further cleanup needed ...
```

When a parameter must be reused across multiple operations, save it to a callee-saved register byte between swap and pop:

```
        swap    hl             ; HL=arg
        ld      t,l            ; T=arg low byte
        ld      d,t            ; D=arg (saved for later use)
        pop     hl             ; HL=ret restored
```

**Combining strategies:** a function might `PICK` a frequently-accessed parameter (like a loop bound or pointer) and swap-pop the rest. The `PUSH HL` is only needed if `PICK` is used.

**Stack cleanup:**

After accessing parameters, any remaining entries on the HL stack must be discarded. If there are just a few entries, individual `POP HL` instructions are fine. For many entries, use `LCR` to advance the HL stack pointer in bulk:

```
        ; Discard N+1 entries (N stack params + 1 sacrificial slot)
        ld      c,RC8_SP_HL
        lcr     t,(c)          ; T = SP_HL
        add     t,N+1          ; advance past remaining entries
        lcr     (c),t          ; SP_HL = new value
        ; HL = Stack[new_SP] = return address (if aligned correctly)
```

**Epilogue — saving the return value:**

The callee computes its result in FT. The epilogue saves the result on the FT stack, puts the return address into FT for HL restoration, cleans up the HL stack, then recovers the result:

```
        ; FT = result, HL = ret
        push    ft             ; dup result on FT stack
        ld      ft,hl          ; FT=ret (overwrites dup; original result preserved below)
        pop     de/bc          ; restore callee-saved
        pop     hl             ; discard sacrificial slot
        ld      hl,ft          ; HL=ret
        pop     ft             ; FT=result (restored from FT stack)
        j       (hl)
```

`PUSH FT` duplicates the result. `LD FT,HL` overwrites the duplicate with the return address while the original result survives at `Stack[SP_FT+1]`. `POP FT` at the end restores it.

**Nested calls:**

When the callee makes a nested call, the inner callee's HL stack cleanup will discard outer stack entries. Save the return address in FT before the inner call:

```
        push    ft             ; save FT
        ld      ft,hl          ; FT=ret
        ; ... set up inner call, jal inner ...
        pop     hl             ; discard inner args + sacrificial
        ld      hl,ft          ; restore outer ret
        pop     ft             ; restore FT
```

**Prologue/epilogue for functions with stack parameters** (swap-pop variant):

```
f:
        push    bc/de          ; callee-saved (NOT hl)

        ; swap-pop each stack param
        swap    hl; ld t,l; pop hl
        ; ... body: compute result in FT ...

        push    ft             ; save result
        ld      ft,hl          ; FT=ret
        pop     de/bc          ; restore callee-saved
        pop     hl             ; discard sacrificial slot
        ld      hl,ft          ; HL=ret
        pop     ft             ; FT=result
        j       (hl)
```

**Prologue/epilogue for functions with stack parameters** (`PICK` variant):

```
f:
        push    hl             ; save ret on HL stack
        push    bc/de          ; callee-saved

        ; PICK each stack param at fixed index
        pick    hl,2           ; HL=arg1
        ; ... body: compute result in FT ...

        push    ft             ; save result
        ld      ft,hl          ; FT=ret
        pop     de/bc          ; restore callee-saved
        ; discard remaining HL stack entries (pops or lcr)
        pop     hl             ; discard ret dup
        pop     hl             ; discard argN
        pop     hl             ; discard ret (saved in FT)
        ld      hl,ft          ; HL=ret
        pop     ft             ; FT=result
        j       (hl)
```

**Prologue/epilogue for functions without stack parameters** (standard):

```
f:
        push    bc/de/hl
        ; ... body ...
        pop     hl/de/bc
        j       (hl)
```

### Return Values

- Returned in FT (or on FT stack for multi-word).
- **8-bit returns**: When returning an 8-bit value in T, there is no need to clear F. The caller reads only T for an 8-bit result, so the contents of F are irrelevant. Only clear F when the return value is genuinely 16-bit and the high byte must be zero.

### Preserved Registers

- **Callee-saves**: BC, DE (including stack values).
- **Caller-saves**: FT (always assumed destroyed by callee).
- **HL**: holds the return address set by `jal`. Management depends on the function type:
  - **No stack parameters**: standard `push bc/de/hl` / `pop hl/de/bc`
  - **Swap-pop variant**: prologue pushes only `bc/de` (not HL). Params are consumed via `swap hl` / `pop hl`, which naturally preserves HL=ret between each pair. Epilogue `pop hl` discards the sacrificial slot.
  - **PICK variant**: prologue `push hl` duplicates ret (shifting params to higher indices). Epilogue discards all remaining HL stack entries (ret dup, ret, and all params) via individual `pop hl` or `LCR` advance.

### Subroutines

- `JAL (r16)` copies the return address into HL (does NOT push).
- Non-leaf subroutines must save/restore HL themselves.
- Return via `J (HL)`.

### Prologue/Epilogue Optimization

Use register-list `push`/`pop` forms in function prologues and epilogues (`push bc/de/hl`, `pop hl/de/bc`, etc.). The assembler can choose compact synthesized sequences automatically.

- `pusha`/`popa` execute in the same time as a single-register `push`/`pop`.
- For procedures that do not return a value in FT, `pusha`/`popa` is usually the fastest full-save prologue/epilogue.
- Any save set can be expressed in at most two instructions: either individual pushes/pops (for 1-2 registers), or `pusha`/`popa` plus one compensating `pop`/`push` (for 3 registers).
- This is why register-list syntax is preferred: it lets the assembler pick the best encoding or expansion strategy.

**Functions with stack parameters** use a different prologue/epilogue. See [Stack Parameters](#stack-parameters) for details. The key difference: HL is not part of the standard `push bc/de/hl` save set — it is managed through the HL stack. The swap-pop variant never pushes HL; it preserves ret naturally between `swap hl` / `pop hl` pairs and discards the sacrificial slot with a final `pop hl` in the epilogue. The PICK variant pushes HL in the prologue to duplicate ret (shifting params to higher indices for `PICK`), then discards all remaining entries in the epilogue.

### Register Allocation and Optimization

Treat register allocation as a first-class part of routine design, not as a cleanup step. Keep long-lived values in registers that the operations using them do not clobber, and verify those read/write effects against the Native Instruction Table.

Prefer maintaining useful derived state, such as a live data pointer, when recomputing it would require repeated address arithmetic or register transfers. Use the independent register stacks as local storage when recursion or nested calls need values preserved across calls.

When optimizing a routine, first understand the allocation strategy and then re-derive it for the current inputs, outputs, and clobber constraints. Similar-looking code is not a substitute for independently checking register lifetimes, stack balance, and algorithmic correctness.

Because every data path routes through `T` or `FT`, the choice of algorithm and the assignment of registers are tightly coupled. A design that looks optimal on paper may become awkward when every move must pass through the hub. When a routine fights the register set, reconsider the representation: pointer-based subranges, live pointers, and stack-saved temporaries often eliminate the need for repeated register-to-register transfers.

## Instruction Set Reference

### Notation

- `R8` — 8-bit register (F, T, B, C, D, E, H, L)
- `R16` — 16-bit register pair (FT, BC, DE, HL)
- `i8` — 8-bit immediate value
- `s8` — signed 8-bit immediate (-128 to +127)
- `i16` — 16-bit immediate value
- `CC` — condition code
- `Code[addr]` — byte fetched from code memory at addr
- `Data[addr]` — byte fetched from/stored to data memory at addr
- `IO[addr]` — byte from/to I/O space at addr
- `CR[n]` — configuration register at index n
- `Flags[x - y]` — sets F to comparison flags of x minus y

---

### Arithmetic

#### ADD T, R8
Add 8-bit register to T.
```
T <- T + R8
```
Opcode: `01000rrr`

#### ADD FT, R16
Add 16-bit register to FT.
```
FT <- FT + R16
```
Opcode: `111101rr`

#### ADD R8, i8
Add immediate byte to 8-bit register.
```
PC <- PC + 1
R8 <- R8 + Code[PC]
```
Opcode: `10100rrr`

#### ADD R16, s8
Add signed immediate byte to 16-bit register.
```
PC <- PC + 1
R16 <- R16 + SignExtend(Code[PC])
```
Opcode: `101111rr`

#### ADD R16, i16
Add immediate 16-bit value to 16-bit register.
- **Synthesized** by assembler.

#### SUB T, R8
Subtract 8-bit register from T.
```
T <- T - R8
```
Opcode: `01010rrr` (r=1 invalid)

#### SUB FT, R16
Subtract 16-bit register from FT.
```
FT <- FT - R16
```
Opcode: `111100rr` (r=0 invalid)

**`SUB R16,R16` is only native for FT.** There is no `SUB BC,DE`, `SUB DE,BC`, etc. To subtract two non-FT 16-bit registers, route through FT: `LD FT,BC` / `SUB FT,DE` / `LD BC,FT`. Subtracting immediates from any 16-bit register (`SUB BC,s8`, `SUB DE,i16`) is synthesized via `ADD` with a negated immediate.

#### SUB R8, i8 / SUB R16, s8 / SUB R16, i16
Subtract immediate values.
- **Synthesized** by assembler.

#### NEG T
Negate T (two's complement).
```
T <- -T
```
Opcode: `01010001`

#### NEG FT
Negate FT (two's complement).
```
FT <- -FT
```
Opcode: `11110000`

#### EXT
Sign-extend T into F.
```
F <- if T[7] == 1 then $FF else $00
```
Opcode: `01001001`

---

### Bitwise

#### AND T, R8
```
T <- T & R8
```
Opcode: `01101rrr` (r=1 invalid)

#### AND T, i8
```
PC <- PC + 1
T <- T & Code[PC]
```
Opcode: `10110001`

#### OR T, R8
```
T <- T | R8
```
Opcode: `01100rrr` (r=1 invalid)

#### OR T, i8
```
PC <- PC + 1
T <- T | Code[PC]
```
Opcode: `10110000`

#### XOR T, R8
```
T <- T ^ R8
```
Opcode: `01110rrr` (r=1 invalid)

#### XOR T, i8
```
PC <- PC + 1
T <- T ^ Code[PC]
```
Opcode: `10110010`

#### NOT F
One's complement of F.
```
F <- ~F
```
Opcode: `00011000`

#### NOT T / NOT FT
One's complement.
- **Synthesized** by assembler.

#### LS FT, R8
Left-shift FT by amount in 8-bit register (masked to 4 bits).
```
FT <- FT << (R8 & 0xF)
```
Opcode: `11100rrr` (r=0,1 invalid)

#### LS FT, i8
Left-shift FT by immediate amount (masked to 4 bits).
```
PC <- PC + 1
FT <- FT << (Code[PC] & 0xF)
```
Opcode: `10111000`

#### RS FT, R8
Logical right-shift FT by amount in 8-bit register (masked to 4 bits).
```
FT <- FT >> (R8 & 0xF)
```
Opcode: `11101rrr` (r=0,1 invalid)

#### RS FT, i8
Logical right-shift FT by immediate amount (masked to 4 bits).
```
PC <- PC + 1
FT <- FT >> (Code[PC] & 0xF)
```
Opcode: `10111001`

#### RSA FT, R8
Arithmetic right-shift FT by amount in 8-bit register (masked to 4 bits). Sign bit is preserved.
```
FT <- FT >>> (R8 & 0xF)
```
Opcode: `11111rrr` (r=0,1 invalid)

#### RSA FT, i8
Arithmetic right-shift FT by immediate amount (masked to 4 bits).
```
PC <- PC + 1
FT <- FT >>> (Code[PC] & 0xF)
```
Opcode: `10111011`

---

### Comparison

#### CMP T, R8
Set F to flags of T minus the 8-bit register.
```
F <- Flags[T - R8]
```
Opcode: `01001rrr` (r=1 invalid)

#### CMP R8, i8
Set F to flags of the 8-bit register minus the immediate byte.
```
PC <- PC + 1
F <- Flags[R8 - Code[PC]]
```
Opcode: `10101rrr`

#### CMP FT, R16
Set F to flags of FT minus the 16-bit register.
```
F <- Flags[FT - R16]
```
Opcode: `110011rr` (r=0 invalid)

#### TST R16
Set F to flags of the 16-bit register minus zero.
```
F <- Flags[R16 - 0]
```
Opcode: `110101rr`

---

### Register Transfer

**CRITICAL CONSTRAINT: DO NOT ATTEMPT DIRECT REGISTER-TO-REGISTER TRANSFERS.**

Every `LD` instruction that moves data between registers **must** have `T` (8-bit) or `FT` (16-bit) as one of the two operands. There is no opcode for any other combination. Attempting `LD DE, BC`, `LD C, B`, `LD E, D`, etc. will produce an assembler error — these instructions do not exist.

**8-bit transfers: always route through `T`**

```
        ld t, b        ; T <- B
        ld c, t        ; C <- T   (effectively C = B)
```

Or use `EXG` for exchanges:

```
        exg t, b       ; swap T and B
```

**16-bit transfers: always route through `FT`**

```
        ld ft, bc      ; FT <- BC
        ld de, ft      ; DE <- FT   (effectively DE = BC)
```

**There is no exception to this rule.** If you need to copy a value between two registers, one of them must be `T` (8-bit) or `FT` (16-bit). Otherwise, insert an intermediate step through `T`/`FT`.

#### Thinking in T/FT

The syntax `LD T, R8` / `LD R8, T` may suggest symmetric register-to-register moves, but the mental model is better framed as **directional transfers through the accumulator**:

- `LD T, r8` = **"move to T"** (load T from a source register)
- `LD r8, T` = **"move from T"** (store T into a destination register)

This pattern is not limited to `LD` — it applies across the instruction set. Arithmetic instructions (`ADD`, `AND`, `OR`, `XOR`, `CMP`, etc.) always operate on `T` as the accumulator. When an instruction accepts two register operands, one is almost invariably `T` (8-bit) or `FT` (16-bit). Think of T/FT as the hub through which all register data flows, not as peers among equals.

#### LD R8, i8
Load immediate byte into 8-bit register.
```
PC <- PC + 1
R8 <- Code[PC]
```
Opcode: `10000rrr`

#### LD T, R8
Load T from 8-bit register.
```
T <- R8
```
Opcode: `01011rrr` (r=1 invalid)

#### LD R8, T
Load 8-bit register from T.
```
R8 <- T
```
Opcode: `01111rrr` (r=1 invalid)

#### LD FT, R16
Load FT from 16-bit register.
```
FT <- R16
```
Opcode: `110100rr` (r=0 invalid)

#### LD R16, FT
Load 16-bit register from FT.
```
R16 <- FT
```
Opcode: `110110rr` (r=0 invalid)

#### LD R16, i16
Load 16-bit immediate into register.
- **Synthesized** by assembler.

#### EXG T, R8
Exchange T with an 8-bit register.
```
Temp <- T
T <- R8
R8 <- Temp
```
Opcode: `00010rrr` (r=1 invalid)

#### EXG FT, R16
Exchange FT with a 16-bit register.
```
Temp <- FT
FT <- R16
R16 <- Temp
```
Opcode: `110010rr` (r=0 invalid)

---

### Memory Access

#### LD T, (R16)
Load T from data memory at address in R16.
```
T <- Data[R16]
```
Opcode: `000001rr`

#### LD (R16), T
Store T to data memory at address in R16.
```
Data[R16] <- T
```
Opcode: `000000rr` (r=0 invalid)

#### LD R8, (FT)
Load 8-bit register from data memory at address in FT.
```
R8 <- Data[FT]
```
Opcode: `00101rrr` (r=1 invalid)

#### LD (FT), R8
Store 8-bit register to data memory at address in FT.
```
Data[FT] <- R8
```
Opcode: `00100rrr` (r=0,1 invalid)

#### LIO T, (R16)
Load T from I/O space at address in R16. Any 16-bit register pair (FT, BC, DE, HL) is valid.
```
T <- IO[R16]
```
Opcode: `000011rr`

#### LIO (R16), T
Store T to I/O space at address in R16. Any 16-bit register pair (FT, BC, DE, HL) is valid.
```
IO[R16] <- T
```
Opcode: `000010rr`

#### LCO T, (R16)
Load T from code memory at address in R16. Asserts the CODE signal. Any 16-bit register pair (FT, BC, DE, HL) is valid.
```
T <- Code[R16]
```
Opcode: `000001rr`

#### LCR T, (C)
Load T from configuration register indexed by C.
```
T <- CR[C]
```
Opcode: `00001011`

#### LCR (C), T
Store T to configuration register indexed by C.
```
CR[C] <- T
```
Opcode: `00001010`

---

### Stack Operations

#### PUSH R16
Duplicate register onto its stack (decrements the pointer; the register remains as stack top).
```
Push R16
```
Opcode: `110000rr`

#### POP R16
Restore register from its stack (increments the pointer; the register becomes the new stack top).
```
Pop R16
```
Opcode: `110001rr`

#### PUSHA
Push all 16-bit registers onto their stacks simultaneously.
```
Push FT, BC, DE, HL
```
Opcode: `11111000`

#### POPA
Pop all 16-bit registers from their stacks simultaneously.
```
Pop FT, BC, DE, HL
```
Opcode: `11111001`

**Stack width:** Each register stack is a 16-bit word stack. `PUSH R16` stores the full 16-bit value of the register, and `POP R16` restores a full 16-bit value. The two 8-bit halves are saved and restored together as a single word.

```
        ; FT = $ABCD
        push    ft              ; stack[FT] <- $ABCD
        ; F and T still hold $AB and $CD
        pop     ft              ; FT <- stack[FT]
        ; F and T are restored together as $ABCD
```

**PUSHA/POPA efficiency:** `PUSHA` and `POPA` are not sequences of four individual pushes/pops. They are single instructions that push or pop all four 16-bit registers (FT, BC, DE, HL) simultaneously and take roughly the same time as a single `PUSH`/`POP`.

#### SWAP R16
Exchange register (stack top) with next stack entry.
```
Swap R16 with Stack[r][1]
```
Opcode: `110111rr`

#### SWAPA
Swap all registers with their next stack entries.
```
Swap FT, BC, DE, HL with their respective next stack entries
```
Opcode: `11001100`

#### PICK R16
Replace register (stack top) with stack entry at index (index = register value & 0xFF). Index 0 = the register itself (current top), index 1 = the last pushed value, index 2 = the entry before that, and so on deeper into the stack. After PICK, the register holds the picked value and thus becomes the new stack top.
```
R16 <- Stack[r][R16 & 0xFF]
```
Opcode: `000111rr`

---

### Program Flow

#### NOP
No operation.
```
PC <- PC + 1
```
Opcode: `00000000`

#### J s8
Unconditional relative jump.
```
PC <- PC + 1
PC <- PC + SignExtend(Code[PC])
```
Opcode: `10011010`

#### J (R16)
Indirect jump through 16-bit register.
```
PC <- R16
```
Opcode: `001111rr`

#### JAL (R16)
Jump and link — stores return address in HL.
```
HL <- PC + 1
PC <- R16
```
Opcode: `001110rr`

#### JAL i16
Jump to absolute address, store return in HL.
- **Synthesized** by assembler: `LD HL, i16; JAL (HL)`

#### J/CC s8
Conditional relative jump based on F register flags. This is a **native** instruction (not synthesized) — it is the fundamental branching primitive. Other conditional constructs (e.g., `INST/CC`) are synthesized by expanding to `J/!CC` to skip the target instruction.
```
PC <- PC + 1
if (F satisfies CC) then
    PC <- PC + SignExtend(Code[PC])
else
    PC <- PC + 1
```
Opcode: `1001CCrr` (CC < 10)

For the meanings of the condition codes (`LE`, `GT`, `LT`, `GE`, `LEU`, `GTU`, `LTU`, `GEU`, `EQ`/`Z`, `NE`/`NZ`), see the [Condition Codes](#condition-codes) section.

#### DJ R8, s8
Decrement register, jump if non-zero. **Important:** DJ decrements *first*, then tests. Load the exact iteration count into the register before the loop — do not pre-decrement.
```
PC <- PC + 1
R8 <- R8 - 1
if (R8 != 0) then
    PC <- PC + SignExtend(Code[PC])
else
    PC <- PC + 1
```
Opcode: `10001rrr`

Although commonly used for counted loops, `DJ` is also useful for early-exit checks. Load a countdown value, then use `DJ` to decrement it and branch only when the result is non-zero; execution falls through when the value reaches zero. This can avoid a separate decrement and zero comparison when a function needs to return early based on a bounded count.

**Nested DJ for counts > 256**: Since DJ operates on an 8-bit register, counts above 255 require two nested DJ loops. The convention is to pre-load a 16-bit value into DE, then loop on E (inner) and D (outer):

```
        ; count = n (16-bit)
        sub de,1
        add d,1
        add e,1
.loop   ; ... body ...
        dj e,.loop
        dj d,.loop
```
The `SUB DE,1; ADD D,1; ADD E,1` idiom converts count N into the pair `(N-1)/256 + 1, (N-1)%256 + 1` so both loops execute the correct number of iterations.

#### SYS i8
Software interrupt; PC is set to (immediate × 8).
```
SYSF <- 1
Push (PC + 2) to HL stack
PC <- Code[PC + 1] * 8
```
Opcode: `10011011`

---

### System Control

#### EI
Enable external interrupts.
```
IEF <- 1
```
Opcode: `00011010`

#### DI
Disable external interrupts.
```
IEF <- 0
```
Opcode: `00011011`

#### RETI
Return from interrupt.
```
PC <- HL
Pop HL stack
Resume previous interrupt mode
```
Opcode: `01011001`

#### RETI/CC
Conditional return from interrupt.
- **Synthesized**: `J/!CC,@+2` / `RETI`

---

## Native Instruction Table

Complete table of all native (non-synthesized) instruction permutations.

### Register Mapping Note

For the 16-bit/8-bit register mapping and clobber-checking rules, see [Register Aliasing](#register-aliasing-1) in the Register File section.

### Instructions

**Reads notation**: `r8/r8/... - r16/r16/...`.

- Left side of `-`: 8-bit registers used directly by the operation, plus child bytes of any explicit 16-bit operands.
- Right side of `-`: 16-bit registers used directly by the operation, plus parent pairs of any explicit 8-bit operands.
- **Writes** lists only the 8-bit registers that the instruction actually modifies.
- **Clobbers** lists the 16-bit parent pairs of any written byte; the other byte of a clobbered pair is not changed and remains valid as an independent 8-bit value (e.g., `F` survives instructions that write only `T`, so flags set by `cmp` persist across a following `add t,i8`). See [Flags survival across T/F writes](#flags-survival-across-tf-writes) for the idiom.
- Non-register side effects (stack state, `PC`, `IEF`) are appended after `;` in the Clobbers column.

| Instruction | Reads | Writes | Clobbers |
|---|---|---|---|
| `add t,f` | F/T - FT | T | FT |
| `add t,t` | T - FT | T | FT |
| `add t,b` | T/B - FT/BC | T | FT |
| `add t,c` | T/C - FT/BC | T | FT |
| `add t,d` | T/D - FT/DE | T | FT |
| `add t,e` | T/E - FT/DE | T | FT |
| `add t,h` | T/H - FT/HL | T | FT |
| `add t,l` | T/L - FT/HL | T | FT |
| `add ft,ft` | F/T - FT | F/T | FT |
| `add ft,bc` | F/T/B/C - FT/BC | F/T | FT |
| `add ft,de` | F/T/D/E - FT/DE | F/T | FT |
| `add ft,hl` | F/T/H/L - FT/HL | F/T | FT |
| `add f,i8` | F - FT | F | FT |
| `add t,i8` | T - FT | T | FT |
| `add b,i8` | B - BC | B | BC |
| `add c,i8` | C - BC | C | BC |
| `add d,i8` | D - DE | D | DE |
| `add e,i8` | E - DE | E | DE |
| `add h,i8` | H - HL | H | HL |
| `add l,i8` | L - HL | L | HL |
| `add ft,s8` | F/T - FT | F/T | FT |
| `add bc,s8` | B/C - BC | B/C | BC |
| `add de,s8` | D/E - DE | D/E | DE |
| `add hl,s8` | H/L - HL | H/L | HL |
| `sub t,f` | F/T - FT | T | FT |
| `sub t,b` | T/B - FT/BC | T | FT |
| `sub t,c` | T/C - FT/BC | T | FT |
| `sub t,d` | T/D - FT/DE | T | FT |
| `sub t,e` | T/E - FT/DE | T | FT |
| `sub t,h` | T/H - FT/HL | T | FT |
| `sub t,l` | T/L - FT/HL | T | FT |
| `sub ft,bc` | F/T/B/C - FT/BC | F/T | FT |
| `sub ft,de` | F/T/D/E - FT/DE | F/T | FT |
| `sub ft,hl` | F/T/H/L - FT/HL | F/T | FT |
| `neg t` | T - FT | T | FT |
| `neg ft` | F/T - FT | F/T | FT |
| `ext` | T - FT | F | FT |
| `and t,f` | F/T - FT | T | FT |
| `and t,b` | T/B - FT/BC | T | FT |
| `and t,c` | T/C - FT/BC | T | FT |
| `and t,d` | T/D - FT/DE | T | FT |
| `and t,e` | T/E - FT/DE | T | FT |
| `and t,h` | T/H - FT/HL | T | FT |
| `and t,l` | T/L - FT/HL | T | FT |
| `and t,i8` | T - FT | T | FT |
| `or t,f` | F/T - FT | T | FT |
| `or t,b` | T/B - FT/BC | T | FT |
| `or t,c` | T/C - FT/BC | T | FT |
| `or t,d` | T/D - FT/DE | T | FT |
| `or t,e` | T/E - FT/DE | T | FT |
| `or t,h` | T/H - FT/HL | T | FT |
| `or t,l` | T/L - FT/HL | T | FT |
| `or t,i8` | T - FT | T | FT |
| `xor t,f` | F/T - FT | T | FT |
| `xor t,b` | T/B - FT/BC | T | FT |
| `xor t,c` | T/C - FT/BC | T | FT |
| `xor t,d` | T/D - FT/DE | T | FT |
| `xor t,e` | T/E - FT/DE | T | FT |
| `xor t,h` | T/H - FT/HL | T | FT |
| `xor t,l` | T/L - FT/HL | T | FT |
| `xor t,i8` | T - FT | T | FT |
| `not f` | F - FT | F | FT |
| `ls ft,b` | F/T/B - FT/BC | F/T | FT |
| `ls ft,c` | F/T/C - FT/BC | F/T | FT |
| `ls ft,d` | F/T/D - FT/DE | F/T | FT |
| `ls ft,e` | F/T/E - FT/DE | F/T | FT |
| `ls ft,h` | F/T/H - FT/HL | F/T | FT |
| `ls ft,l` | F/T/L - FT/HL | F/T | FT |
| `ls ft,i8` | F/T - FT | F/T | FT |
| `rs ft,b` | F/T/B - FT/BC | F/T | FT |
| `rs ft,c` | F/T/C - FT/BC | F/T | FT |
| `rs ft,d` | F/T/D - FT/DE | F/T | FT |
| `rs ft,e` | F/T/E - FT/DE | F/T | FT |
| `rs ft,h` | F/T/H - FT/HL | F/T | FT |
| `rs ft,l` | F/T/L - FT/HL | F/T | FT |
| `rs ft,i8` | F/T - FT | F/T | FT |
| `rsa ft,b` | F/T/B - FT/BC | F/T | FT |
| `rsa ft,c` | F/T/C - FT/BC | F/T | FT |
| `rsa ft,d` | F/T/D - FT/DE | F/T | FT |
| `rsa ft,e` | F/T/E - FT/DE | F/T | FT |
| `rsa ft,h` | F/T/H - FT/HL | F/T | FT |
| `rsa ft,l` | F/T/L - FT/HL | F/T | FT |
| `rsa ft,i8` | F/T - FT | F/T | FT |
| `cmp t,f` | F/T - FT | F | FT |
| `cmp t,b` | T/B - FT/BC | F | FT |
| `cmp t,c` | T/C - FT/BC | F | FT |
| `cmp t,d` | T/D - FT/DE | F | FT |
| `cmp t,e` | T/E - FT/DE | F | FT |
| `cmp t,h` | T/H - FT/HL | F | FT |
| `cmp t,l` | T/L - FT/HL | F | FT |
| `cmp f,i8` | F - FT | F | FT |
| `cmp t,i8` | T - FT | F | FT |
| `cmp b,i8` | B - BC | F | FT |
| `cmp c,i8` | C - BC | F | FT |
| `cmp d,i8` | D - DE | F | FT |
| `cmp e,i8` | E - DE | F | FT |
| `cmp h,i8` | H - HL | F | FT |
| `cmp l,i8` | L - HL | F | FT |
| `cmp ft,bc` | F/T/B/C - FT/BC | F | FT |
| `cmp ft,de` | F/T/D/E - FT/DE | F | FT |
| `cmp ft,hl` | F/T/H/L - FT/HL | F | FT |
| `tst ft` | F/T - FT | F | FT |
| `tst bc` | B/C - BC | F | FT |
| `tst de` | D/E - DE | F | FT |
| `tst hl` | H/L - HL | F | FT |
| `ld f,i8` | — - — | F | FT |
| `ld t,i8` | — - — | T | FT |
| `ld b,i8` | — - — | B | BC |
| `ld c,i8` | — - — | C | BC |
| `ld d,i8` | — - — | D | DE |
| `ld e,i8` | — - — | E | DE |
| `ld h,i8` | — - — | H | HL |
| `ld l,i8` | — - — | L | HL |
| `ld t,f` | F - FT | T | FT |
| `ld t,b` | B - BC | T | FT |
| `ld t,c` | C - BC | T | FT |
| `ld t,d` | D - DE | T | FT |
| `ld t,e` | E - DE | T | FT |
| `ld t,h` | H - HL | T | FT |
| `ld t,l` | L - HL | T | FT |
| `ld f,t` | T - FT | F | FT |
| `ld b,t` | T - FT | B | BC |
| `ld c,t` | T - FT | C | BC |
| `ld d,t` | T - FT | D | DE |
| `ld e,t` | T - FT | E | DE |
| `ld h,t` | T - FT | H | HL |
| `ld l,t` | T - FT | L | HL |
| `ld ft,bc` | B/C - BC | F/T | FT |
| `ld ft,de` | D/E - DE | F/T | FT |
| `ld ft,hl` | H/L - HL | F/T | FT |
| `ld bc,ft` | F/T - FT | B/C | BC |
| `ld de,ft` | F/T - FT | D/E | DE |
| `ld hl,ft` | F/T - FT | H/L | HL |
| `exg t,f` | F/T - FT | F/T | FT |
| `exg t,b` | T/B - FT/BC | T/B | FT/BC |
| `exg t,c` | T/C - FT/BC | T/C | FT/BC |
| `exg t,d` | T/D - FT/DE | T/D | FT/DE |
| `exg t,e` | T/E - FT/DE | T/E | FT/DE |
| `exg t,h` | T/H - FT/HL | T/H | FT/HL |
| `exg t,l` | T/L - FT/HL | T/L | FT/HL |
| `exg ft,bc` | F/T/B/C - FT/BC | F/T/B/C | FT/BC |
| `exg ft,de` | F/T/D/E - FT/DE | F/T/D/E | FT/DE |
| `exg ft,hl` | F/T/H/L - FT/HL | F/T/H/L | FT/HL |
| `ld t,(ft)` | F/T - FT | T | FT |
| `ld t,(bc)` | B/C - BC | T | FT |
| `ld t,(de)` | D/E - DE | T | FT |
| `ld t,(hl)` | H/L - HL | T | FT |
| `ld (ft),t` | F/T - FT | — | — |
| `ld (bc),t` | T/B/C - FT/BC | — | — |
| `ld (de),t` | T/D/E - FT/DE | — | — |
| `ld (hl),t` | T/H/L - FT/HL | — | — |
| `ld f,(ft)` | F/T - FT | F | FT |
| `ld b,(ft)` | F/T - FT | B | BC |
| `ld c,(ft)` | F/T - FT | C | BC |
| `ld d,(ft)` | F/T - FT | D | DE |
| `ld e,(ft)` | F/T - FT | E | DE |
| `ld h,(ft)` | F/T - FT | H | HL |
| `ld l,(ft)` | F/T - FT | L | HL |
| `ld (ft),b` | F/T/B - FT/BC | — | — |
| `ld (ft),c` | F/T/C - FT/BC | — | — |
| `ld (ft),d` | F/T/D - FT/DE | — | — |
| `ld (ft),e` | F/T/E - FT/DE | — | — |
| `ld (ft),h` | F/T/H - FT/HL | — | — |
| `ld (ft),l` | F/T/L - FT/HL | — | — |
| `lio t,(ft)` | F/T - FT | T | FT |
| `lio t,(bc)` | B/C - BC | T | FT |
| `lio t,(de)` | D/E - DE | T | FT |
| `lio t,(hl)` | H/L - HL | T | FT |
| `lio (ft),t` | F/T - FT | — | — |
| `lio (bc),t` | T/B/C - FT/BC | — | — |
| `lio (de),t` | T/D/E - FT/DE | — | — |
| `lio (hl),t` | T/H/L - FT/HL | — | — |
| `lco t,(ft)` | F/T - FT | T | FT |
| `lco t,(bc)` | B/C - BC | T | FT |
| `lco t,(de)` | D/E - DE | T | FT |
| `lco t,(hl)` | H/L - HL | T | FT |
| `lcr t,(c)` | C - BC | T | FT |
| `lcr (c),t` | T/C - FT/BC | — | — |
| `push ft` | F/T - FT | — | — ; FT (stack) |
| `push bc` | B/C - BC | — | — ; BC (stack) |
| `push de` | D/E - DE | — | — ; DE (stack) |
| `push hl` | H/L - HL | — | — ; HL (stack) |
| `pop ft` | — - — | F/T | FT ; FT (stack) |
| `pop bc` | — - — | B/C | BC ; BC (stack) |
| `pop de` | — - — | D/E | DE ; DE (stack) |
| `pop hl` | — - — | H/L | HL ; HL (stack) |
| `pusha` | F/T/B/C/D/E/H/L - FT/BC/DE/HL | — | — ; FT (stack)/BC (stack)/DE (stack)/HL (stack) |
| `popa` | — - — | F/T/B/C/D/E/H/L | FT/BC/DE/HL ; FT (stack)/BC (stack)/DE (stack)/HL (stack) |
| `swap ft` | F/T - FT | F/T | FT ; FT (stack) |
| `swap bc` | B/C - BC | B/C | BC ; BC (stack) |
| `swap de` | D/E - DE | D/E | DE ; DE (stack) |
| `swap hl` | H/L - HL | H/L | HL ; HL (stack) |
| `swapa` | F/T/B/C/D/E/H/L - FT/BC/DE/HL | F/T/B/C/D/E/H/L | FT/BC/DE/HL ; FT (stack)/BC (stack)/DE (stack)/HL (stack) |
| `pick ft` | F/T - FT | F/T | FT ; FT (stack) |
| `pick bc` | B/C - BC | B/C | BC ; BC (stack) |
| `pick de` | D/E - DE | D/E | DE ; DE (stack) |
| `pick hl` | H/L - HL | H/L | HL ; HL (stack) |
| `nop` | — - — | — | — |
| `j s8` | — - — | — | — ; PC |
| `j (ft)` | F/T - FT | — | — ; PC |
| `j (bc)` | B/C - BC | — | — ; PC |
| `j (de)` | D/E - DE | — | — ; PC |
| `j (hl)` | H/L - HL | — | — ; PC |
| `jal (ft)` | F/T - FT | H/L | HL ; PC |
| `jal (bc)` | B/C - BC | H/L | HL ; PC |
| `jal (de)` | D/E - DE | H/L | HL ; PC |
| `jal (hl)` | H/L - HL | H/L | HL ; PC |
| `j/le s8` | F - FT | — | — ; PC |
| `j/gt s8` | F - FT | — | — ; PC |
| `j/lt s8` | F - FT | — | — ; PC |
| `j/ge s8` | F - FT | — | — ; PC |
| `j/leu s8` | F - FT | — | — ; PC |
| `j/gtu s8` | F - FT | — | — ; PC |
| `j/ltu s8` | F - FT | — | — ; PC |
| `j/geu s8` | F - FT | — | — ; PC |
| `j/eq s8` | F - FT | — | — ; PC |
| `j/ne s8` | F - FT | — | — ; PC |
| `dj f,s8` | F - FT | F | FT ; PC |
| `dj t,s8` | T - FT | T | FT ; PC |
| `dj b,s8` | B - BC | B | BC ; PC |
| `dj c,s8` | C - BC | C | BC ; PC |
| `dj d,s8` | D - DE | D | DE ; PC |
| `dj e,s8` | E - DE | E | DE ; PC |
| `dj h,s8` | H - HL | H | HL ; PC |
| `dj l,s8` | L - HL | L | HL ; PC |
| `sys i8` | — - — | — | — ; HL (stack)/PC |
| `ei` | — - — | — | — ; IEF |
| `di` | — - — | — | — ; IEF |
| `reti` | H/L - HL | H/L | HL ; HL (stack)/PC |

### Native Instruction Notes

- **Reads/Writes/Clobbers notation**: see [How to read the Native Instruction Table](#how-to-read-the-native-instruction-table) for column definitions and a worked `cmp b,i8` example.
- **Stack operations in the table:** For `PUSH R16`, the **Reads** column lists the two bytes of the 16-bit register that is pushed as a single word. The `; R16 (stack)` side effect in the **Clobbers** column means the value is duplicated onto that register's independent 16-bit stack. For `POP R16`, the **Writes** column lists the two bytes that are updated when the popped 16-bit word is loaded back into the register, and the **Clobbers** column lists the affected parent pair. `PUSHA`/`POPA` do the same for all four pairs at once.
- **Stack operation semantics** (`push`/`pop`/`swap`/`pick`): see [Register Stacks](#register-stacks) and the [PUSH](#push-r16)/[POP](#pop-r16)/[SWAP](#swap-r16)/[PICK](#pick-r16) instruction subsections.
- **J/CC condition codes**: `le`, `gt`, `lt`, `ge` (signed), `leu`, `gtu`, `ltu`, `geu` (unsigned), `eq`/`z` (equal), `ne`/`nz` (not equal). All read F to evaluate the condition.
- **LIO/LCO**: Assert IO or CODE signal respectively during the memory access.
- **LCO** has no store form (only load).
- **DJ** decrements the register first, then tests for zero. The register is listed in the **Writes** column because it is always modified; `PC` is listed after `;` in the **Clobbers** column because it may be modified.
- **reti** pops the HL stack and restores PC from the popped value, then resumes the previous interrupt mode.

---

## Synthesized Instructions

The assembler generates these convenience forms as safe sequences of native instructions with no unintended side effects. Use of synthesized instructions is encouraged — they produce the same machine code as the expanded form but yield shorter, more readable source.

**Cycle counting**: each synthesized instruction expands to the number of native instructions listed in its "Expands To" column. Every native instruction costs 4 cycles. When counting cycles or instructions per loop iteration, count the *expanded* form, not the source form — for example, `LD T,(BC+)` is one source instruction but two native instructions (`LD T,(BC)` + `ADD BC,1` = 8 cycles).

### Synthesized Instruction Table

Instructions synthesized by the assembler into sequences of native instructions. The **Reads** column uses the same `r8/r8/... - r16/r16/...` notation as the native table. The **Writes** column lists only the 8-bit registers actually modified, and the **Clobbers** column lists the 16-bit parent pairs affected. Non-register side effects (stack state, `PC`) are appended after `;` in the Clobbers column.

#### Immediate Loading

| Instruction | Reads | Writes | Clobbers | Expands To |
|---|---|---|---|---|
| `ld ft,i16` | — - — | F/T | FT | Two `LD` instructions; optimized for FT (see below) |
| `ld bc,i16` | — - — | B/C | BC | `LD B, high` / `LD C, low` |
| `ld de,i16` | — - — | D/E | DE | `LD D, high` / `LD E, low` |
| `ld hl,i16` | — - — | H/L | HL | `LD H, high` / `LD L, low` |

For `LD FT, i16`, the assembler optimizes based on the immediate value:
- `LD FT, $00xx` → `LD T, xx` / `EXT` (3 bytes)
- `LD FT, $xxxx` where both bytes equal → `LD T, xx` / `LD F, T` (3 bytes)
- All other values → `LD F, high` / `LD T, low` (4 bytes)

#### Arithmetic with Immediates (SUB)

Native `SUB` supports only register-to-register. These forms are synthesized by expanding to `ADD` with a negated immediate. They do **not** set F flags — only `CMP` and `TST` set flags.

| Instruction | Reads | Writes | Clobbers | Expands To |
|---|---|---|---|---|
| `sub f,i8` | F - FT | F | FT | `ADD F, -i8` |
| `sub t,i8` | T - FT | T | FT | `ADD T, -i8` |
| `sub b,i8` | B - BC | B | BC | `ADD B, -i8` |
| `sub c,i8` | C - BC | C | BC | `ADD C, -i8` |
| `sub d,i8` | D - DE | D | DE | `ADD D, -i8` |
| `sub e,i8` | E - DE | E | DE | `ADD E, -i8` |
| `sub h,i8` | H - HL | H | HL | `ADD H, -i8` |
| `sub l,i8` | L - HL | L | HL | `ADD L, -i8` |
| `sub ft,s8` | F/T - FT | F/T | FT | `ADD FT, -s8` |
| `sub bc,s8` | B/C - BC | B/C | BC | `ADD BC, -s8` |
| `sub de,s8` | D/E - DE | D/E | DE | `ADD DE, -s8` |
| `sub hl,s8` | H/L - HL | H/L | HL | `ADD HL, -s8` |
| `sub ft,i16` | F/T - FT | F/T | FT | `ADD FT, -i16` (further expanded) |
| `sub bc,i16` | B/C - BC | B/C | BC | `ADD BC, -i16` (further expanded) |
| `sub de,i16` | D/E - DE | D/E | DE | `ADD DE, -i16` (further expanded) |
| `sub hl,i16` | H/L - HL | H/L | HL | `ADD HL, -i16` (further expanded) |

#### 16-bit ADD with Immediate

| Instruction | Reads | Writes | Clobbers | Expands To |
|---|---|---|---|---|
| `add ft,i16` | F/T - FT | F/T | FT | `ADD FT, s8` if it fits; otherwise split into low-byte and high-byte adjustments |
| `add bc,i16` | B/C - BC | B/C | BC | `ADD BC, s8` if it fits; otherwise split into low-byte and high-byte adjustments |
| `add de,i16` | D/E - DE | D/E | DE | `ADD DE, s8` if it fits; otherwise split into low-byte and high-byte adjustments |
| `add hl,i16` | H/L - HL | H/L | HL | `ADD HL, s8` if it fits; otherwise split into low-byte and high-byte adjustments |

#### NOT

| Instruction | Reads | Writes | Clobbers | Expands To |
|---|---|---|---|---|
| `not t` | T - FT | T | FT | `XOR T, $FF` |
| `not ft` | F/T - FT | F/T | FT | `XOR T, $FF` / `NOT F` |

#### Jump and Link with Immediate

| Instruction | Reads | Writes | Clobbers | Expands To |
|---|---|---|---|---|
| `jal i16` | — - — | H/L | HL ; PC | `LD HL, i16` / `JAL (HL)` |

#### Post-increment and Pre-decrement Memory Access

The native instruction set does not include auto-increment/decrement addressing. These forms are synthesized for any 16-bit register pair (`FT`, `BC`, `DE`, `HL`).

**Target constraint**: the post-increment and pre-decrement load forms listed below target `T` only. There is no `LD C,(DE+)`, `LD B,(BC+)`, etc. — to load an 8-bit register other than `T` through a post-incrementing non-FT pointer, either load into `T` and transfer via `LD R8,T`, or use `FT` as the pointer (the `LD R8,(FT+)` / `LD R8,(-FT)` forms are synthesized separately; see [8-bit Memory Access through FT with Post-increment/Pre-decrement](#8-bit-memory-access-through-ft-with-post-incrementpre-decrement)).

| Instruction | Reads | Writes | Clobbers |
|---|---|---|---|
| `ld t,(ft+)` | F/T - FT | F/T | FT |
| `ld t,(bc+)` | B/C - BC | T/B/C | FT/BC |
| `ld t,(de+)` | D/E - DE | T/D/E | FT/DE |
| `ld t,(hl+)` | H/L - HL | T/H/L | FT/HL |
| `ld (ft+),t` | F/T - FT | F/T | FT |
| `ld (bc+),t` | T/B/C - FT/BC | B/C | BC |
| `ld (de+),t` | T/D/E - FT/DE | D/E | DE |
| `ld (hl+),t` | T/H/L - FT/HL | H/L | HL |
| `ld t,(-ft)` | F/T - FT | F/T | FT |
| `ld t,(-bc)` | B/C - BC | T/B/C | FT/BC |
| `ld t,(-de)` | D/E - DE | T/D/E | FT/DE |
| `ld t,(-hl)` | H/L - HL | T/H/L | FT/HL |
| `ld (-ft),t` | F/T - FT | F/T | FT |
| `ld (-bc),t` | T/B/C - FT/BC | B/C | BC |
| `ld (-de),t` | T/D/E - FT/DE | D/E | DE |
| `ld (-hl),t` | T/H/L - FT/HL | H/L | HL |

**Expansions:**
- `LD T, (R16+)` → `LD T, (R16)` / `ADD R16, 1`
- `LD (R16+), T` → `LD (R16), T` / `ADD R16, 1`
- `LD T, (-R16)` → `ADD R16, -1` / `LD T, (R16)`
- `LD (-R16), T` → `ADD R16, -1` / `LD (R16), T`

#### 16-bit Memory Loads/Stores through R16

Loading or storing a full 16-bit register pair through a non-FT pointer requires synthesized sequences.

| Instruction | Reads | Writes | Clobbers |
|---|---|---|---|
| `ld ft,(bc)` | B/C - BC | F/T | FT |
| `ld ft,(de)` | D/E - DE | F/T | FT |
| `ld ft,(hl)` | H/L - HL | F/T | FT |
| `ld (bc),ft` | F/T/B/C - FT/BC | B/C | BC |
| `ld (de),ft` | F/T/D/E - FT/DE | D/E | DE |
| `ld (hl),ft` | F/T/H/L - FT/HL | H/L | HL |
| `ld ft,(bc+)` | B/C - BC | F/T/B/C | FT/BC |
| `ld ft,(de+)` | D/E - DE | F/T/D/E | FT/DE |
| `ld ft,(hl+)` | H/L - HL | F/T/H/L | FT/HL |
| `ld (bc+),ft` | F/T/B/C - FT/BC | B/C | BC |
| `ld (de+),ft` | F/T/D/E - FT/DE | D/E | DE |
| `ld (hl+),ft` | F/T/H/L - FT/HL | H/L | HL |
| `ld ft,(-bc)` | B/C - BC | F/T/B/C | FT/BC |
| `ld ft,(-de)` | D/E - DE | F/T/D/E | FT/DE |
| `ld ft,(-hl)` | H/L - HL | F/T/H/L | FT/HL |
| `ld (-bc),ft` | F/T/B/C - FT/BC | B/C | BC |
| `ld (-de),ft` | F/T/D/E - FT/DE | D/E | DE |
| `ld (-hl),ft` | F/T/H/L - FT/HL | H/L | HL |

**Expansions:**
- `LD FT, (R16)` → `LD T, (R16)` / `ADD R16, 1` / `EXG F,T` / `LD T, (R16)` / `ADD R16, -1` / `EXG F,T`
- `LD (R16), FT` → `LD (R16), T` / `ADD R16, 1` / `EXG F,T` / `LD (R16), T` / `ADD R16, -1` / `EXG F,T`
- `LD FT, (R16+)` → `LD T, (R16)` / `ADD R16, 1` / `EXG F,T` / `LD T, (R16)` / `EXG F,T`
- `LD (R16+), FT` → `LD (R16), T` / `ADD R16, 1` / `EXG F,T` / `LD (R16), T` / `EXG F,T`
- `LD FT, (-R16)` → `LD T, (R16)` / `ADD R16, -1` / `EXG F,T` / `LD T, (R16)`
- `LD (-R16), FT` → `EXG F,T` / `LD (R16), T` / `ADD R16, -1` / `EXG F,T` / `LD (R16), T`

#### 16-bit Memory Loads/Stores through FT

Loading or storing a full 16-bit register pair through `FT` also uses synthesized sequences; these temporarily swap `F` and `T` via `EXG F,T` to avoid corrupting the `FT` pointer.

| Instruction | Reads | Writes | Clobbers |
|---|---|---|---|
| `ld bc,(ft)` | F/T - FT | B/C | BC |
| `ld de,(ft)` | F/T - FT | D/E | DE |
| `ld hl,(ft)` | F/T - FT | H/L | HL |
| `ld (ft),bc` | F/T/B/C - FT/BC | F/T | FT |
| `ld (ft),de` | F/T/D/E - FT/DE | F/T | FT |
| `ld (ft),hl` | F/T/H/L - FT/HL | F/T | FT |
| `ld bc,(ft+)` | F/T - FT | F/T/B/C | FT/BC |
| `ld de,(ft+)` | F/T - FT | F/T/D/E | FT/DE |
| `ld hl,(ft+)` | F/T - FT | F/T/H/L | FT/HL |
| `ld (ft+),bc` | F/T/B/C - FT/BC | F/T | FT |
| `ld (ft+),de` | F/T/D/E - FT/DE | F/T | FT |
| `ld (ft+),hl` | F/T/H/L - FT/HL | F/T | FT |
| `ld bc,(-ft)` | F/T - FT | F/T/B/C | FT/BC |
| `ld de,(-ft)` | F/T - FT | F/T/D/E | FT/DE |
| `ld hl,(-ft)` | F/T - FT | F/T/H/L | FT/HL |
| `ld (-ft),bc` | F/T/B/C - FT/BC | F/T | FT |
| `ld (-ft),de` | F/T/D/E - FT/DE | F/T | FT |
| `ld (-ft),hl` | F/T/H/L - FT/HL | F/T | FT |

**Expansions (for BC/DE/HL through FT indirect):**
- `LD R16, (FT)` → `LD low, (FT)` / `ADD FT, 1` / `LD high, (FT)` / `ADD FT, -1`
- `LD (FT), R16` → `LD (FT), low` / `ADD FT, 1` / `LD (FT), high` / `ADD FT, -1`
- `LD R16, (FT+)` → `LD low, (FT)` / `ADD FT, 1` / `LD high, (FT)`
- `LD (FT+), R16` → `LD (FT), low` / `ADD FT, 1` / `LD (FT), high`
- `LD R16, (-FT)` → `LD high, (FT)` / `ADD FT, -1` / `LD low, (FT)`
- `LD (-FT), R16` → `ADD FT, -1` / `LD (FT), high` / `LD (FT), low`

#### 8-bit Memory Access through FT with Post-increment/Pre-decrement

The native `LD R8, (FT)` and `LD (FT), R8` instructions transfer any 8-bit register (B, C, D, E, H, L) through the FT pointer without corrupting it — unlike `LD T, (FT)` which overwrites T. Their post-increment and pre-decrement forms are synthesized by appending or prepending `ADD FT, ±1`. `F` is excluded because `LD F, (FT)` destroys the FT pointer, and `T` is covered by the [Post-increment and Pre-decrement Memory Access](#post-increment-and-pre-decrement-memory-access) section above.

| Instruction | Reads | Writes | Clobbers |
|---|---|---|---|
| `ld b,(ft+)` | F/T - FT | B/F/T | FT/BC |
| `ld c,(ft+)` | F/T - FT | C/F/T | FT/BC |
| `ld d,(ft+)` | F/T - FT | D/F/T | FT/DE |
| `ld e,(ft+)` | F/T - FT | E/F/T | FT/DE |
| `ld h,(ft+)` | F/T - FT | H/F/T | FT/HL |
| `ld l,(ft+)` | F/T - FT | L/F/T | FT/HL |
| `ld (ft+),b` | F/T/B - FT/BC | F/T | FT |
| `ld (ft+),c` | F/T/C - FT/BC | F/T | FT |
| `ld (ft+),d` | F/T/D - FT/DE | F/T | FT |
| `ld (ft+),e` | F/T/E - FT/DE | F/T | FT |
| `ld (ft+),h` | F/T/H - FT/HL | F/T | FT |
| `ld (ft+),l` | F/T/L - FT/HL | F/T | FT |
| `ld b,(-ft)` | F/T - FT | B/F/T | FT/BC |
| `ld c,(-ft)` | F/T - FT | C/F/T | FT/BC |
| `ld d,(-ft)` | F/T - FT | D/F/T | FT/DE |
| `ld e,(-ft)` | F/T - FT | E/F/T | FT/DE |
| `ld h,(-ft)` | F/T - FT | H/F/T | FT/HL |
| `ld l,(-ft)` | F/T - FT | L/F/T | FT/HL |
| `ld (-ft),b` | F/T/B - FT/BC | F/T | FT |
| `ld (-ft),c` | F/T/C - FT/BC | F/T | FT |
| `ld (-ft),d` | F/T/D - FT/DE | F/T | FT |
| `ld (-ft),e` | F/T/E - FT/DE | F/T | FT |
| `ld (-ft),h` | F/T/H - FT/HL | F/T | FT |
| `ld (-ft),l` | F/T/L - FT/HL | F/T | FT |

**Expansions:**
- `LD R8, (FT+)` → `LD R8, (FT)` / `ADD FT, 1`
- `LD (FT+), R8` → `LD (FT), R8` / `ADD FT, 1`
- `LD R8, (-FT)` → `ADD FT, -1` / `LD R8, (FT)`
- `LD (-FT), R8` → `ADD FT, -1` / `LD (FT), R8`

#### LIO with Post-increment/Pre-decrement

Supported for any 16-bit register pair (`FT`, `BC`, `DE`, `HL`).

| Instruction | Reads | Writes | Clobbers |
|---|---|---|---|
| `lio t,(ft+)` | F/T - FT | F/T | FT |
| `lio t,(bc+)` | B/C - BC | T/B/C | FT/BC |
| `lio t,(de+)` | D/E - DE | T/D/E | FT/DE |
| `lio t,(hl+)` | H/L - HL | T/H/L | FT/HL |
| `lio (ft+),t` | F/T - FT | F/T | FT |
| `lio (bc+),t` | T/B/C - FT/BC | B/C | BC |
| `lio (de+),t` | T/D/E - FT/DE | D/E | DE |
| `lio (hl+),t` | T/H/L - FT/HL | H/L | HL |
| `lio t,(-ft)` | F/T - FT | F/T | FT |
| `lio t,(-bc)` | B/C - BC | T/B/C | FT/BC |
| `lio t,(-de)` | D/E - DE | T/D/E | FT/DE |
| `lio t,(-hl)` | H/L - HL | T/H/L | FT/HL |
| `lio (-ft),t` | F/T - FT | F/T | FT |
| `lio (-bc),t` | B/C - BC | B/C | BC |
| `lio (-de),t` | T/D/E - FT/DE | D/E | DE |
| `lio (-hl),t` | T/H/L - FT/HL | H/L | HL |

**Expansions:** `LIO` follows the same expansion pattern as the corresponding `LD` form (above): replace `LD` with `LIO` in each line. The `IO` access-type signal is asserted through each native step.

#### LCO with Post-increment/Pre-decrement

Supported for any 16-bit register pair (`FT`, `BC`, `DE`, `HL`). `LCO` is load-only (no store form).

| Instruction | Reads | Writes | Clobbers |
|---|---|---|---|
| `lco t,(ft+)` | F/T - FT | F/T | FT |
| `lco t,(bc+)` | B/C - BC | T/B/C | FT/BC |
| `lco t,(de+)` | D/E - DE | T/D/E | FT/DE |
| `lco t,(hl+)` | H/L - HL | T/H/L | FT/HL |
| `lco t,(-ft)` | F/T - FT | F/T | FT |
| `lco t,(-bc)` | B/C - BC | T/B/C | FT/BC |
| `lco t,(-de)` | D/E - DE | T/D/E | FT/DE |
| `lco t,(-hl)` | H/L - HL | T/H/L | FT/HL |

**Expansions:** load-only subset of the `LD` pattern (above): `LCO T, (R16+)` → `LCO T, (R16)` / `ADD R16, 1`; `LCO T, (-R16)` → `ADD R16, -1` / `LCO T, (R16)`. The `CODE` access-type signal is asserted through each native step.

#### Multi-register Stack Operations

| Instruction | Reads | Writes | Clobbers |
|---|---|---|---|
| `push bc-hl` | B/C/D/E/H/L - BC/DE/HL | — | — ; BC (stack)/DE (stack)/HL (stack) |
| `pop bc-hl` | — - — | B/C/D/E/H/L | BC/DE/HL ; BC (stack)/DE (stack)/HL (stack) |

**Expansions:**
- `PUSH BC-HL` → `PUSHA` / `POP FT` (push all except FT)
- `POP BC-HL` → `POPA` / `PUSH FT` (pop all except FT)
- `PUSH R16/R16` → individual `PUSH` per listed register
- `POP R16/R16` → individual `POP` per listed register
- `SWAP R16/R16` → individual `SWAP` per listed register

Register range syntax uses `-` for ranges and `/` for individual registers. Any combination is accepted, e.g., `PUSH FT/HL`, `PUSH BC-HL`, `PUSH BC-DE`.

#### EXG Register-to-Register (not involving T/FT)

| Instruction | Reads | Writes | Clobbers |
|---|---|---|---|
| `exg f,b` | F/B - FT/BC | F/B | FT/BC |
| `exg f,c` | F/C - FT/BC | F/C | FT/BC |
| `exg f,d` | F/D - FT/DE | F/D | FT/DE |
| `exg f,e` | F/E - FT/DE | F/E | FT/DE |
| `exg f,h` | F/H - FT/HL | F/H | FT/HL |
| `exg f,l` | F/L - FT/HL | F/L | FT/HL |
| `exg b,c` | B/C - BC | B/C | BC |
| `exg b,d` | B/D - BC/DE | B/D | BC/DE |
| `exg b,e` | B/E - BC/DE | B/E | BC/DE |
| `exg b,h` | B/H - BC/HL | B/H | BC/HL |
| `exg b,l` | B/L - BC/HL | B/L | BC/HL |
| `exg c,d` | C/D - BC/DE | C/D | BC/DE |
| `exg c,e` | C/E - BC/DE | C/E | BC/DE |
| `exg c,h` | C/H - BC/HL | C/H | BC/HL |
| `exg c,l` | C/L - BC/HL | C/L | BC/HL |
| `exg d,e` | D/E - DE | D/E | DE |
| `exg d,h` | D/H - DE/HL | D/H | DE/HL |
| `exg d,l` | D/L - DE/HL | D/L | DE/HL |
| `exg e,h` | E/H - DE/HL | E/H | DE/HL |
| `exg e,l` | E/L - DE/HL | E/L | DE/HL |
| `exg h,l` | H/L - HL | H/L | HL |
| `exg bc,de` | B/C/D/E - BC/DE | B/C/D/E | BC/DE |
| `exg bc,hl` | B/C/H/L - BC/HL | B/C/H/L | BC/HL |
| `exg de,hl` | D/E/H/L - DE/HL | D/E/H/L | DE/HL |

**Expansions:**
- `EXG R8, R8` → `EXG T, src` / `EXG T, dest` / `EXG T, src` (3-exchange via T)
- `EXG R16, R16` → `EXG FT, src` / `EXG FT, dest` / `EXG FT, src` (3-exchange via FT)

#### PICK with Immediate

| Instruction | Reads | Writes | Clobbers |
|---|---|---|---|
| `pick ft,i8` | F/T - FT | F/T | FT ; FT (stack) |
| `pick bc,i8` | B/C - BC | B/C | BC ; BC (stack) |
| `pick de,i8` | D/E - DE | D/E | DE ; DE (stack) |
| `pick hl,i8` | H/L - HL | H/L | HL ; HL (stack) |

**Expansion:**
- `PICK R16, imm` → `LD low_reg, imm` / `PICK R16`, where `low_reg` is the low byte of `R16`.

The immediate form loads the low byte of the register pair with the index value, then executes `PICK`. This is a shorthand that avoids manually setting the register to the index value first.

#### Template Synthesized Instructions

The following cannot be fully enumerated as they apply to any instruction or any valid register combination:

| Syntax | Expands To | Notes |
|---|---|---|
| `INST/CC operands` | `J/!CC,@+n` / `INST operands` | Any native instruction + any CC. `J/CC` itself is native — it is the primitive used to synthesize these forms. |
| `J/CC (R16)` | `J/!CC,@+2` / `J (R16)` | 10 CCs × 4 R16 = 40 permutations |
| `PUSH R16/R16` | Individual `PUSH` per register | Any subset of FT, BC, DE, HL (15 combos) |
| `POP R16/R16` | Individual `POP` per register | Any subset of FT, BC, DE, HL (15 combos) |
| `SWAP R16/R16` | Individual `SWAP` per register | Any subset of FT, BC, DE, HL (15 combos) |

**Conditional execution:** Any instruction can be made conditional by appending `/CC`. The assembler emits a native `J/!CC` (inverted condition) that skips the instruction if the condition is not met. For example, `LD/EQ T, 1` becomes `J/NE, @+2` / `LD T, 1`. The special form `J/CC (R16)` expands to `J/!CC, @+2` / `J (R16)`.

### Synthesized Instruction Notes

- **Expansion**: Each synthesized instruction expands to a safe sequence of native instructions with no unintended side effects. The exact expansion may vary based on assembler optimizations.
- **SUB synthesized forms**: Do **not** set F flags — only `CMP` and `TST` set flags.
- **Post-increment/pre-decrement**: The register is modified as a side effect of the synthesized sequence.
- **INST/CC**: The `!CC` is the inverse condition (for example, `EQ` → `NE`, `LT` → `GE`).

---

## Practical Tips and Common Idioms

Core syntax and assembler conventions are in [Quick Start](#quick-start). This section focuses on usage patterns and idioms.

### Conditional Execution — `INST/CC`
Any instruction can be made conditional by appending `/CC`. The assembler emits a `J/!CC` that skips the instruction when the condition is not met.

| Pattern | Expands To | Use Case |
|---------|-----------|----------|
| `LD/GTU C,T` | — - — ; `J/LEU/@+2`/`LD C/T` | — - — ; Unsigned max: `C = max(C, T)` |
| `LD/LTU C,T` | — - — ; `J/GEU/@+2`/`LD C/T` | — - — ; Unsigned min: `C = min(C, T)` |
| `LD/EQ T,1` | — - — ; `J/NE/@+2`/`LD T/1` | — - — ; Conditional assignment |
| `INC/NE T` | — - — ; `J/EQ/@+2`/`INC T` | — - — ; Conditional increment |

This is the preferred idiom for single-instruction conditionals — it avoids branching overhead and is more compact than a full if-then-else.

### DJ — Decrement-Then-Test

`DJ R8, label` decrements the register **first**, then tests for zero — load the exact iteration count (no pre-decrement). For the full pseudocode, nested-DJ idiom for counts > 256, and the canonical loop pattern, see [DJ R8, s8](#dj-r8-s8).

### DJ Jump-to-End Trick
When the first iteration seeds a running value (max, min, accumulator, etc.), you can avoid pre-decrementing the counter and checking for empty or length=1 by jumping directly to the `DJ` instruction at the bottom. `DJ` decrements first, so with count=1 it decrements to 0 and exits immediately — no out-of-bounds access, no flag manipulation needed:

```
        ld      b,count          ; B = length (exact count, no adjustment)
        ; ... set up pointer (e.g. DE) ...
        ; ... process first element, seed running value ...
        j       .loop_end        ; jump straight to DJ
.loop
        ; ... process next element, update running value ...
.loop_end
        dj      b,.loop          ; DJ handles length=1 naturally
        ; ... extract result ...
```

See the [Examples](#examples) section below for concrete, fully-commented implementations (max, sum, popcount, sort, recursion, stack-pointer manipulation, etc.).

### H and L as Loop Counters

H and L are the preferred registers for `DJ` loop counters. Because HL is callee-saved, the callee must push/pop HL anyway. Using H or L as the counter means the `push`/`pop` that preserves HL also preserves the remaining iterations — no extra save/restore overhead for the counter itself.

This is especially valuable when T is used as an input parameter or scratch register, and BC/DE are occupied with pointers or accumulators.

```
        ; T = count, BC = pointer
        push    bc/de/hl
        ld      h,t             ; counter in H (preserved by push/pop hl)
.loop
        ld      t,(bc+)         ; T is scratch — safe to clobber
        ; ... process T ...
        dj      h,.loop         ; H decrements, preserved across any calls
        pop     hl/de/bc
        j       (hl)
```

When both H and L are needed (e.g., nested loops), they can each hold a counter independently since `DJ` operates on the 8-bit register without affecting the other byte of HL.

#### Splitting H/L Between Counter and Accumulator

The same independence lets H and L serve two *different* roles at once: one half as the `DJ` loop counter, the other half as a single-byte running value (max, min, running total's low byte, etc.). Since HL is pushed/popped as a pair regardless, this fits an entire "scan array, track one running value" function into just `push bc/hl` / `pop hl/bc` — no `DE` needed at all:

```
        ; T = length, BC = pointer
        push    bc/hl

        ld      h,t             ; H = loop counter = length
        ld      t,(bc+)         ; T = arr[0]
        ld      l,t             ; L = running value, seeded from arr[0]
        j       .loop_end       ; DJ jump-to-end trick handles length=1

.loop
        ld      t,(bc+)         ; T = arr[i]
        cmp     t,l             ; F <- Flags[T - L]
        ld/gtu  l,t             ; L = T if T > L (unsigned) — running max

.loop_end
        dj      h,.loop

        ld      t,l             ; T = return value

        pop     hl/bc
        j       (hl)
```

This combines three idioms — DJ jump-to-end, H/L as counter, and `INST/CC` conditional assignment — into a minimal-register-pressure scan loop. Prefer this pattern over reaching for `DE` whenever a function needs exactly one loop counter and one accumulator alongside a pointer parameter.

### SUB Does Not Set Flags
`SUB R8, i8` is synthesized as `ADD R8, -i8`, which does **not** set the F register flags. Only `CMP` and `TST` set flags. If you need to test the result of a subtraction, use `CMP` instead.

### Zero-Initialization
`LD FT, 0` is synthesized by the assembler as `LD T, 0` / `EXT` (3 bytes total), sign-extending the zero in T into F. This is the most compact way to clear FT.

### PUSH/POP FT as a One-Instruction Temporary

`PUSH FT` / `POP FT` is the most compact way to hold a single byte across a few instructions when no other register is free. Although `FT` is caller-saved, it is still a valid operand for the stack instructions. Use it as a temporary in short regions where no comparison or other `FT`-clobbering operation occurs between the save and the restore. A common use is swapping two bytes through memory.

### Pointer-Based Subranges

For routines that walk a byte array, represent subranges as pointer pairs (`start`, `end`) rather than base-plus-index. This avoids the repeated `FT = base + index` address computations required by the T/FT hub and keeps the inner loop compact. Recursive divide-and-conquer algorithms naturally accept pointer bounds, which also simplifies parameter passing.

### SWAP FT as Dual Accumulator

`SWAP FT` exchanges the full 16-bit FT register with the next entry on the FT stack, effectively giving you two working accumulators in the FT domain. This is useful when BC and DE are both occupied as pointers and a second scratch value is needed. `PUSH FT` first creates the second slot, then `SWAP FT` alternates between the two values. Remember that `SWAP FT` clobbers both F and T — extract any byte you need to keep (via `LD R8,T` or `LD R8,F`) before swapping.

```
        ; FT = acc_a (incoming), DE = seed for acc_b
        push    ft              ; stack[1] = acc_a, FT = acc_a
        ld      ft,de           ; FT = acc_b
        ; FT = acc_b, stack[1] = acc_a

        swap    ft              ; FT = acc_a, stack[1] = acc_b
        add     ft,bc           ; FT = acc_a + BC
        swap    ft              ; FT = acc_b, stack[1] = acc_a + BC
        add     ft,de           ; FT = acc_b + DE

        ; Collect: FT = acc_b + DE, stack[1] = acc_a + BC
        swap    ft              ; FT = acc_a + BC, stack[1] = acc_b + DE
        ld      de,ft           ; DE = acc_a + BC
        pop     ft              ; FT = acc_b + DE, stack cleaned
```

This fits two independent 16-bit running values into the FT register and its stack, leaving BC and DE free for pointer duty. Avoid this pattern when a comparison or other F-writing instruction appears between swaps — the flags will be destroyed by the next `SWAP FT`.

### EXG Techniques

Native `EXG` exchanges T with any 8-bit register, or FT with any 16-bit register. The assembler synthesizes `EXG R8,R8` and `EXG R16,R16` for any other pair using a 3-exchange through the hub — but the native primitives are the building blocks for everything.

**3-exchange via T**: swap any two non-T registers by routing through T. T is restored:

```
        ; swap B and D (T restored)
        exg     t,b            ; T=oldB, B=oldT
        exg     t,d            ; T=oldD, D=oldB
        exg     t,b            ; T=oldT, B=oldD
```

**EXG vs LD**: `EXG` swaps (both values preserved, 3 instructions via hub). `LD` through the hub copies (source preserved, destination overwritten, 2 instructions). Use EXG when you need to reorganize values without losing either; use LD when you only need the destination.

**Register permutation — cycle technique**: any permutation decomposes into disjoint cycles. A k-cycle `(r1 r2 r3 ... rk)` (r1 gets rk's value, r2 gets r1's, etc.) executes in exactly k exchanges by streaming values through T. The first exchange parks T's original value in r1; subsequent exchanges deposit each carried value into the next register; the final exchange with r1 restores T:

```
        ; Cycle (B D E): B gets E, D gets B, E gets D
        ; Before: T=x, B=b, D=d, E=e
        exg     t,b            ; T=b,  B=x    (park T's original in B)
        exg     t,d            ; T=d,  D=b    (stream b into D)
        exg     t,e            ; T=e,  E=d    (stream d into E)
        exg     t,b            ; T=x,  B=e    (stream e into B, restore T)
        ; After:  B=e, D=b, E=d, T=x
```

k registers, k exchanges, T restored. A 2-cycle `(B D)` is the 3-exchange swap: `exg t,b; exg t,d; exg t,b`. Arbitrary permutations chain disjoint cycles sequentially — T restores after each cycle, so they compose cleanly.

**Byte reorganization within FT**: `EXG F,T` is native — it swaps the two bytes of FT. Use it for byte-by-byte 16-bit operations without saving to another register:

```
        ; 16-bit AND: FT &= BC (byte by byte)
        and     t,c            ; T = T & C (low byte)
        exg     f,t            ; F = low result, T = old F (high byte)
        and     t,b            ; T = T & B (high byte)
        exg     f,t            ; F = high result, T = low result
        ; FT now contains the 16-bit AND result
```

**Swap FT:FT' with BC:BC'**: exchange the top two entries of the FT stack with the top two of the BC stack, using six native instructions:

```
        ; Before: FT=a, FT'[1]=a'  |  BC=b, BC'[1]=b'
        swap    ft              ; FT=a'
        exg     ft,bc           ; FT=b, BC=a'
        swap    ft              ; FT=a, FT'[1]=b
        swap    bc              ; BC=b', BC'[1]=a'
        exg     ft,bc           ; FT=b', BC=a
        swap    ft              ; FT=b, FT'[1]=b'
        ; After:  FT=b, FT'[1]=b'  |  BC=a, BC'[1]=a'
```

See [`examples/exg_techniques.md`](examples/exg_techniques.md) for fully traced examples.

### Four Independent Stacks as Scratch Space

The FT, BC, DE, and HL stacks are fully independent 16-bit word stacks. While BC, DE, and HL are primarily used for callee-saved register storage, each stack can serve as independent scratch space. `PUSH BC` does not affect the DE or HL stacks, so you can use one stack as a temporary buffer, another for parameter passing, and still another for loop counters — all without interfering. `PICK R16, imm` gives O(1) indexed access into any stack, and `LCR` can read or write the stack pointer directly for bulk rewind or depth queries. When data-segment memory is scarce, the register stacks are a natural place to park intermediate values that do not need to survive a function return.

### LD R8,(FT) for Pointer-Safe Loads

The `LD R8,(FT)` instruction family loads a byte from memory at the address in FT directly into any 8-bit register — F, T, B, C, D, E, H, or L — without modifying FT. This is the preferred way to read through an FT pointer when the pointer must survive the load. The complementary `LD (FT),R8` family stores any 8-bit register to memory at FT without modifying FT. To advance the pointer, follow with `ADD FT,1`.

```
        ; Load two consecutive bytes into B and C, FT preserved
        ld      b,(ft)          ; B = mem[FT]
        add     ft,1            ; FT = FT + 1
        ld      c,(ft)          ; C = mem[FT]
        add     ft,1            ; advance for next use
```

Load into T is also supported (`LD T,(FT)`) but overwrites the low byte of FT; use it only when the pointer is no longer needed or has been saved. For the full list of available destinations, see the `LD R8,(FT)` rows in the Native Instruction Table.

### Anti-Patterns: Do and Don't

#### Don't: Push result after a binary stack operation

Because the register IS the stack top (see [Register-Stack Architecture](#register-stack-architecture)), binary operations leave the result in the register, which is already the stack top. Adding `push ft` after the operation duplicates the result — the stack ends up one entry deeper than it should be.

**Anti-pattern:**
```
        ld      bc,ft        ; BC = b (TOS)
        pop     ft           ; FT = a (2nd)
        add     ft,bc        ; FT = a + b
        push    ft           ; WRONG: duplicates result, stack: [..., a+b, a+b]
```

**Correct:**
```
        ld      bc,ft        ; BC = b (TOS)
        pop     ft           ; FT = a (2nd)
        add     ft,bc        ; FT = a + b (TOS), stack: [..., a+b]
```

The only time `push ft` is correct after an operation is when the operation is meant to *preserve* the original TOS (e.g., computing a value while keeping the stack unchanged). For standard `( a b -- result )` semantics, no push is needed.

#### Don't: Load via T when `LD R8, (FT)` is available

**Anti-pattern:**
```
        ld t,(ft)
        ld b,t
```

**Why it's wrong:** `LD T, (R16)` overwrites `T`, which is the low byte of the `FT` pair. If `FT` is being used as a pointer, this destroys the pointer. Even when `FT` is not a pointer, routing through `T` wastes an instruction cycle and a byte when `LD R8, (FT)` can load directly into the destination.

**Preferred:**
```
        ld b,(ft)      ; B = *(FT), FT pointer preserved
```

The `LD R8, (FT)` instruction family (`00101rrr`) loads directly into any 8-bit register (`F`, `T`, `B`, `C`, `D`, `E`, `H`, `L`) without corrupting `FT`. Reserve `LD T, (R16)` for cases where `R16` is not `FT`, or when you genuinely do not need the `FT` value after the load.

When you need two consecutive bytes through `FT`:
```
        ld b,(ft)      ; B = *(FT), FT pointer preserved
        add ft,1
        ld c,(ft)      ; C = *(FT+1), FT pointer preserved
```

This is especially useful after computing a target address in `FT` where `T` held intermediate data that has already been consumed (e.g., `sub t,1` to compute an offset, then `ld c,(ft)` to read the value at that address).

#### Don't: Attempt any direct register-to-register transfer without T or FT

**Anti-pattern:**
```
        ld      de,bc           ; ILLEGAL — no such instruction exists
        ld      c,b             ; ILLEGAL — no such instruction exists
        ld      e,d             ; ILLEGAL — no such instruction exists
```

**Every register transfer must involve `T` (8-bit) or `FT` (16-bit).** There is no opcode for moving data between any two other registers. For the canonical `T`/`FT` routing code patterns (e.g., `ld t,b` / `ld c,t`, or `ld ft,bc` / `ld de,ft`), see the [Register Transfer](#register-transfer) section and [The T Hub](#the-t-hub) in Quick Start.

This is a common gotcha when copying a pointer from a callee-saved register to a scratch register. The assembler will not silently accept `LD DE, BC` — it is an illegal instruction.

### Callee-Saved Discipline

When writing a function that may call other functions, remember:
- **Must save/restore**: BC, DE, HL (callee-saved).
- **No save needed**: FT (caller-saved, always assumed destroyed).
- **No save needed**: F and T (part of caller-saved FT).
- **Must save/restore the containing pair**: if you modify B or C, you have modified BC; if you modify D or E, you have modified DE; if you modify H or L, you have modified HL. Save and restore the 16-bit pair, not the individual byte.

A practical approach: if you need BC for pointer arithmetic but must preserve it, save and restore BC around the work with push/pop.

### Optimizations

#### Eliminate redundant accumulator round-trips

When computing an offset that is already in T, avoid copying it to another register just to subtract. Operate on T directly when the original value has already been consumed. See the [Register Transfer](#register-transfer) section for the full T/FT hub rule.

```
        ; Before: 3 instructions to compute count-1 in E
        ld      d,t             ; D = count
        sub     d,1             ; D = count - 1
        ld      t,d             ; T = count - 1
        ld      e,t             ; E = count - 1

        ; After: 2 instructions — subtract in-place on T
        sub     t,1             ; T = count - 1
        ld      e,t             ; E = count - 1
```

This pairs well with the `LD R8, (FT)` form: after `sub t,1` the low byte of FT is the offset, and `LD R8, (FT)` reads through the computed address without needing T to hold the result.

### Recursion Patterns

#### Saving Parameters Across Recursive Calls

A parameter passed in T must be saved in a callee-saved register (B, C, D, E, H, or L) before the first recursive call, so it can be recovered to compute further arguments:

```
        ; T = n, need fib(n-1) and fib(n-2)
        ld      b,t             ; save n in callee-saved B
        sub     t,1
        jal     fibonacci       ; FT = fib(n-1)
        push    ft              ; save fib(n-1)
        ld      t,b             ; restore n
        sub     t,2
        jal     fibonacci       ; FT = fib(n-2)
```

B is preferred because it is part of BC (already pushed/popped at function boundaries), so no extra save/restore overhead.

#### Saving Multiple Parameters with PICK

When a recursive routine needs more than one parameter preserved across a recursive call, the independent register stacks are the cleanest storage. Push the parameter register, then use `PICK R16, imm` to retrieve it. Because `PICK` uses only the low byte of the register as the stack index, the immediate form is safe even with 16-bit pointer values. For example, after pushing `BC` and `DE` at function entry, `pick bc,1` restores the saved `BC` and `pick de,1` restores the saved `DE` without disturbing the stack.

#### POP vs SWAP for Stack Retrieval

`POP R16` increments the stack pointer and loads the new top into the register. This retrieves a stacked value **and** cleans up the stack in one instruction — preferred over `SWAP` followed by `POP` to discard the swapped-out entry:

```
        ; stack: [fib(n-1)], FT = fib(n-2)
        ld      bc,ft           ; BC = fib(n-2)
        pop     ft              ; FT = fib(n-1), stack cleaned
        add     ft,bc           ; FT = fib(n-1) + fib(n-2)
```

```
        ; Anti-pattern: swap then pop to discard
        ld      bc,ft           ; BC = fib(n-2)
        swap    ft              ; FT = fib(n-1), stack: [fib(n-2)]
        add     ft,bc           ; FT = fib(n-1) + fib(n-2)
        pop     ft              ; discard fib(n-2) — extra instruction
```

#### Stack Discipline in Recursive Functions

Every `PUSH FT` must have a matching `POP FT` before the function returns. A missed pop leaks one stack entry per call frame, which accumulates and eventually causes stack overflow. A practical rule: count pushes and pops in the recursive path — they must match.

#### Check Base Cases Before Pushing Callee-Saved Registers

If a recursive function has a trivial base case, test it before pushing callee-saved registers. The leaf path can then return after saving only the return address. Pushing `BC`, `DE`, and other callee-saved registers only when recursion is actually needed avoids unnecessary stack traffic and keeps the fast path compact.

#### Commutative Operations Avoid Double-Swap

When combining two values where one is on the stack and one is in FT, save FT first, then `POP` the stacked value. Do not `SWAP` to reorder operands for a commutative operation like `ADD`:

```
        ; stack: [a], FT = b
        ld      bc,ft           ; BC = b
        pop     ft              ; FT = a
        add     ft,bc           ; FT = a + b  (same as b + a)
```

The double-swap pattern (`swap ft; ld bc,ft; swap ft; add ft,bc`) reverses the operands of `ADD` unnecessarily — addition is commutative, so the reversal wastes an instruction.

## Examples

Concrete, fully-commented implementations of common patterns and idioms live in the [`examples/`](examples/) directory. Each file documents the algorithm, register allocation rationale, and the full source with inline comments.

| Example | File | Demonstrates |
|---|---|---|
| Max of an array | [`u8_max.md`](examples/u8_max.md) | DJ jump-to-end trick; H/L split between counter and running value; `INST/CC` conditional assignment |
| Sum of u8 array into u16 | [`sum_u8_array.md`](examples/sum_u8_array.md) | 16-bit accumulator in DE with FT as scratch add register; zero-extension via `LD F,0` |
| Multiprecision add | [`add_mp.md`](examples/add_mp.md) | F-as-carry-lane pattern; source added to destination in place; in-out pointers in BC/DE; two 16-bit adds per byte; `PUSH BC` / `POP BC` saves the destination pointer around scratch reuse; `LD (BC),T` / `ADD BC,1` store-and-advance; carry extracted via `LD T,F` / `LD L,T` |
| Bubble sort | [`bubble_sort_u8.md`](examples/bubble_sort_u8.md) | Nested DJ loops (H outer, L inner); saving the base pointer on the BC stack around an inner loop that clobbers C |
| Quicksort | [`quicksort_u8.md`](examples/quicksort_u8.md) | Recursion with callee-saved stack frames; `PICK` to recover saved base/count; `LD R8,(FT)` to load pivot without clobbering T; FT stack to preserve intermediate values across recursive calls |
| Recursive Fibonacci | [`fibonacci.md`](examples/fibonacci.md) | Saving parameters in callee-saved B across calls; `PUSH FT`/`POP FT` to stack intermediate results; `SWAP`/`POP` idiom for retrieval |
| Population count (PICK table) | [`popcount_array.md`](examples/popcount_array.md) | `PICK` as O(1) lookup table; building a 16-entry table on the DE stack; nibble splitting; break-even analysis vs. naive per-bit loop |
| LCR stack pointer manipulation | [`lcr_stack_pointer.md`](examples/lcr_stack_pointer.md) | Reading/writing register stack pointers via `LCR`; bulk discard/rewind in 3 instructions; stack depth query |
| Nibble to ASCII hex | [`nibble_to_hex.md`](examples/nibble_to_hex.md) | `INST/CC` conditional add; `'0'` base offset + conditional 7 for 'A'-'F' shift; no callee-saved registers needed |
| setjmp/longjmp via LCR | [`setjmp_longjmp.md`](examples/setjmp_longjmp.md) | Saving/restoring all 4 register stack pointers via `LCR`; `PUSH HL` to capture return address; PUSH-duplicate survival across `LD C,i8` clobbering; `ADD T,1` to compensate for prologue pushes contaminating `SP_FT`, `SP_BC`, `SP_HL`; `PUSH FT`/`POP FT` to save the buffer pointer across the `SP_BC` restore (since `LD C,i8` clobbers `Stack[SP_BC]` at the restore point) |
| Computed branch via FLAGS table | [`computed_branch.md`](examples/computed_branch.md) | 256-entry `FLAGS_*` dispatch table in code space; `LCO T,(FT)` lookup; `LD F,T` flag injection; `J/CC` chain with most-specific-first ordering; disjoint-range opcode dispatch where a value table cannot branch and a `CMP` cascade bloats |
| RPN stack machine (Forth-like) | [`rpn_stack_machine.md`](examples/rpn_stack_machine.md) | FT stack as Forth data stack; `PUSH FT` = `dup`, `POP FT` = `drop`, `SWAP FT` = `swap`, `PICK FT` = `n@`; BC as second-operand holding cell; operand pattern for all binary operations; full token-based interpreter dispatch |
| Stack parameter passing | [`stack_params.md`](examples/stack_params.md) | HL stack parameter spillover; caller push order with sacrificial slot; callee `PUSH HL` / swap-pop parameter access; `PUSH FT` / `LD FT,HL` epilogue to save result and restore return address; nested call handling |
