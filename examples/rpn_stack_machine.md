# RPN Stack Machine: Forth-like Calculator on the FT Stack

The RC800 is a natural stack machine. The FT register has its own 256-word
independent stack with native `PUSH` (duplicate), `POP` (drop), `SWAP`
(exchange with next entry), and `PICK` (indexed access). Combined with BC as a
working operand register, the architecture implements a complete Forth-like RPN
calculator without any memory-based stack pointer management.

The FT stack is the data stack. `FT` is the top-of-stack (TOS). Every binary
operation follows the same pattern: copy TOS to BC, pop the second operand into
FT, operate. The result is already in FT which is TOS.

### Register Allocation

| Register | Role | Rationale |
|----------|------|-----------|
| FT | TOS / accumulator | All arithmetic targets FT; `PUSH`/`POP`/`SWAP`/`PICK` operate on FT stack |
| BC | Second operand | Callee-saved; holds one operand while FT operates on the other |
| DE | Temporary / scratch | Callee-saved; available for multi-step operations |
| H | Loop counter | Callee-saved; `DJ` loop counter for multiply bit loop |
| T | 8-bit scratch | Part of FT; used for loading immediates and 8-bit transfers |

### Stack Primitives

Every Forth stack manipulation word maps to one or two native instructions.
`PUSH FT` duplicates the current value — it is Forth's `dup`, not a
conventional push. `POP FT` drops TOS and loads the next value — it is Forth's
`drop`, not a conventional pop.

| Forth word | Stack effect | RC800 | Cycles |
|---|---|---|---|
| `dup` | `( a -- a a )` | `PUSH FT` | 4 |
| `drop` | `( a -- )` | `POP FT` | 4 |
| `swap` | `( a b -- b a )` | `SWAP FT` | 4 |
| `over` | `( a b -- a b a )` | `PUSH FT` / `PICK FT,2` | 8 |
| `nip` | `( a b -- b )` | `SWAP FT` / `POP FT` | 8 |
| `tuck` | `( a b -- b a b )` | `SWAP FT` / `PUSH FT` / `PICK FT,2` | 12 |
| `2dup` | `( a b -- a b a b )` | `PUSH FT` / `PUSH FT` | 8 |
| `rot` | `( a b c -- b c a )` | 6 instructions via BC | 24 |
| `n@` | `( ... n -- ... n )` | `PICK FT,n` | 4 |

`PICK FT` uses the low byte of FT as the stack index. Index 0 is the register
itself (TOS), index 1 is the last pushed value, index 2 is the one before that.
The assembler immediate form `PICK FT,n` loads the index and executes in two
instructions.

### The Operand Pattern

Every binary operation follows the same four-instruction skeleton. BC is the
holding cell — callee-saved, 16-bit, and never part of the FT stack:

```
        ld      bc,ft      ; BC = TOS (b)
        pop     ft         ; FT = 2nd (a), drops TOS
        ; ... operate on FT and BC ...
        ; result is in FT which is TOS
```

`EXG FT,BC` is useful when the operation needs the operands reversed:

```
        ld      bc,ft      ; BC = b
        pop     ft         ; FT = a
        exg     ft,bc      ; FT = b, BC = a
        ; ... now FT has TOS, BC has 2nd ...
```

### Why This Is Natural

1. **No SP management** — each register has its own 256-word stack. `PUSH FT`
   and `POP FT` never corrupt BC, DE, or HL stacks.
2. **PUSH is dup** — `PUSH FT` duplicates the top value. `dup` is a single
   4-cycle instruction, not a memory round-trip.
3. **PICK is native** — `PICK FT` gives O(1) access to any stack depth. No
   loop, no pointer arithmetic.
4. **SWAP is native** — `SWAP FT` exchanges register with stack[1] in one
   instruction.
5. **FT is the accumulator** — all arithmetic targets FT, which is exactly the
   TOS. No extra register transfers needed.
6. **BC is the coprocessor** — callee-saved, 16-bit, perfect for holding the
   second operand while FT operates.
7. **256 entries** — deep enough for serious expression evaluation without
   stack overflow.

### Stack Trace: `3 4 + 5 *`

**Note:** The stack notation shows logical contents. FT is TOS (index 0). Stack
entries below FT are at indices 1, 2, 3, etc.

