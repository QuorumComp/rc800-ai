# RC800 AI Documentation

Documentation and code examples for the RC800 (RC8xx) CPU architecture.

## ⚠️ AI-Generated Content

This repository is **largely AI-generated**. The documentation and examples were produced by an AI assistant using the RC800 architecture specification as input. While care has been taken to verify correctness against the native instruction table and architecture rules, **review all content critically** before relying on it. If you find errors, please report them.

## 🤖 For LLMs

**[`REFERENCE.md`](REFERENCE.md)** is the single authoritative source for the RC800 ISA. It is written as a comprehensive prompt so an LLM can learn the architecture in one read — covering registers, calling convention, assembler syntax, the full native instruction table, synthesized instructions, and coding idioms. It also serves as a good introduction and tutorial for human developers.

Feed `REFERENCE.md` to your context window, then consult the [`examples/`](examples/) directory for concrete implementations of common patterns.

## What Is This?

The RC800 is a hypothetical 8-bit CPU architecture exploring what a mid-to-late 1980s CPU might look like if FWIW (Fixed Width Instruction Word) and RISC techniques had been applied. It features:

- **8-bit FWIW opcodes** — every instruction is exactly 8 bits, with optional trailing bytes fetched during execution
- **T/FT hub architecture** — all data flows through the T (8-bit) or FT (16-bit) accumulator
- **Four independent register stacks** — BC, DE, HL, and FT each have their own 256-word on-chip stack
- **Deterministic execution** — every instruction completes in exactly 4 clock cycles

## Files

| File | Description |
|------|-------------|
| [`REFERENCE.md`](REFERENCE.md) | **Single authoritative source** for the ISA. See [For LLMs](#-for-llms) above. |
| [`SUGGESTIONS.md`](SUGGESTIONS.md) | Recommended algorithms to implement on the RC800 platform, organized by difficulty tier. |
| [`AGENTS.md`](AGENTS.md) | Instructions for AI assistants working in this repository. |

## Examples

The [`examples/`](examples/) directory contains concrete, fully-commented code implementations of common patterns and idioms:

| Example | Demonstrates |
|---------|-------------|
| `add_mp` | Multi-precision addition, F-as-carry-lane pattern |
| `binary_search_u8` | Binary search on sorted arrays |
| `bubble_sort_u8` | Nested DJ loops, BC stack for pointer preservation |
| `computed_branch` | FLAGS table dispatch via LCO |
| `exg_techniques` | EXG swap patterns, register permutation |
| `fibonacci` | Recursive Fibonacci with FT stack intermediates |
| `lcr_stack_pointer` | LCR-based stack pointer manipulation |
| `nibble_to_hex` | Conditional add, INST/CC pattern |
| `popcount_array` | PICK as O(1) lookup table |
| `quicksort_u8` | Recursive quicksort with callee-saved stack frames |
| `rpn_stack_machine` | Forth-like RPN calculator |
| `setjmp_longjmp` | Non-local jumps via LCR |
| `stack_params` | HL stack parameter spillover |
| `strchr` | Character search |
| `strcmp` | String comparison |
| `strlen` | String length |
| `sum_u8_array` | 16-bit accumulator sum |
| `u8_max` | Array max, DJ jump-to-end trick |

## Related

The RC800 RTL implementation lives in the [`rc800`](https://github.com/QuorumComp/rc800) repository. Architecture documentation for the RTL is in `docs/`.

## License

MIT License. See [LICENSE](LICENSE).
