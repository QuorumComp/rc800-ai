# SUGGESTIONS.md

Recommended algorithms to implement on the RC800 platform.

Each entry includes:
- **Why**: why it's a good choice
- **Techniques**: RC800-specific patterns it exercises
- **Difficulty**: estimated effort

---

## Already Implemented

| File | Algorithm |
|---|---|
| `add_mp` | Multi-precision add |
| `fibonacci` | Recursive Fibonacci |
| `bubble_sort_u8` | Bubble sort |
| `quicksort_u8` | Quicksort (Lomuto) |
| `rpn_stack_machine` | RPN calculator |
| `nibble_to_hex` | 4-bit → ASCII hex |
| `popcount_array` | Population count via PICK |
| `exg_techniques` | EXG swaps |
| `computed_branch` | FLAGS table dispatch |
| `lcr_stack_pointer` | LCR stack pointer manipulation |
| `setjmp_longjmp` | setjmp/longjmp via LCR |
| `stack_params` | Stack parameter spillover |
| `strchr` | Character search |
| `strcmp` | String comparison |
| `strlen` | String length |
| `sum_u8_array` | 16-bit accumulator sum |
| `u8_max` | Array max (DJ jump-to-end) |
| `binary_search_u8` | Binary search |

---

## Recommended Implementations

### Tier 1 — Core Fundamentals (start here)

#### 1. GCD (Euclidean Algorithm)
- **Why**: Classic divide-and-conquer, teaches repeated modulo via subtraction
- **Techniques**: Modulo via repeated subtraction, flag-based branching, 16-bit arithmetic
- **Difficulty**: Low
- **Registers**: FT for dividend, BC for divisor, DE as 16-bit scratch

#### 2. Modular Exponentiation
- **Why**: Binary exponentiation, teaches bit-checking and shift-and-add
- **Techniques**: Bit masking, shift-and-add loop, 16-bit overflow handling
- **Difficulty**: Low-Medium
- **Registers**: FT for result, BC for base, DE for exponent, T for scratch

#### 3. Merge Sort
- **Why**: Alternative to quicksort, teaches divide-and-conquer with merging
- **Techniques**: Recursion, temporary storage, merge step with two pointers
- **Difficulty**: Medium
- **Registers**: BC/DE for pointers, FT for temporary storage, H/L for counters

#### 4. KMP String Search
- **Why**: Better than `strchr`, teaches preprocessing and state machine
- **Techniques**: Prefix function computation, linear scan, pointer arithmetic
- **Difficulty**: Medium
- **Registers**: BC for text pointer, DE for pattern pointer, T for scratch

#### 5. Radix Sort (on u8 array)
- **Why**: Non-comparison sort, teaches counting sort as subroutine
- **Techniques**: Multiple passes, counting sort, stable sort, pointer management
- **Difficulty**: Medium
- **Registers**: BC for array pointer, DE for count array, FT for accumulation

---

### Tier 2 — Data Structures

#### 7. Binary Search Tree
- **Why**: Pointer-heavy recursion, teaches linked data structures
- **Techniques**: Insert/delete/search, node allocation, recursive traversal
- **Difficulty**: Medium-High
- **Registers**: FT for node pointer, BC for child pointers, DE for comparison

#### 8. Largest Rectangle in Histogram
- **Why**: Stack-based algorithm, teaches monotonic stack
- **Techniques**: Stack usage, while-loop with stack pops, area calculation
- **Difficulty**: Medium
- **Registers**: BC for array pointer, FT for stack operations, DE for area accumulator

#### 9. Sieve of Eratosthenes
- **Why**: Prime generation, teaches nested loops and bit flags
- **Techniques**: Marking multiples, bit manipulation, array bounds
- **Difficulty**: Medium
- **Registers**: BC for main pointer, DE for inner loop, T for scratch

#### 10. Union-Find with Path Compression
- **Why**: Efficient disjoint set, teaches pointer chasing and path compression
- **Techniques**: Recursive find, path compression, union by rank
- **Difficulty**: Medium
- **Registers**: FT for root pointer, BC for node, DE for rank

---