```
; Push 3 (first value, stack is empty)
        ld      ft,3         ; FT=3,    stack: [3] (just TOS)

; Push 4 (push current TOS, then load new value)
        push    ft           ; SP decrements, Stack[SP]=3 (duplicate)
        ld      ft,4         ; FT=4,    stack: [3, 4]

; Execute +  (a=3, b=4 -- a+b=7)
        ld      bc,ft        ; BC=4,    stack: [3, 4]
        pop     ft           ; FT=3,    stack: [3]
        add     ft,bc        ; FT=7,    stack: [7] (result is TOS)

; Push 5
        push    ft           ; SP decrements, Stack[SP]=7 (duplicate)
        ld      ft,5         ; FT=5,    stack: [7, 5]

; Execute *  (a=7, b=5 -- a*b=35)
        ld      bc,ft        ; BC=5,    stack: [7, 5]
        pop     ft           ; FT=7,    stack: [7]
        ; ... multiply FT by BC ...
        ; FT=35,   stack: [35] (result is TOS)
```

### Arithmetic Words

Basic arithmetic uses the operand pattern with a single operation instruction.

```
; --
; -- rpn_add
; --   ( a b -- a+b )  16-bit signed/unsigned addition
; --
; -- Inputs:
; --   FT: b (TOS)
; --   FT stack: a (2nd)
; --
; -- Outputs:
; --   FT: a+b (TOS)
; --
        ld      bc,ft        ; BC = b
        pop     ft           ; FT = a
        add     ft,bc        ; FT = a + b (TOS)
        j       rpn_dispatch ; return to dispatch loop

; --
; -- rpn_sub
; --   ( a b -- a-b )  16-bit signed/unsigned subtraction
; --
; -- Inputs:
; --   FT: b (TOS)
; --   FT stack: a (2nd)
; --
; -- Outputs:
; --   FT: a-b (TOS)
; --
        ld      bc,ft        ; BC = b
        pop     ft           ; FT = a
        sub     ft,bc        ; FT = a - b (TOS)
        j       rpn_dispatch ; return to dispatch loop

; --
; -- rpn_neg
; --   ( a -- -a )  Negate TOS in place
; --
; -- Inputs:
; --   FT: a (TOS)
; --
; -- Outputs:
; --   FT: -a (TOS)
; --
        neg     ft           ; FT = -a (TOS)
        j       rpn_dispatch ; return to dispatch loop

; --
; -- rpn_abs
; --   ( a -- |a| )  Absolute value, replaces TOS in place
; --
; -- Inputs:
; --   FT: a (TOS)
; --
; -- Outputs:
; --   FT: |a| (TOS)
; --
; -- Clobbers: BC
; --
        ld      bc,ft        ; BC = a (save before tst clobbers F)
        tst     ft           ; F = Flags[FT - 0] (clobbers F)
        j/ge    .rpn_abs_pos ; if a >= 0, skip negation
        ld      ft,bc        ; restore FT (was clobbered by tst)
        neg     ft           ; FT = -a
        j       rpn_dispatch
.rpn_abs_pos
        ld      ft,bc        ; restore FT (was clobbered by tst)
        j       rpn_dispatch ; return to dispatch loop
```

### Bitwise Words

Bitwise operations on 16-bit values require byte-by-byte processing since the
native `AND`/`OR`/`XOR` only operate on 8-bit T. The operand is saved to BC,
then each byte is processed through T with `EXG F,T` to swap results into
place:

