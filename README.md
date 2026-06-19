#  AI Code Review Examples

A curated set of **buggy Python functions** paired with detailed analysis, edge case documentation, corrected implementations, and automated tests. Designed as a portfolio piece for AI training and code evaluation roles.

---

## Purpose

This repository demonstrates the ability to:
- **Identify bugs** that don't always raise obvious errors
- **Reason about edge cases** systematically
- **Produce corrected code** that is safe, typed, and documented
- **Write tests** that verify both the fix and the original failure

---

## Skills Demonstrated

| Skill | Usage |
|---|---|
| **Bug identification** | 6 distinct bug patterns across real-world scenarios |
| **Edge case analysis** | Concrete failing inputs documented per example |
| **Code correction** | Type hints, docstrings, input validation, clean logic |
| **Testing** | Inline `assert`-based tests; each covers the bug and boundary conditions |

---

## Installation

```bash
# Clone the repository
git clone https://github.com/your-username/ai-code-review-examples.git
cd ai-code-review-examples

# No external dependencies required
python ai_code_review_examples.py
```

---

## Examples

### Example 1 — `average()` · ZeroDivisionError on Empty Input

```python
# ❌ Buggy
def average(numbers):
    return sum(numbers) / len(numbers)

# ✅ Fixed
def average(numbers: list[float], default: float | None = None) -> float | None:
    if not isinstance(numbers, (list, tuple)):
        raise TypeError(f"Expected list or tuple, got {type(numbers).__name__}")
    if not numbers:
        return default
    if not all(isinstance(n, (int, float)) for n in numbers):
        raise TypeError("All elements must be numeric.")
    return sum(numbers) / len(numbers)
```

**What's wrong:** `len([])` is `0` → `ZeroDivisionError`. No type validation either.

**Edge cases:**
```
average([])          → ZeroDivisionError  (crash)
average(None)        → TypeError          (misleading message)
average(["a", "b"]) → TypeError          (inside sum, not at entry)
```

---

### Example 2 — `find_max_subarray_sum()` · Wrong Initialisation

```python
# ❌ Buggy — max_sum = 0 breaks all-negative arrays
def find_max_subarray_sum(arr):
    max_sum = 0
    current_sum = 0
    for num in arr:
        current_sum = max(num, current_sum + num)
        max_sum = max(max_sum, current_sum)
    return max_sum

# ✅ Fixed — seed with arr[0]
def find_max_subarray_sum(arr: list[int | float]) -> int | float:
    if not arr:
        raise ValueError("Array must not be empty.")
    max_sum = current_sum = arr[0]
    for num in arr[1:]:
        current_sum = max(num, current_sum + num)
        max_sum = max(max_sum, current_sum)
    return max_sum
```

**What's wrong:** Initialising `max_sum = 0` makes all-negative arrays return `0` instead of the least-negative element.

**Edge cases:**
```
[-3, -1, -2]  → 0   (WRONG, should be -1)
[-5]          → 0   (WRONG, should be -5)
```

---

### Example 3 — `add_item()` · Mutable Default Argument

```python
# ❌ Buggy — default list is shared across ALL calls
def add_item(item, items=[]):
    items.append(item)
    return items

# ✅ Fixed — use None sentinel
def add_item(item: object, items: list | None = None) -> list:
    if items is None:
        items = []
    return [*items, item]   # new list; never mutates input
```

**What's wrong:** Default `[]` is evaluated once at definition — all calls share the same object.

**Edge cases:**
```python
a = add_item("x")   # → ["x"]
b = add_item("y")   # → ["x", "y"]  ← WRONG, should be ["y"]
a is b              # → True         ← same object!
```

---

### Example 4 — `count_words()` · Silent Data Loss

```python
# ❌ Buggy — split(" ") inflates count on multiple spaces
def count_words(text):
    words = text.split(" ")
    return len(words)

# ✅ Fixed — split() with no arg handles all whitespace
def count_words(text: str) -> int:
    if not isinstance(text, str):
        raise TypeError(f"Expected str, got {type(text).__name__}")
    return len(text.split())
```

**What's wrong:** `"hello  world".split(" ")` → `["hello", "", "world"]` → `3`, not `2`.

