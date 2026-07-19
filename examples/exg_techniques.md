# EXG Techniques

Native `EXG` exchanges T with any 8-bit register, or FT with any 16-bit register. The assembler synthesizes `EXG R8,R8` and `EXG R16,R16` for any other pair, but this file uses only native primitives.

## 3-Exchange via T

Swap any two non-T registers by routing through T. T is restored at the end:

```
        ; Initial: T=$10, B=$20, D=$30
        exg     t,b            ; T=$20, B=$10
        exg     t,d            ; T=$30, D=$20
        exg     t,b            ; T=$10, B=$30
        ; Result:  B=$30, D=$20, T=$10 (restored)
```

T is restored to its original value. The 3-exchange swaps B and D while leaving T unchanged.

The same pattern works for 16-bit via FT:

```
        ; swap BC and DE via FT
        exg     ft,bc          ; FT=oldBC, BC=oldFT
        exg     ft,de          ; FT=oldDE, DE=oldBC
        exg     ft,bc          ; FT=oldBC, BC=oldDE
```

## Register Permutation

Any permutation decomposes into swaps. All examples below use native EXG only.

### Cycle B → D → E → B

B receives E's value, D receives B's, E receives D's. This is a 3-cycle `(B D E)` — it executes in 4 exchanges by streaming values through T. The first exchange parks T's original value in B; subsequent exchanges deposit each carried value into the next register; the final exchange with B restores T:

```
        ; Initial: T=x, B=$11, D=$22, E=$33
        exg     t,b            ; T=$11, B=x    (park T's original in B)
        exg     t,d            ; T=$22, D=$11  (stream $11 into D)
        exg     t,e            ; T=$33, E=$22  (stream $22 into E)
        exg     t,b            ; T=x,   B=$33  (stream $33 into B, restore T)
        ; Result:  B=$33, D=$11, E=$22, T=x
```

k-cycle = k exchanges. A 2-cycle `(B D)` is the 3-exchange swap: `EXG T,B; EXG T,D; EXG T,B`. Arbitrary permutations decompose into disjoint cycles and chain them sequentially — T restores after each cycle.

### Swap B and E via T

```
        ; Initial: T=x, B=$11, E=$44
        exg     t,b            ; T=$11, B=x
        exg     t,e            ; T=$44, E=$11
        exg     t,b            ; T=x,    B=$44
        ; Result:  B=$44, E=$11, T=x (restored)
```

## Byte Reorganization Within FT

`EXG F,T` is native — it swaps the two bytes of FT. Use it to operate on both bytes without saving to another register.

### 16-bit AND (FT &= BC)

```
        ; FT=$ABCD, BC=$1234
        and     t,c            ; T = $CD & $34 = $04 (low byte)
        exg     f,t            ; F=$04, T=$AB (high byte exposed)
        and     t,b            ; T = $AB & $12 = $02 (high byte)
        exg     f,t            ; F=$02, T=$04 (restore order)
        ; FT=$0204
```

### Byte Swap Within FT

```
        ; FT=$ABCD
        exg     f,t            ; FT=$CDAB
```

## Swap FT:FT' with BC:BC'

Exchange the top two entries of the FT stack with the top two of the BC stack. All six instructions are native:

```
        ; Before: FT=$0001, FT'[1]=$0002  |  BC=$0003, BC'[1]=$0004
        swap    ft              ; FT=$0002, FT'[1]=$0001
        exg     ft,bc           ; FT=$0003, BC=$0002
        swap    ft              ; FT=$0001, FT'[1]=$0003
        swap    bc              ; BC=$0004, BC'[1]=$0002
        exg     ft,bc           ; FT=$0004, BC=$0001
        swap    ft              ; FT=$0003, FT'[1]=$0004
        ; After:  FT=$0003, FT'[1]=$0004  |  BC=$0001, BC'[1]=$0002
```

### Trace

| Step | FT | FT'[1] | BC | BC'[1] |
|------|----|--------|----|--------|
| Start | $0001 | $0002 | $0003 | $0004 |
| `SWAP FT` | $0002 | $0001 | $0003 | $0004 |
| `EXG FT,BC` | $0003 | $0001 | $0002 | $0004 |
| `SWAP FT` | $0001 | $0003 | $0002 | $0004 |
| `SWAP BC` | $0001 | $0003 | $0004 | $0002 |
| `EXG FT,BC` | $0004 | $0003 | $0001 | $0002 |
| `SWAP FT` | $0003 | $0004 | $0001 | $0002 |

The pattern alternates `SWAP` and `EXG` to walk both stacks in lockstep, exchanging corresponding entries. The final `SWAP FT` restores correct ordering.

### Trace with 32-bit values

When FT:FT' holds a 32-bit value (low word in FT, high word in FT'), this pattern swaps two 32-bit values between the FT and BC stacks:

```
        ; FT:FT' = $00020001 (32-bit value, little-endian)
        ; BC:BC' = $00040003 (32-bit value, little-endian)
        swap    ft              ; FT=$0002
        exg     ft,bc           ; FT=$0003, BC=$0002
        swap    ft              ; FT=$0001, FT'[1]=$0003
        swap    bc              ; BC=$0004, BC'[1]=$0002
        exg     ft,bc           ; FT=$0004, BC=$0001
        swap    ft              ; FT=$0003, FT'[1]=$0004
        ; FT:FT' = $00040003
        ; BC:BC' = $00020001
```