```
; --
; -- rpn_and16
; --   ( a b -- a AND b )  16-bit bitwise AND
; --
; -- Inputs:
; --   FT: b (TOS)
; --   FT stack: a (2nd)
; --
; -- Outputs:
; --   FT: a & b (TOS)
; --
; -- Clobbers: BC
; --
        ld      bc,ft        ; BC = b
        pop     ft           ; FT = a
        and     t,c          ; T = a_low & b_low
        exg     f,t          ; F = a_low & b_low, T = a_high
        and     t,b          ; T = a_high & b_high
        exg     f,t          ; F = a_high & b_high, T = a_low & b_low
        j       rpn_dispatch ; return to dispatch loop

; --
; -- rpn_or16
; --   ( a b -- a OR b )  16-bit bitwise OR
; --
; -- Clobbers: BC
; --
        ld      bc,ft        ; BC = b
        pop     ft           ; FT = a
        or      t,c          ; T = a_low | b_low
        exg     f,t          ; F = a_low | b_low, T = a_high
        or      t,b          ; T = a_high | b_high
        exg     f,t          ; F = a_high | b_high, T = a_low | b_low
        j       rpn_dispatch ; return to dispatch loop

; --
; -- rpn_xor16
; --   ( a b -- a XOR b )  16-bit bitwise XOR
; --
; -- Clobbers: BC
; --
        ld      bc,ft        ; BC = b
        pop     ft           ; FT = a
        xor     t,c          ; T = a_low ^ b_low
        exg     f,t          ; F = a_low ^ b_low, T = a_high
        xor     t,b          ; T = a_high ^ b_high
        exg     f,t          ; F = a_high ^ b_high, T = a_low ^ b_low
        j       rpn_dispatch ; return to dispatch loop

; --
; -- rpn_invert16
; --   ( a -- ~a )  16-bit bitwise NOT, replaces TOS in place
; --
; -- Inputs:
; --   FT: a (TOS)
; --
; -- Outputs:
; --   FT: ~a (TOS)
; --
        not     ft           ; FT = ~a (TOS)
        j       rpn_dispatch ; return to dispatch loop

; --
; -- rpn_lshift
; --   ( a n -- a << n )  Left shift by n (masked to 4 bits, 0-15)
; --
; -- Inputs:
; --   FT: n (TOS, shift count)
; --   FT stack: a (2nd, value)
; --
; -- Outputs:
; --   FT: a << n (TOS)
; --
; -- Clobbers: BC
; --
        ld      bc,ft        ; BC = n (shift count)
        pop     ft           ; FT = a (value)
        ls      ft,c         ; FT = a << (n & 0xF) (TOS)
        j       rpn_dispatch ; return to dispatch loop

; --
; -- rpn_rshift
; --   ( a n -- a >> n )  Logical right shift by n (masked to 4 bits, 0-15)
; --
; -- Inputs:
; --   FT: n (TOS, shift count)
; --   FT stack: a (2nd, value)
; --
; -- Outputs:
; --   FT: a >> n (TOS)
; --
; -- Clobbers: BC
; --
        ld      bc,ft        ; BC = n (shift count)
        pop     ft           ; FT = a (value)
        rs      ft,c         ; FT = a >> (n & 0xF) (TOS)
        j       rpn_dispatch ; return to dispatch loop

; --
; -- rpn_arshift
; --   ( a n -- a >>> n )  Arithmetic right shift by n (masked to 4 bits, 0-15)
; --
; -- Inputs:
; --   FT: n (TOS, shift count)
; --   FT stack: a (2nd, value)
; --
; -- Outputs:
; --   FT: a >>> n (TOS)
; --
; -- Clobbers: BC
; --
        ld      bc,ft        ; BC = n (shift count)
        pop     ft           ; FT = a (value)
        rsa     ft,c         ; FT = a >>> (n & 0xF) (TOS)
        j       rpn_dispatch ; return to dispatch loop
```

### Multiply

Unsigned 16x16 multiply producing a 16-bit result (low word). Uses
shift-and-add: for each set bit in the multiplier, adds the multiplicand to
the running product.

```
; --
; -- rpn_mul
; --   ( a b -- a*b )  Unsigned 16x16 -> u16 (low word of product)
; --   Shift-and-add: for each set bit in b, adds (a << bit_pos) to product.
; --
; -- Inputs:
; --   FT: b (TOS, multiplier)
; --   FT stack: a (2nd, multiplicand)
; --
; -- Outputs:
; --   FT: (a * b) & $FFFF (TOS)
; --
; -- Clobbers: BC, DE, H
; --
        ld      de,ft        ; DE = b (multiplier)
        pop     ft           ; FT = a (multiplicand)
        ld      bc,ft        ; BC = a (for shifting)
        ld      ft,0         ; FT = product = 0
        ld      h,16         ; H = bit counter
.rpn_mul_loop
        ; Check lowest bit of multiplier (E)
        ld      t,e          ; T = b_low
        and     t,1          ; T = b & 1
        cmp     t,0
        add/gtu  ft,bc       ; if bit set, add multiplicand to product
        ; Shift multiplicand left and multiplier right via FT
        ; (ISA only has LS/RS on FT; save product on FT stack)
        push    ft           ; save product
        ld      ft,bc        ; FT = multiplicand
        ls      ft,1         ; FT = multiplicand << 1
        ld      bc,ft        ; BC = shifted multiplicand
        ld      ft,de        ; FT = multiplier
        rs      ft,1         ; FT = multiplier >> 1
        ld      de,ft        ; DE = shifted multiplier
        pop     ft           ; restore product
        dj      h,.rpn_mul_loop
        j       rpn_dispatch ; return to dispatch loop
```

