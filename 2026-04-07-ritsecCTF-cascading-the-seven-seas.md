# Cascading the Seven Seas

**Category:** Misc/Reversing
**Author:** sy1vi3
**Flag:** `RS{CR3D1T_T0_LYR4_R3B4N3_F1BDF5}`

## Overview

The challenge presents a webpage at `https://css.ctf.ritsec.club/` that runs an
interactive "Pirate Trivia" game — implemented entirely in CSS. No JavaScript.
The page uses bleeding-edge CSS features (`@property`, `@function`, `:has()`
selectors, and `if()`) to emulate a 16-bit x86 CPU in pure stylesheets.

## Analysis

### Extracting the program

The page contains ~724 KB of CSS. Memory is modeled as CSS custom properties
`--m0` through `--m8463`, each declared with `@property` and an
`initial-value`. Dumping these gives us a flat memory image:

- **Address 0:** `0xCC` (INT 3 — halt)
- **Addresses 256–1217:** x86-16 machine code
- **Addresses 1218–1452:** String data (prompts, messages)
- **Addresses 8192–8463:** I/O mapped memory (screen buffer, keyboard state)

### Disassembling the code

Using Capstone in 16-bit mode (base address `0x100`), the program disassembles
into a straightforward trivia game:

1. **Entry point (`0x301`):** Sets up I/O function pointers, calls main.
2. **`main` (`0x248`):** Prints welcome banner, then asks three questions:
   - Q1: "Which ocean is the largest?" — expects 7-character answer
   - Q2: "Name an aquatic mammal" — expects 5-character answer
   - Q3: "What's the flag?" — expects 32-character answer
3. **`print_str` (`0x100`):** Prints a null-terminated string 8 bytes at a time.
4. **`read_input` (`0x11e`):** Reads up to 34 characters from the CSS keyboard,
   one at a time (RETURN = newline = submit).
5. **`validate` (`0x1a6`):** The core check function.

### Understanding the validation

The `validate` function at `0x1a6` takes three arguments on the stack:
`(input_buf, table_ptr, entry_count)`. It iterates over a table of 8-byte
entries, each containing three index bytes and an expected result:

```
struct entry {
    uint16_t idx0;  // XOR operand index
    uint16_t idx1;  // ADD operand 1 index
    uint16_t idx2;  // ADD operand 2 index
    uint16_t expected;
};
```

For each entry, the check is:

```
((input[idx1] + input[idx2]) ^ input[idx0]) & 0xFF == expected
```

If all entries pass, validation succeeds (returns 0); otherwise it fails early.

### Extracting the flag table

For Q3 (the flag), the table starts at address `0x320` with 32 entries. Each
entry references three character positions in the input and an expected byte.
This gives us a system of 32 equations over 32 unknowns.

### Known values

The flag format is `RS{...}`, giving us:
- `input[0] = 'R'` (0x52)
- `input[1] = 'S'` (0x53)
- `input[2] = '{'` (0x7B)
- `input[31] = '}'` (0x7D)

## Solution

With 32 equations, 32 unknowns, and 4 known values, Z3 solves the constraint
system instantly.

```py
from z3 import *

# The CSS page implements a full x86-16 CPU emulator. Memory is stored as CSS
# custom properties (--m0 through --m8463). The program is a "pirate trivia"
# game; question 3 asks for the flag (32 chars).
#
# The flag validation function at 0x1a6 iterates over a table of 8-byte entries:
#   [word ptr0, word ptr1, word ptr2, word expected]
# For each entry it checks:
#   ((input[ptr1] + input[ptr2]) ^ input[ptr0]) & 0xFF == expected
#
# The flag table lives at address 0x320 with 32 entries (one per character).

TABLE = [
    (18, 12, 25, 247), ( 5, 11,  0, 177), (14, 20, 28, 223), ( 6, 23, 12, 214),
    (28,  3, 15, 209), ( 2,  1,  4, 222), (14, 27,  3, 220), ( 1, 24, 19, 193),
    (29,  7, 22,  57), ( 8,  9,  6, 247), ( 6, 27, 30,  51), (18, 10,  6, 202),
    (10, 28,  3, 211), (16, 21, 26,  81), (12, 20, 24, 254), (11, 10,  4, 150),
    (13, 28, 17, 239), ( 2, 15, 12, 202), (12, 19, 18, 218), ( 4, 27, 30,  37),
    ( 6, 17, 26, 212), (17, 14, 16, 210), (31, 27, 17, 220), (31, 18, 29, 229),
    (13, 25,  7,  59), (28, 18, 10, 226), (31, 30,  8, 244), ( 7,  5,  9, 163),
    (16, 28, 30,  77), (27, 12,  6, 225), ( 5, 27, 28, 181), (31, 18, 10, 219),
]

FLAG_LEN = 32

flag = [BitVec(f"f{i}", 16) for i in range(FLAG_LEN)]

s = Solver()

# Known prefix/suffix: RS{...}
s.add(flag[0] == ord("R"))
s.add(flag[1] == ord("S"))
s.add(flag[2] == ord("{"))
s.add(flag[31] == ord("}"))

# Printable ASCII
for v in flag:
    s.add(v >= 0x20, v <= 0x7E)

# Validation constraints
for p0, p1, p2, exp in TABLE:
    s.add(((flag[p1] + flag[p2]) ^ flag[p0]) & 0xFF == exp)

assert s.check() == sat
m = s.model()
print("".join(chr(m[flag[i]].as_long()) for i in range(FLAG_LEN)))

```
