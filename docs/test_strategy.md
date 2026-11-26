# Test Strategy Document

## 1. Scope
Target: Parson JSON library (parson.c)
Baseline coverage: line 84.0%, function 85.4%, branch 72.6%

## 2. Equivalence Partitioning

### 2.1 Array APIs (owner: 廖涵玉, Issue #2)

#### json_array_replace_* (null / boolean / number / string_with_len)
Implementation check: `array == NULL || value == NULL || value->parent != NULL || ix >= count` → JSONFailure

| Input | Valid partitions | Invalid partitions |
|---|---|---|
| array | non-empty array | NULL; empty array (any ix is invalid) |
| ix | 0 ≤ ix < count | ix ≥ count |
| string (string_with_len) | valid UTF-8 | NULL; invalid UTF-8 |
| len (string_with_len) | 0; normal length; length including embedded '\0' | — |

Boundary values for ix (array with count = 3): **0, 2 (count-1)** → success; **3 (count)**, **SIZE_MAX** → failure.
Expected: on success the element is replaced and the type changes; on failure the array is unchanged.

#### json_array_append_* (null / boolean / string_with_len)
| Input | Valid | Invalid |
|---|---|---|
| array | empty array; non-empty array | NULL |
| string / len | same as above | NULL string; invalid UTF-8 |

Boundaries: append to empty array (count 0 → 1).
Expected: count increases by 1 and the new last element has the correct type and value.

#### json_array_get_* (boolean / array / object / string_len)
| Input | Partitions |
|---|---|
| array | valid; NULL |
| index | 0; count-1 (valid); count (invalid) |
| element type | matches the getter; does not match (e.g. get_boolean on a number) |

Expected on invalid input or type mismatch: get_boolean → -1, get_array / get_object → NULL, get_string_len → 0.

#### json_array_clear
Partitions: NULL array (→ JSONFailure); empty array; array with primitive values; array with nested arrays/objects.
Expected: count becomes 0, no memory leaks, and the array can still be appended to afterwards.

### Risk Assessment (Array APIs)
| Risk | APIs | Reason |
|---|---|---|
| High | replace_*, clear | Free the old values → risk of memory leaks or use-after-free |
| High | *_string_with_len | UTF-8 validation and embedded '\0' are easy to get wrong |
| Medium | append_* | Memory reallocation when the array grows |
| Low | get_* | Thin wrappers around json_array_get_value |

## 3. Boundary Value Analysis

## 4. Risk Assessment

## 5. Test Case Design

## 6. Coverage Goals
- Function >= 98%, Line >= 92%, Branch >= 85%