Trace with a=3, b=4:

| Iter | BC (a<<n) | DE (b>>n) | E&1 | FT (product) |
|------|-----------|-----------|-----|-------------|
| 0    | $0003     | $0004     | 0   | $0000       |
| 1    | $0006     | $0002     | 0   | $0000       |
| 2    | $000C     | $0001     | 1   | $000C       |
| 3+   | shifts    | $0000     | 0   | $000C       |

Result: FT = $000C = 12. Correct: 3 * 4 = 12.

Trace with a=5, b=7:

| Iter | BC (a<<n) | DE (b>>n) | E&1 | FT (product) |
|------|-----------|-----------|-----|-------------|
| 0    | $0005     | $0007     | 1   | $0005       |
| 1    | $000A     | $0003     | 1   | $000F       |
| 2    | $0014     | $0001     | 1   | $0023       |
| 3+   | shifts    | $0000     | 0   | $0023       |

Result: FT = $0023 = 35. Correct: 5 * 7 = 35.

### Division

Unsigned 16-bit division returning quotient only. Uses repeated subtraction —
simple and correct, but O(quotient) iterations. A binary restoring division
would be faster for large quotients but significantly more complex.

```
; --
; -- rpn_div
; --   ( a b -- a/b )  Unsigned 16-bit division, quotient only
; --   Repeated subtraction. Division by zero returns $FFFF.
; --
; -- Inputs:
; --   FT: b (TOS, divisor)
; --   FT stack: a (2nd, dividend)
; --
; -- Outputs:
; --   FT: a / b (TOS)
; --
; -- Clobbers: BC, DE
; --
        ld      bc,ft        ; BC = b (divisor)
        pop     ft           ; FT = a (dividend)
        cmp     bc,0         ; division by zero check
        j/ne    .rpn_div_ok
        ld      ft,$ffff
        j       .rpn_div_end
.rpn_div_ok
        ld      de,0         ; DE = quotient = 0
.rpn_div_sub
        cmp     ft,bc        ; FT >= BC?
        j/ltu  .rpn_div_done
        sub     ft,bc        ; FT -= BC
        add     de,1         ; quotient++
        j       .rpn_div_sub
.rpn_div_done
        ld      ft,de        ; FT = quotient (TOS)
.rpn_div_end
        j       rpn_dispatch ; return to dispatch loop
```

### Comparison Words

Comparison words pop two operands, compare them, and produce a flag result
($00 = false, $01 = true) as TOS.

```
; --
; -- rpn_equal
; --   ( a b -- flag )  flag = 1 if a == b, 0 otherwise
; --
; -- Clobbers: BC
; --
        ld      bc,ft        ; BC = b
        pop     ft           ; FT = a
        cmp     ft,bc        ; F = Flags[a - b]
        ld      t,0          ; T = 0 (false)
        ld/eq    t,1         ; if equal, T = 1
        ld      f,0          ; F = 0 (high byte, FT = $00:flag)
        j       rpn_dispatch ; return to dispatch loop

; --
; -- rpn_gt
; --   ( a b -- flag )  Signed: flag = 1 if a > b, 0 otherwise
; --
; -- Clobbers: BC
; --
        ld      bc,ft        ; BC = b
        pop     ft           ; FT = a
        cmp     ft,bc        ; F = Flags[a - b]
        ld      t,0          ; T = 0 (false)
        ld/gt    t,1         ; if a > b (signed), T = 1
        ld      f,0          ; F = 0 (high byte, FT = $00:flag)
        j       rpn_dispatch ; return to dispatch loop

; --
; -- rpn_gtu
; --   ( a b -- flag )  Unsigned: flag = 1 if a > b, 0 otherwise
; --
; -- Clobbers: BC
; --
        ld      bc,ft        ; BC = b
        pop     ft           ; FT = a
        cmp     ft,bc        ; F = Flags[a - b]
        ld      t,0          ; T = 0 (false)
        ld/gtu   t,1         ; if a > b (unsigned), T = 1
        ld      f,0          ; F = 0 (high byte, FT = $00:flag)
        j       rpn_dispatch ; return to dispatch loop
```

