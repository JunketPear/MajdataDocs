# Expansion and Shorthand Rules

For convenience, Slide Code allows consecutive identical instructions to be omitted. The parser expands the code into a sequence of `(instruction, parameter)` pairs.

## Expansion Rule

If a numeric parameter is not preceded by an instruction letter, search backward until the most recently encountered instruction is found, and use that instruction.

## Example 1

* **Written:**

  `5Q9A1P98CQ49K5`

* **Fully expanded:**

  `(X)5 - Q9 - A1 - P9 - P8 - C - Q4 - Q9 - K5`

* **Explanation:** In `P98`, the `8` has no instruction letter immediately before it, so the parser searches backward and finds `P`, resulting in `P8`.

  Likewise, the leading `5` has no preceding instruction and is therefore interpreted as the starting point `X5`.

## Example 2

* **Written:**

  `1A3571P9K1`

* **Fully expanded:**

  `(X)1 - A3 - A5 - A7 - A1 - P9 - K1`

* **Explanation:** The shorthand rule makes it convenient to describe paths that repeatedly move between positions in the A area.

---
