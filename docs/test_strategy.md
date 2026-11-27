# Test Strategy Document

## 1. Scope

| Item | Description |
|---|---|
| Target | Parson JSON library, `parson.c` (https://github.com/kgabis/parson) |
| Baseline | commit `ec53fb6` (git tag `baseline`) — line 84.0%, function 85.4%, branch 72.6% |
| Coverage tools | gcc `--coverage`, gcov, LCOV / genhtml (only `parson.c` is measured, not `tests.c`) |
| Techniques | Equivalence Partitioning (EP), Boundary Value Analysis (BVA), risk-based prioritization |
| Test oracle | Return status (`JSONSuccess` / `JSONFailure`), resulting values and types, exact serialized output, round trip `parse(serialize(x)) == x`, and no memory leaks (`g_malloc_count == 0`) |
| Out of scope | Performance, thread safety, platform-specific behavior |

## 2. Test Strategy

### 2.1 Array APIs (owner: 廖涵玉, Issue #2)

Target functions (12, all uncovered in the baseline):
`json_array_append_null / boolean / string_with_len`, `json_array_replace_null / boolean / number / string_with_len`,
`json_array_get_boolean / array / object / string_len`, `json_array_clear`

#### Equivalence Partitioning

**json_array_replace_\*** — implementation check: `array == NULL || value == NULL || value->parent != NULL || ix >= count` → `JSONFailure`

| Input | Valid partitions | Invalid partitions |
|---|---|---|
| array | non-empty array | NULL; empty array (any ix is invalid) |
| ix | 0 ≤ ix < count | ix ≥ count |
| string (string_with_len) | valid UTF-8 | NULL; invalid UTF-8 |
| len (string_with_len) | 0; normal length; length including embedded `'\0'` | — |

**json_array_append_\***

| Input | Valid partitions | Invalid partitions |
|---|---|---|
| array | empty array; non-empty array | NULL |
| string / len | same as replace | NULL string; invalid UTF-8 |

**json_array_get_\***

| Input | Partitions |
|---|---|
| array | valid; NULL |
| index | valid (0 ≤ index < count); invalid (index ≥ count) |
| element type | matches the getter; does not match (e.g. `get_boolean` on a number) |

**json_array_clear**: NULL array; empty array; array of primitive values; array containing nested arrays/objects.

#### Boundary Value Analysis

| Variable | Boundary values | Expected |
|---|---|---|
| ix / index (array with count = 3) | 0, 2 (count-1) | success |
| ix / index | 3 (count), `SIZE_MAX` | failure / NULL / -1 / 0 |
| array size | empty array (count = 0) | append: count 0 → 1; replace: failure; clear: success |
| string length | 0; length including embedded `'\0'` | stored with the exact length |

#### Risk Assessment

| Risk | APIs | Reason |
|---|---|---|
| High | replace_\*, clear | Free the old values → risk of memory leaks or use-after-free |
| High | \*_string_with_len | UTF-8 validation and embedded `'\0'` are easy to get wrong |
| Medium | append_\* | Memory reallocation when the array grows |
| Low | get_\* | Thin wrappers around `json_array_get_value` |

#### Test Cases (`test_array_apis()` in tests.c)

| ID | API | Input | Expected | Technique |
|---|---|---|---|---|
| A-01 | append_\* | NULL array | JSONFailure | EP (invalid) |
| A-02 | append_null | empty array | count 0 → 1, element is JSONNull | BVA |
| A-03 | append_boolean | true, false | get_boolean returns 1, 0 | EP (valid) |
| A-04 | append_string_with_len | len 0; first 5 chars of a longer string; `"a\0b"` with len 3 | exact length and content | EP + BVA |
| A-05 | append_string_with_len | NULL string; invalid UTF-8 `\xff\xfe` | JSONFailure, count unchanged | EP (invalid) |
| A-06 | replace_\* | ix = 0, ix = count-1 | JSONSuccess, element type changes | BVA |
| A-07 | replace_\* | ix = count, ix = `SIZE_MAX` | JSONFailure, count unchanged | BVA |
| A-08 | replace_\* | NULL array; empty array | JSONFailure | EP (invalid) |
| A-09 | replace_string_with_len | NULL string; invalid UTF-8 | JSONFailure, old element unchanged | EP (invalid) |
| A-10 | get_\* | matching element type | correct value | EP (valid) |
| A-11 | get_\* | mismatched element type | -1 / NULL / 0 | EP (invalid) |
| A-12 | get_\* | index = count; NULL array | -1 / NULL / 0 | BVA + EP |
| A-13 | clear | NULL array | JSONFailure | EP (invalid) |
| A-14 | clear | empty array | JSONSuccess, count 0 | BVA |
| A-15 | clear | array with nested array/object | count 0, nested values freed, array usable afterwards | EP (valid) |
| A-16 | all of the above | — | no memory leaks (`g_malloc_count == 0`) | Oracle |

### 2.2 Object / Value APIs (owner: 林宜頡, Issue #3)

Target functions (9, all uncovered in the baseline):
`json_object_dotset_null / boolean / string_with_len`, `json_object_dotget_object / string_len`,
`json_boolean`, `json_number`, `json_string_len`, `json_type`

#### Equivalence Partitioning
TODO

#### Boundary Value Analysis
TODO

#### Risk Assessment
TODO

#### Test Cases
TODO

### 2.3 String Serialization (owner: 廖涵玉, Issue #4)

Target: `json_serialize_string` (static), tested through `json_serialize_to_string`,
`json_serialize_to_string_pretty` and `json_serialize_to_buffer`.

#### Equivalence Partitioning

| Partition | Representative values | Expected |
|---|---|---|
| Characters with short escapes | `"` `\` `/` `\b` `\f` `\n` `\r` `\t` | `\"` `\\` `\/` `\b` ... |
| Other control characters | 0x00–0x07, 0x0b, 0x0e–0x1f | `\u00XX` (lowercase hex) |
| Normal characters | printable ASCII, UTF-8 | copied unchanged |
| Invalid: buffer too small | buffer size < serialization size | JSONFailure |

#### Boundary Value Analysis

| Variable | Boundary values | Expected |
|---|---|---|
| character code | 0x1f (last escaped) / 0x20 (first unescaped) | `\u001f` / space |
| character code | 0x7f (DEL) | not escaped (JSON only requires escaping 0x00–0x1f) |
| string length | 0; 32 (all control characters) | exact output |

#### Risk Assessment

**High** — 26 escape branches (0x01–0x07, 0x0b, 0x0e–0x1f) were never executed in the baseline (found via gcov).
An incorrect escape produces invalid JSON that other parsers would reject.

#### Test Cases (`test_serialize_string_escapes()` in tests.c)

| ID | Input | Expected | Technique |
|---|---|---|---|
| S-01 | string of all 32 characters 0x00–0x1f | exact escaped output; serialization size matches; round trip equals the original | EP |
| S-02 | `"\x1f\x20"` | `"\u001f "` | BVA |
| S-03 | `"a\x7fb"` | DEL copied unchanged | BVA |
| S-04 | object `{"k\x01": "v\x1e"}`, pretty serialization | key and value both escaped | EP |
| S-05 | serialize `"\x01\x02"` into a 4-byte buffer | JSONFailure | EP (invalid) |
| S-06 | all of the above | no memory leaks | Oracle |

### 2.4 Deep Copy (owner: 林宜頡, Issue #5)

Target: `json_value_deep_copy`

#### Equivalence Partitioning
TODO

#### Boundary Value Analysis
TODO

#### Risk Assessment
TODO

#### Test Cases
TODO

## 3. Coverage Goals

| Metric | Baseline | Goal |
|---|---|---|
| Function | 85.4% | ≥ 98% |
| Line | 84.0% | ≥ 92% |
| Branch | 72.6% | ≥ 85% |

Coverage is only a supporting metric: every test must also check behavior with assertions
(return values, resulting state, exact output), not just execute code.