### Stack-Manipulation Words

Stack-manipulation words (`DUP`, `DROP`, `SWAP`, `OVER`, `ROT`) change stack
depth or order naturally using native stack operations.

```
; --
; -- rpn_dup
; --   ( a -- a a )  Duplicate TOS
; --
        push    ft           ; stack: [..., a, a]
        j       rpn_dispatch ; return to dispatch loop

; --
; -- rpn_drop
; --   ( a -- )  Drop TOS
; --
        pop     ft           ; stack: [...]
        j       rpn_dispatch ; return to dispatch loop

; --
; -- rpn_swap
; --   ( a b -- b a )  Swap top two elements
; --
        swap    ft           ; stack: [..., b, a]
        j       rpn_dispatch ; return to dispatch loop

; --
; -- rpn_over
; --   ( a b -- a b a )  Copy 2nd to TOS
; --
        push    ft           ; stack: [a, b, b], FT = b
        pick    ft,2         ; stack: [a, b, a], FT = a
        j       rpn_dispatch ; return to dispatch loop

; --
; -- rpn_rot
; --   ( a b c -- b c a )  Rotate top three elements
; --
; -- Clobbers: BC
; --
        ld      bc,ft        ; BC = c
        pop     ft           ; FT = b
        swap    ft           ; FT = a
        exg     ft,bc        ; FT = c, BC = a
        push    ft           ; stack: [..., c, c, b]
        ld      ft,bc        ; FT = a
        j       rpn_dispatch ; return to dispatch loop
```

### Literal Push

Pushing a constant value onto the stack is a two-instruction sequence:

```
        push    ft           ; create slot (duplicate current TOS)
        ld      ft,value     ; FT = value (new TOS)
```

For the first value (empty stack), just load FT directly:

```
        ld      ft,value     ; FT = value (TOS, stack empty)
```

For 8-bit literals that fit in T, the assembler optimizes `LD FT, i16` when
the high byte is zero:

```
        push    ft           ; create slot
        ld      ft,42        ; expands to: LD T,42 / EXT (FT = $002A)
```

### Complete Interpreter Dispatch

A simple interpreter reads a byte from a token stream and dispatches to the
appropriate word. BC holds the token pointer, FT is the data stack:

```
; --
; -- rpn_interpret
; --   Interprets a sequence of RPN tokens from memory.
; --   Token 0xFF terminates execution.
; --
; --   Token encoding:
; --     0x00-0xFF (non-reserved): push immediate byte value
; --     0x80: rpn_add
; --     0x81: rpn_sub
; --     0x82: rpn_mul
; --     0x83: rpn_div
; --     0x84: rpn_neg
; --     0x85: rpn_dup
; --     0x86: rpn_drop
; --     0x87: rpn_swap
; --     0x88: rpn_over
; --     0x89: rpn_rot
; --     0x8A: rpn_and16
; --     0x8B: rpn_or16
; --     0x8C: rpn_xor16
; --     0x8D: rpn_equal
; --     0x8E: rpn_gtu
; --     0x8F: rpn_lshift
; --     0x90: rpn_rshift
; --     0xFF: halt (end of program)
; --
; -- Inputs:
; --   BC: pointer to token stream
; --
; -- Outputs:
; --   FT: final TOS value
; --
; -- Clobbers: DE, H
; --
        push    bc/de/hl     ; save callee-saved registers

rpn_dispatch
        ld      t,(bc+)      ; T = next token
        cmp     t,$ff        ; halt token?
        j/eq    .rpn_halt
        cmp     t,$80        ; reserved opcode range?
        j/ltu  .rpn_literal  ; < 0x80 -> push immediate
        ; Dispatch to word handlers (compare T, not FT)
        cmp     t,$80
        j/eq    rpn_add
        cmp     t,$81
        j/eq    rpn_sub
        cmp     t,$82
        j/eq    rpn_mul
        cmp     t,$83
        j/eq    rpn_div
        cmp     t,$84
        j/eq    rpn_neg
        cmp     t,$85
        j/eq    rpn_dup
        cmp     t,$86
        j/eq    rpn_drop
        cmp     t,$87
        j/eq    rpn_swap
        cmp     t,$88
        j/eq    rpn_over
        cmp     t,$89
        j/eq    rpn_rot
        cmp     t,$8A
        j/eq    rpn_and16
        cmp     t,$8B
        j/eq    rpn_or16
        cmp     t,$8C
        j/eq    rpn_xor16
        cmp     t,$8D
        j/eq    rpn_equal
        cmp     t,$8E
        j/eq    rpn_gtu
        cmp     t,$8F
        j/eq    rpn_lshift
        cmp     t,$90
        j/eq    rpn_rshift
        j       rpn_dispatch ; unknown token, skip
.rpn_literal
        ; T already holds the value from the load above
        push    ft           ; create slot (duplicate current TOS)
        ld      f,0          ; F = 0 (high byte)
        ; T still holds the value from the load above
        j       rpn_dispatch
.rpn_halt
        pop     hl/de/bc     ; restore callee-saved registers
        j       (hl)
```

