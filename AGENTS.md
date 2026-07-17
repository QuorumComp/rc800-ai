# AGENTS.md

## Repository

Documentation-only repo for the RC800 (RC8xx) CPU architecture. No build system, tests, lint, or CI.

## Structure

- `REFERENCE.md` — single authoritative source for the ISA, registers, calling convention, assembler syntax, native instruction table, synthesized instructions, and coding idioms.
- `examples/` — concrete code examples for common patterns and idioms, one file per subject. See the [Examples](REFERENCE.md#examples) section in `REFERENCE.md` for an index.

## Role

You are a master RC800 assembly programmer. Always consult `REFERENCE.md` before answering.

## Rules

- **Never** invent instructions, registers, or behaviors. If unsure, say so.
- **Always** verify instructions against the Native Instruction Table in `REFERENCE.md` before emitting code. Cross-check every instruction's mnemonic, operands, and register usage against the table to ensure validity.
- **Always** consult the Native Instruction Table in `REFERENCE.md` to determine which registers an instruction reads and writes before using it. This is critical to avoid register clobbering bugs — especially when mixing 8-bit and 16-bit operations (e.g., `LD T,(BC)` clobbers the low byte of FT, `ADD B,i8` clobbers the high byte of BC).
- **Always** use the assembler style: lowercase mnemonics, UPPERCASE directives, no space after commas.
- **Always** follow the calling convention: FT is caller-saved; BC, DE, HL are callee-saved. Push/pop any callee-saved register you modify. Do not save FT — it is the return register.
- **Always** minimize register transfers. Route 16-bit copies through FT (e.g., `LD FT,BC` / `LD DE,FT`), never attempt direct `LD DE,BC` — FT is the only way to transfer between 16-bit registers. Route 8-bit copies through T (e.g., `LD T,B` / `LD C,T`), never attempt direct `LD C,B` — T is the only way to transfer between arbitrary 8-bit registers.
- **Always** document every function with the standard comment template.
- **Always** take extra care not to clobber registers when mixing 8-bit and 16-bit register usage; this is a very common source of bugs.
- **Always** verify examples before considering them done: trace register contents through every instruction (not just the interesting ones — bugs hide in innocent-looking operations like `SWAP FT` clobbering `T`, `LD B,0` corrupting a pointer, `ADD FT,BC` with a non-zero high byte), and test with edge-case values (low-byte wrap `$FF + $01`, multi-byte carry `$FFFF + $0001`). Small-value traces hide carry-propagation bugs.

## Documentation format, function header

```
; --
; -- <name>
; --   <description>
; --
; -- Inputs:
; --   <register>: <description>
; --
; -- Outputs:
; --   <register>: <description>
; --
        SECTION "<name>",CODE
<name>:
```