**Edge cases:**
```
"hello  world"  → 3   (WRONG, should be 2)
"  hi  "        → 5   (WRONG, should be 1)
""              → 1   (WRONG, should be 0)
"hello\tworld"  → 1   (WRONG, tab not split)
```

---

### Example 5 — `remove_negatives()` · Mutating While Iterating

```python
# ❌ Buggy — mutating list during iteration skips elements
def remove_negatives(numbers):
    for num in numbers:
        if num < 0:
            numbers.remove(num)
    return numbers

# ✅ Fixed — list comprehension, never mutates input
def remove_negatives(numbers: list[int | float]) -> list[int | float]:
    if not isinstance(numbers, list):
        raise TypeError(f"Expected list, got {type(numbers).__name__}")
    return [n for n in numbers if n >= 0]
```

**What's wrong:** Removing items shifts indices; the iterator skips the next element. `list.remove()` also removes the first occurrence, not the current index.

**Edge cases:**
```
[-1, -2, 3]   → [-2, 3]   (WRONG, -2 skipped)
[-1, -1, 1]   → [-1, 1]   (WRONG, duplicate missed)
[-3, -3, -3]  → [-3]      (WRONG, two survive)
```

---

### Example 6 — `read_config()` · TOCTOU Race Condition

```python
# ❌ Buggy — gap between check and use; no JSON error handling
def read_config(filepath):
    if os.path.exists(filepath):
        with open(filepath) as f:
            return json.load(f)
    return {}

# ✅ Fixed — attempt open() directly; handle all failure modes
def read_config(filepath: str | Path, default: dict | None = None) -> dict:
    try:
        with open(Path(filepath), encoding="utf-8") as f:
            content = f.read().strip()
            return json.loads(content) if content else (default or {})
    except FileNotFoundError:
        if default is not None: return default
        raise ConfigError(f"Config file not found: {filepath}") from None
    except PermissionError:
        raise ConfigError(f"Permission denied reading: {filepath}") from None
    except json.JSONDecodeError as e:
        raise ConfigError(f"Invalid JSON in {filepath}: {e}") from e
```

**What's wrong:** File can be deleted between `os.path.exists()` and `open()`. Malformed JSON crashes with no useful message.

**Edge cases:**
```
File deleted between check and open  → FileNotFoundError (race, unhandled)
File contains invalid JSON           → json.JSONDecodeError (unhandled)
File exists but no read permission   → PermissionError (unhandled)
```

---

## Sample Output

```
=================================================================
  EXAMPLE 1: average()  —  Division on Empty Collection
=================================================================

BUGS FOUND
──────────
1. ZeroDivisionError when `numbers` is empty — len([]) == 0.
2. No type validation — passing strings or None raises misleading TypeError.
3. Returns raw float; callers can't distinguish "0.0" from "no data".

  ✔  average() — all tests passed

=================================================================
  EXAMPLE 2: find_max_subarray_sum()  —  Off-by-One
=================================================================
...
  ✔  find_max_subarray_sum() — all tests passed

[ examples 3–6 follow the same pattern ]

=================================================================
  ALL EXAMPLES COMPLETE — Summary of Bug Patterns
=================================================================

  #  Function                  Bug Pattern
  ─  ────────────────────────  ──────────────────────────────────────
  1  average()                 ZeroDivisionError on empty input
  2  find_max_subarray_sum()   Wrong initialisation (all-negative case)
  3  add_item()                Mutable default argument / shared state
  4  count_words()             Silent data loss with split(" ")
  5  remove_negatives()        Mutating a list while iterating
  6  read_config()             TOCTOU race + uncaught JSON/IO errors
```

---

## Bug Pattern Reference

| Pattern | Risk Level | How to Prevent |
|---|---|---|
| Empty input / division by zero | High | Guard clauses at function entry |
| Wrong initialisation | Medium | Seed accumulators with first element, not a constant |
| Mutable default argument | High | Use `None` sentinel; never use `[]` or `{}` as default |
| Silent data loss | Medium | Test with whitespace, Unicode, and boundary inputs |
| Mutate while iterating | High | Use list comprehensions or iterate a copy |
| TOCTOU race condition | High | Attempt the operation directly; handle `FileNotFoundError` |

---

## Dependencies

```
# None — pure Python 3.10+
```