### Example Program: `(2 + 3) * 4`

Token stream for `(2 + 3) * 4`:

```
        DB      2, 3, 0x80, 4, 0x82, 0xFF
        ;       2  3  +     4  *    HALT
```

Stack trace:

| Token | Action | Stack |
|-------|--------|-------|
| 2 | push 2 | [2] |
| 3 | push 3 | [2, 3] |
| 0x80 | add | [5] |
| 4 | push 4 | [5, 4] |
| 0x82 | mul | [20] |
| 0xFF | halt | FT = 20 |

### Example Program: `5 1 - 4 * 3 2 - /`

Token stream for `5 1 - 4 * 3 2 - /`:

```
        DB      5, 1, 0x81, 4, 0x82, 3, 2, 0x81, 0x83, 0xFF
        ;       5  1  -     4  *     3  2  -    /   HALT
```

Stack trace:

| Token | Action | Stack |
|-------|--------|-------|
| 5 | push 5 | [5] |
| 1 | push 1 | [5, 1] |
| 0x81 | sub | [4] |
| 4 | push 4 | [4, 4] |
| 0x82 | mul | [16] |
| 3 | push 3 | [16, 3] |
| 2 | push 2 | [16, 3, 2] |
| 0x81 | sub | [16, 1] |
| 0x83 | div | [16] |
| 0xFF | halt | FT = 16 |

Result: `(5-1) * 4 / (3-2)` = `4 * 4 / 1` = 16. Correct.

### Extended Words

Additional Forth words that compose from the primitives:

| Forth word | Stack effect | Implementation | Cycles |
|---|---|---|---|
| `2drop` | `( a b -- )` | `POP FT` / `POP FT` | 8 |
| `2dup` | `( a b -- a b a b )` | `PUSH FT` / `PUSH FT` | 8 |
| `2swap` | `( a b c d -- c d a b )` | requires complex implementation via BC/DE/HL | 60+ |
| `nip` | `( a b -- b )` | `SWAP FT` / `POP FT` | 8 |
| `tuck` | `( a b -- b a b )` | `PUSH FT` / `SWAP FT` | 8 |
| `pick` | `( xn...x1 x0 n -- xn...x1 x0 xn )` | `PUSH FT` then `PICK FT,n` | 8+ |
| `?dup` | `( n -- n\|0 n\|0 )` | `PUSH FT` / `TST FT` / `POP/NE FT` | 12 |
| `depth` | `( ... -- ... n )` | `LCR T,(C)` with C=RC8_SP_FT, then `NOT T` | 12 |

`DEPTH` reads the FT stack pointer via `LCR` and inverts it to get the number
of entries. See [`lcr_stack_pointer.md`](lcr_stack_pointer.md) for details on
SP manipulation.

### Limitations

- **Stack depth**: 256 words per register stack. The FT stack is the data
  stack; BC, DE, and HL stacks are independent and can be used for additional
  storage.
- **Multiply**: produces only the low 16-bit word of the product. Full 32-bit
  multiply requires the `ft:ft'` convention with `SWAP FT` for the high word.
- **Division**: the subtraction-based version is O(quotient). For better
  performance, implement binary restoring division (16 iterations regardless
  of quotient magnitude).
- **No native carry flag**: arithmetic does not set flags. Use the F-as-carry
  lane pattern (see [`add_mp.md`](add_mp.md)) for multi-precision arithmetic.