### Tier 3 — Advanced Algorithms

#### 11. Dijkstra's Algorithm
- **Why**: Graph shortest path, teaches priority queue simulation
- **Techniques**: Adjacency list, distance array, stack-based "priority queue"
- **Difficulty**: High
- **Registers**: FT for distance, BC for current node, DE for neighbor, H/L for counters

#### 12. Huffman Coding
- **Why**: Data compression, teaches tree building and traversal
- **Techniques**: Min-heap simulation, tree building, bit packing
- **Difficulty**: High
- **Registers**: FT for node pointer, BC/DE for children, H/L for counters

#### 13. Aho-Corasick Multi-Pattern Search
- **Why**: Multi-pattern string search, teaches automaton construction
- **Techniques**: Failure links, state machine, trie traversal
- **Difficulty**: High
- **Registers**: BC for text pointer, DE for state pointer, FT for transition

#### 14. K-Nearest Neighbors
- **Why**: ML primitive, teaches distance calculation and k-selection
- **Techniques**: Distance computation, sorting, k-selection algorithm
- **Difficulty**: Medium-High
- **Registers**: FT for distance, BC/DE for point coordinates, H/L for counters

#### 15. Memory Allocator (Free List)
- **Why**: Pointer manipulation, teaches memory management
- **Techniques**: Free list traversal, block splitting, coalescing
- **Difficulty**: High
- **Registers**: FT for block pointer, BC for list head, DE for next pointer

---

## Design Decisions & Patterns to Practice

### Register Allocation Patterns

| Pattern | When to Use | Example |
|---|---|---|
| **T/FT hub** | All data flows through T or FT | `LD T,(BC+)` / `ADD FT,DE` |
| **BC pointer + T scratch** | Array traversal | `strchr`, `strlen`, `strchr` |
| **BC pointer + DE pointer** | Dual array traversal | `strcmp`, `memcpy` |
| **H/L as pair** | Loop counter + accumulator | `u8_max`, `sum_u8_array` |
| **F as stash** | Preserve byte across CMP | `u8_max` uses `LD H,T` instead |
| **DE as accumulator** | 16-bit sum without FT clobber | `sum_u8_array` |
| **FT stack intermediates** | Save across recursive calls | `fibonacci`, `quicksort_u8` |

### Loop Patterns

| Pattern | Use Case | Example |
|---|---|---|
| **DJ jump-to-end** | Seed first element, then loop | `u8_max` |
| **Nested DJ** | Count > 255 | `add_mp` |
| **DJ with flag check** | Early exit | `strlen` |
| **DJ with accumulator** | Running total | `sum_u8_array` |

### Stack Patterns

| Pattern | Use Case | Example |
|---|---|---|
| **PUSH/POP callee-saved** | Preserve BC/DE across calls | All functions |
| **PUSH FT intermediate** | Save result across recursive call | `fibonacci` |
| **PICK fixed index** | Read stack param by index | `stack_params` |
| **LCR SP adjust** | Bulk discard/restore | `popcount_array`, `setjmp` |

### Flag Survival Patterns

| Pattern | Why It Works | Example |
|---|---|---|
| **CMP → add T → J/CC** | `add T` writes T only, F preserved | `nibble_to_hex` |
| **CMP → LD F → J/CC** | `LD F` writes F only, T preserved | `computed_branch` |
| **CMP → LD R8 → J/CC** | `LD R8` writes R8 only, F preserved | `u8_max` |

---

## Suggested Order

1. **GCD** — teaches modulo, flag usage
2. **Modular Exponentiation** — bit-checking loops
3. **Merge Sort** — recursion + merge step
4. **KMP** — preprocessing + linear scan
5. **Radix Sort** — multiple passes + counting sort
7. **Binary Search Tree** — pointer-heavy recursion
8. **Largest Rectangle** — stack-based algorithm
9. **Sieve** — nested loops + bit flags
10. **Union-Find** — path compression
11. **Dijkstra's** — graph algorithm
12. **Huffman** — tree building + compression
13. **Aho-Corasick** — automaton
14. **KNN** — distance + k-selection
15. **Memory Allocator** — pointer management
