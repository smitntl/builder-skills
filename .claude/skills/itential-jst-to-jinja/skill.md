---
name: itential-jst-to-jinja
description: Convert Itential JST (JSON Schema Transformation) definitions into Jinja2 templates. Use when someone wants to migrate transformations off the JST engine, reproduce JST logic in a template, or understand what a JST does before rewriting it. Trigger it for phrases like "convert this JST to Jinja", "rewrite this transformation as a Jinja template", "what does this JST do", "translate JST logic", or "I need a Jinja equivalent of this transformation".
argument-hint: "[JST name or path to .jst.json file]"
---

## Fetching a JST from the Platform

If the JST is not already available as a local file, fetch it from the platform before tracing:

```bash
curl -s "http://<platform>/transformations/?name=<jstName>&limit=200&token=<TOKEN>" \
  | jq '.results[] | select(.name == "<jstName>")'
```

Save the result to a local file and work from that. Do not load the raw JSON into the conversation — it can be hundreds of lines. Read only `incoming`, `outgoing`, `steps`, and `functions` as needed.

---

# Reading Itential JSTs (JSON Schema Transformations)

A JST is a visual, node-based data transformation stored as JSON. It has no imperative code — only a graph of method calls connected by assign wires. This guide teaches you how to read a JST JSON definition and mentally execute it.

---

## Top-Level Structure

```json
{
  "name": "myTransformation",
  "incoming": [...],   // input variables
  "outgoing": [...],   // output variables
  "steps": [...],      // all nodes and wires
  "functions": [...]   // inline lambdas (used by map/filter/etc.)
}
```

---

## Inputs and Outputs

`incoming` and `outgoing` are arrays of variable declarations:

```json
{ "$id": "payload", "type": "object" }
```

- `$id` is the variable name — used as `"name"` in assign steps with `"location": "incoming"` or `"location": "outgoing"`
- `type` is the JSON Schema type — treat it as documentation only; the JST does not enforce it at runtime

---

## Step Types

There are three step types: `method`, `assign`, and `declaration`.

### `method` — call a library function

```json
{
  "id": 6,
  "type": "method",
  "library": "Object",
  "method": "optional chaining",
  "args": [null, "orderSpecification", "ASR1K_Primary"]
}
```

- `args` holds the static arguments to the function. `null` slots are filled by `assign` steps at runtime.
- `args/0` is typically the input object (wired in by an assign).
- The result lives at `method.<id>/return`.

### `assign` — wire a value from one node to another

```json
{
  "id": 8,
  "type": "assign",
  "from": { "location": "incoming", "name": "payload", "ptr": "" },
  "to":   { "location": "method",   "name": 6,         "ptr": "/args/0/value" }
}
```

- `from` and `to` each have three parts: `location`, `name`, and `ptr`.
- `ptr` is a JSON Pointer into the node's internal structure:
  - `""` = the whole value
  - `/return` = the return value of a method node
  - `/args/0/value` = the first argument slot's value
  - `/args/2/value` = the third argument slot's value
- Assigns are the edges of the graph — they define execution order implicitly.

### `declaration` — construct a new value

```json
{
  "id": 59,
  "type": "declaration",
  "library": "Array",
  "method": "new Array",
  "args": [null, null]
}
```

- Like `method`, but used for constructors (new Array, new Object, etc.)
- Referenced by assign steps using `"location": "declaration"`.

---

## Locations Reference

| `location` | `name` field | What it points to |
|---|---|---|
| `incoming` | variable `$id` | An input variable |
| `outgoing` | variable `$id` | An output variable |
| `method` | step `id` (number) | A method step |
| `declaration` | step `id` (number) | A declaration step |

---

## How to Trace a JST

**The right approach:** follow assigns backwards from each `outgoing` variable.

1. Find the assign that writes to `"location": "outgoing", "name": "<output>"`.
2. Note what its `from` points to.
3. Follow that node's inputs by finding assigns that write to `"location": "method", "name": <id>, "ptr": "/args/0/value"`.
4. Repeat until you reach `incoming` variables or static `args` values.

**Reading static args:** When an arg slot is not wired by an assign, the value in `args[i]` is the static default. `null` means "this slot will be wired." Non-null means a hardcoded value (e.g., a key name string, an index number, a sentinel like `"no"`).

---

## Common Libraries and Methods

### Object

| Method | Args | Returns | Notes |
|---|---|---|---|
| `optional chaining` | `(obj, key1, key2, ...)` | value at path | Safe — returns `undefined` if any key is missing |
| `values` | `(obj)` | array | Same as `Object.values()` |
| `keys` | `(obj)` | array | Same as `Object.keys()` |
| `entries` | `(obj)` | array of `[key, value]` pairs | Same as `Object.entries()` |
| `fromEntries` | `(entries)` | object | Reconstruct an object from `[key, value]` pairs |
| `setProperty` | `(obj, key, value)` | new object | Returns a copy with the property set. First arg `{}` means start from empty object |
| `getProperty` | `(obj, key)` | value | Same as `obj[key]` — use when the key is a static string arg |
| `deleteProperty` | `(obj, key)` | new object | Returns a copy with the key removed |
| `hasOwnProperty` | `(obj, key)` | boolean | True if key exists directly on the object |
| `new Object` | `()` | `{}` | Used as a declaration to create a fresh empty object |
| `toString` | `(obj)` | string | `JSON.stringify`-like string representation |

### Array

| Method | Args | Returns | Notes |
|---|---|---|---|
| `new Array` | `(el0, el1, ...)` | array | Used as a declaration; each arg slot is wired separately |
| `getIndex` | `(arr, index)` | element | Zero-based |
| `setIndex` | `(arr, index, value)` | new array | Returns a copy with the element at index replaced |
| `filter` | `(arr, ƒ_query_N)` | filtered array | Second arg is a function name — look it up in `functions` |
| `map` | `(arr, ƒ_map_N)` | mapped array | Second arg is a function name |
| `find` | `(arr, ƒ_query_N)` | first matching element | Returns the first element for which the predicate returns true |
| `findIndex` | `(arr, ƒ_query_N)` | number | Index of the first matching element, or -1 |
| `some` | `(arr, ƒ_query_N)` | boolean | True if any element passes the predicate |
| `every` | `(arr, ƒ_query_N)` | boolean | True if all elements pass the predicate |
| `reduce` | `(arr, ƒ_map_N, initialValue)` | any | Accumulate a single result across all elements |
| `reduceRight` | `(arr, ƒ_map_N, initialValue)` | any | Same as `reduce` but right-to-left |
| `flat` | `(arr, depth)` | array | Flatten nested arrays to the given depth |
| `flatMap` | `(arr, ƒ_map_N)` | array | Map then flatten one level |
| `length` | `(arr)` | number | Same as `arr.length` |
| `isArray` | `(val)` | boolean | True only for arrays — use to branch on array vs object input |
| `includes` | `(arr, value)` | boolean | True if value is in the array |
| `indexOf` | `(arr, value)` | number | First index of value, or -1 |
| `lastIndexOf` | `(arr, value)` | number | Last index of value, or -1 |
| `push` | `(arr, value)` | new array | Returns a copy with value appended |
| `pop` | `(arr)` | new array | Returns a copy with the last element removed |
| `shift` | `(arr)` | new array | Returns a copy with the first element removed |
| `unshift` | `(arr, value)` | new array | Returns a copy with value prepended |
| `concat` | `(arr1, arr2)` | new array | Concatenate two arrays |
| `slice` | `(arr, start, end)` | new array | Extract a sub-array (end exclusive) |
| `splice` | `(arr, start, deleteCount, ...items)` | new array | Remove/replace/insert elements |
| `reverse` | `(arr)` | new array | Reverse the order of elements |
| `sort` | `(arr)` | new array | Sort elements — default lexicographic |
| `join` | `(arr, separator)` | string | Join elements into a string |
| `fill` | `(arr, value, start, end)` | new array | Fill a range with a static value |
| `copyWithin` | `(arr, target, start, end)` | new array | Copy a slice to another position within the array |
| `from` | `(iterable)` | array | Create an array from any iterable |
| `toString` | `(arr)` | string | Elements joined by comma |
| `toLocaleString` | `(arr)` | string | Locale-aware string representation |

### Conditional

| Method | Args | Returns | Notes |
|---|---|---|---|
| `ternary` | `(condition, truthy, falsy)` | truthy or falsy | `args[2]` is often a static sentinel like `"no"` or `0` |
| `if...else` | `(condition, truthy, falsy)` | truthy or falsy | Block-style conditional; functionally identical to `ternary` in Jinja2 |
| `switch` | `(value, case1, result1, ..., default)` | matched result | Match a value against cases; translate as a Jinja2 `if / elif / else` chain |

### Equality

| Method | Args | Returns | Notes |
|---|---|---|---|
| `equality` | `(a, b)` | boolean | `a == b` (loose equality) |
| `inequality` | `(a, b)` | boolean | `a != b` |
| `identity` | `(a, b)` | boolean | `a === b` (strict equality — same type and value) |
| `nonidentity` | `(a, b)` | boolean | `a !== b` (strict inequality) |
| `deepEquals` | `(a, b)` | boolean | Deep structural equality — commonly used to check if an object is `{}` |

### Logical

| Method | Args | Returns | Notes |
|---|---|---|---|
| `and` | `(a, b)` | boolean | `a && b` |
| `or` | `(a, b)` | boolean | `a \|\| b` |
| `not` | `(a)` | boolean | `!a` |
| `nullish` | `(a, fallback)` | value | `a ?? fallback` — returns `a` unless it is `null`/`undefined`, then returns `fallback` |

### Relational

| Method | Args | Returns | Notes |
|---|---|---|---|
| `greaterThan` | `(a, b)` | boolean | `a > b` |
| `lessThan` | `(a, b)` | boolean | `a < b` |
| `greaterThanOrEqual` | `(a, b)` | boolean | `a >= b` |
| `lessThanOrEqual` | `(a, b)` | boolean | `a <= b` |

### Math

| Method | Args | Returns | Notes |
|---|---|---|---|
| `add` | `(a, b)` | number | `a + b` |
| `subtract` | `(a, b)` | number | `a - b` |
| `multiply` | `(a, b)` | number | `a * b` |
| `divide` | `(a, b)` | number | `a / b` |
| `remainder` | `(a, b)` | number | `a % b` (modulo) |
| `exponentiation` | `(a, b)` | number | `a ** b` |
| `round` | `(n)` | integer | Rounds to nearest integer |
| `floor` | `(n)` | integer | Round down |
| `ceil` | `(n)` | integer | Round up |
| `abs` | `(n)` | number | Absolute value |
| `min` | `(a, b)` | number | Smaller of two values |
| `max` | `(a, b)` | number | Larger of two values |
| `random` | `()` | number | Random float in [0, 1) |

### String

| Method | Args | Returns | Notes |
|---|---|---|---|
| `concat` | `(str1, str2, ...)` | string | Args can be a mix of static strings and wired values |
| `split` | `(str, delimiter)` | array | Standard string split |
| `trim` | `(str)` | string | Remove leading/trailing whitespace |
| `trimStart` | `(str)` | string | Remove leading whitespace only |
| `trimEnd` | `(str)` | string | Remove trailing whitespace only |
| `replace` | `(str, search, replacement)` | string | Replace first occurrence |
| `includes` | `(str, substring)` | boolean | `str.includes(substring)` |
| `startsWith` | `(str, prefix)` | boolean | `str.startsWith(prefix)` |
| `endsWith` | `(str, suffix)` | boolean | `str.endsWith(suffix)` |
| `indexOf` | `(str, search)` | number | First index of search string, or -1 |
| `lastIndexOf` | `(str, search)` | number | Last index of search string, or -1 |
| `slice` | `(str, start, end)` | string | Extract substring (end exclusive) |
| `substring` | `(str, start, end)` | string | Like `slice` but treats negative indices as 0 |
| `toLowerCase` | `(str)` | string | All lowercase |
| `toUpperCase` | `(str)` | string | All uppercase |
| `toLocaleLowerCase` | `(str)` | string | Locale-aware lowercase |
| `toLocaleUpperCase` | `(str)` | string | Locale-aware uppercase |
| `padStart` | `(str, length, padChar)` | string | Pad left to target length |
| `padEnd` | `(str, length, padChar)` | string | Pad right to target length |
| `repeat` | `(str, count)` | string | Repeat the string N times |
| `match` | `(str, regex)` | array or null | Return all regex matches |
| `matchAll` | `(str, regex)` | iterator | Return all regex match objects |
| `search` | `(str, regex)` | number | Index of first regex match, or -1 |
| `charAt` | `(str, index)` | string | Character at index |
| `charCodeAt` | `(str, index)` | number | UTF-16 code unit at index |
| `codePointAt` | `(str, index)` | number | Unicode code point at index |
| `fromCharCode` | `(code1, ...)` | string | Create string from UTF-16 code units |
| `fromCodePoint` | `(code1, ...)` | string | Create string from Unicode code points |
| `normalize` | `(str, form)` | string | Unicode normalization (NFC, NFD, NFKC, NFKD) |
| `localeCompare` | `(str, other)` | number | -1 / 0 / 1 locale-aware comparison |
| `length` | `(str)` | number | Number of characters |
| `templateLiteral` | `(template, ...values)` | string | Interpolate values into a template string |
| `new String` | `(value)` | string | Cast value to string — common declaration for an empty string `""` |

### Number

| Method | Args | Returns | Notes |
|---|---|---|---|
| `new Number` | `(value)` | number | Cast value to number — used as a declaration to convert strings to numeric type |
| `parseInt` | `(str, radix)` | integer | Parse integer from string; `radix` is usually 10 |
| `parseFloat` | `(str)` | float | Parse float from string |
| `isNaN` | `(val)` | boolean | True if value is NaN |
| `isInteger` | `(val)` | boolean | True if value is an integer |
| `isFinite` | `(val)` | boolean | True if value is finite |
| `isSafeInteger` | `(val)` | boolean | True if value is within safe integer range |
| `toFixed` | `(n, digits)` | string | Format with fixed decimal places |
| `toPrecision` | `(n, precision)` | string | Format to significant figures |
| `toExponential` | `(n, digits)` | string | Exponential notation |
| `toLocaleString` | `(n)` | string | Locale-aware number formatting |
| `toString` | `(n, radix)` | string | Convert to string in given base (default 10) |

### Boolean

| Method | Args | Returns | Notes |
|---|---|---|---|
| `new Boolean` | `(value)` | boolean | Cast value to boolean |
| `toString` | `(bool)` | string | `"true"` or `"false"` |

### Date

| Method | Args | Returns | Notes |
|---|---|---|---|
| `new Date` | `(value)` | Date | Create a Date from a timestamp, ISO string, or component args |
| `now` | `()` | number | Current time as Unix milliseconds |
| `parse` | `(str)` | number | Parse date string to Unix milliseconds |
| `getFullYear` | `(date)` | number | 4-digit year (local time) |
| `getMonth` | `(date)` | number | Month 0–11 (local time) |
| `getDate` | `(date)` | number | Day of month 1–31 (local time) |
| `getDay` | `(date)` | number | Day of week 0–6, Sunday=0 (local time) |
| `getHours` | `(date)` | number | Hours 0–23 (local time) |
| `getMinutes` | `(date)` | number | Minutes 0–59 (local time) |
| `getSeconds` | `(date)` | number | Seconds 0–59 (local time) |
| `getMilliseconds` | `(date)` | number | Milliseconds 0–999 (local time) |
| `getTime` | `(date)` | number | Unix milliseconds |
| `getTimezoneOffset` | `(date)` | number | UTC offset in minutes |
| `getUTCFullYear` | `(date)` | number | 4-digit year (UTC) |
| `getUTCMonth` | `(date)` | number | Month 0–11 (UTC) |
| `getUTCDate` | `(date)` | number | Day of month (UTC) |
| `getUTCDay` | `(date)` | number | Day of week (UTC) |
| `getUTCHours` | `(date)` | number | Hours (UTC) |
| `getUTCMinutes` | `(date)` | number | Minutes (UTC) |
| `getUTCSeconds` | `(date)` | number | Seconds (UTC) |
| `getUTCMilliseconds` | `(date)` | number | Milliseconds (UTC) |
| `setFullYear` | `(date, year)` | number | Set year, return new timestamp |
| `setMonth` | `(date, month)` | number | Set month |
| `setDate` | `(date, day)` | number | Set day of month |
| `setHours` | `(date, hours)` | number | Set hours |
| `setMinutes` | `(date, min)` | number | Set minutes |
| `setSeconds` | `(date, sec)` | number | Set seconds |
| `setMilliseconds` | `(date, ms)` | number | Set milliseconds |
| `setTime` | `(date, ms)` | number | Set from Unix milliseconds |
| `setUTCFullYear` | `(date, year)` | number | Set year (UTC) |
| `setUTCMonth` | `(date, month)` | number | Set month (UTC) |
| `setUTCDate` | `(date, day)` | number | Set day (UTC) |
| `setUTCHours` | `(date, hours)` | number | Set hours (UTC) |
| `setUTCMinutes` | `(date, min)` | number | Set minutes (UTC) |
| `setUTCSeconds` | `(date, sec)` | number | Set seconds (UTC) |
| `setUTCMilliseconds` | `(date, ms)` | number | Set milliseconds (UTC) |
| `toISOString` | `(date)` | string | ISO 8601 string e.g. `"2026-05-11T14:30:00.000Z"` |
| `toDateString` | `(date)` | string | Human-readable date portion |
| `toTimeString` | `(date)` | string | Human-readable time portion |
| `toUTCString` | `(date)` | string | UTC string representation |
| `toLocaleDateString` | `(date)` | string | Locale date string |
| `toLocaleTimeString` | `(date)` | string | Locale time string |
| `toLocaleString` | `(date)` | string | Locale date+time string |
| `toString` | `(date)` | string | Full date string |

### JSON

| Method | Args | Returns | Notes |
|---|---|---|---|
| `parse` | `(str)` | any | Parse a JSON string into a value |
| `stringify` | `(val)` | string | Serialize a value to a JSON string |
| `type of` | `(val)` | string | Returns the JS `typeof` string: `"string"`, `"number"`, `"boolean"`, `"object"`, `"undefined"` |

### RegExp

| Method | Args | Returns | Notes |
|---|---|---|---|
| `new RegExp` | `(pattern, flags)` | RegExp | Construct a regex from a pattern string |
| `test` | `(regex, str)` | boolean | True if the regex matches the string |
| `exec` | `(regex, str)` | array or null | Return match details array or null |
| `toString` | `(regex)` | string | The regex as a string e.g. `"/pattern/flags"` |

### Set

| Method | Args | Returns | Notes |
|---|---|---|---|
| `new Set` | `(iterable)` | Set | Create a set, optionally from an iterable |
| `add` | `(set, value)` | Set | Add a value |
| `delete` | `(set, value)` | boolean | Remove a value; returns true if it existed |
| `has` | `(set, value)` | boolean | True if value is in the set |
| `clear` | `(set)` | void | Remove all values |
| `size` | `(set)` | number | Number of unique values |
| `values` | `(set)` | iterator | All values |
| `entries` | `(set)` | iterator | `[value, value]` pairs (mirrors Map interface) |

### Bitwise

| Method | Args | Returns | Notes |
|---|---|---|---|
| `AND` | `(a, b)` | number | `a & b` |
| `OR` | `(a, b)` | number | `a \| b` |
| `XOR` | `(a, b)` | number | `a ^ b` |
| `NOT` | `(a)` | number | `~a` (bitwise NOT) |
| `leftShift` | `(a, b)` | number | `a << b` |
| `rightShift` | `(a, b)` | number | `a >> b` |

### Null

| Method | Args | Returns | Notes |
|---|---|---|---|
| `new Null` | `()` | null | Produces a `null` value — used as a declaration to explicitly assign null |

### WorkFlowEngine

| Method | Args | Returns | Notes |
|---|---|---|---|
| `query` | `(taskName, args)` | any | Invoke a workflow engine task inline within the transformation |

---

## Functions (Inline Lambdas)

The `functions` array holds named lambdas referenced by map/filter steps. Each function has its own `incoming`, `outgoing`, and `steps` — read them the same way as the top-level JST.

```json
{
  "name": "ƒ_query_1",
  "incoming": [{ "$id": "element" }, ...],
  "outgoing": [{ "$id": "return", "type": "boolean" }],
  "steps": [...]
}
```

- `ƒ_query_N` names are used for `filter` and `find` predicates — they receive each element and return a boolean.
- `ƒ_map_N` names are used for `map` transforms — they receive each element and return a new value.
- Functions can have **custom names** (e.g. `rollbackmapping_filter`, `rollbackmapping_map`) instead of the default `ƒ_` prefix. Read them the same way.

**Common pattern — filter out sentinel values:**
```
// ƒ_query_1: element !== "no"
// Used after a ternary that returns "no" for disabled branches
Array.filter(arr, ƒ_query_1)
```

**`constantValue1` — extra constant passed to a map/filter function:**

When a `map` or `filter` step has a **third argument** wired in (e.g. `crudAction` wired to `args/2/value`), the function receives it as an extra input declared with `"isConstValue": true` in its `incoming` array. This is the JST way of passing a loop-external variable into a lambda:

```json
// Top-level step:
{ "method": "map", "args": [null, "ƒ_map_2", null] }
// args/2 wired from incoming "crudAction"

// Inside ƒ_map_2's incoming:
{ "$id": "constantValue1", "type": "string", "isConstValue": true }
// constantValue1 receives crudAction at runtime
```

In Jinja2 this is a non-issue — just reference the outer variable directly inside the `for` loop body.

**Dead functions — defined but never referenced:**

Some JSTs declare functions in the `functions` array that are never used by any top-level step or by another function. Check each function name against every `map`, `filter`, and `find` step before tracing it. Skip unreferenced functions entirely — do not translate them.

---

## Common Patterns

### Deep property extraction with key-agnostic access

When the intermediate key name is unknown or variable, the JST uses `Object.values()` to skip it:

```
Object.optional_chaining(payload, "orderSpec", "ASR1K_Primary")
→ Object.values(...)                  // discard the key, get array of values
→ Array.getIndex(..., 0)              // first value
→ Array.getIndex(..., 0)              // first element inside that
→ Object.optional_chaining(..., "device")  // extract specific field
```

This means the payload structure is: `{ anyKey: [ { device: "hostname" } ] }`

### Conditional array building (multi-device looping)

Build an array slot for each candidate, use a sentinel for disabled ones, then filter:

```
ternary(flagA, valueA, "no")  →  slot 0
ternary(flagB, valueB, "no")  →  slot 1
Array.new Array(slot0, slot1) →  [valueA_or_"no", valueB_or_"no"]
Array.filter(arr, ƒ_query_1)  →  only the active values
```

### Object construction with setProperty chaining

Each `setProperty` takes the previous object as its first arg, adding one key at a time:

```
setProperty({}, "key1", val1)       → { key1: val1 }
setProperty(above, "key2", val2)    → { key1: val1, key2: val2 }
```

### Nested function calls

A function body can call another function by name via `Array.find`, `Array.filter`, or `Array.map`. Trace the inner function separately, then inline its logic directly into the outer function's Jinja2 translation:

```
// ƒ_map_1 uses Array.find(lines, ƒ_query_1) internally
// ƒ_query_1: line.includes("pdp-active:")
```
```jinja2
{# Jinja2 — inline ƒ_query_1 as a Python "in" check #}
{%- for line in lines -%}
  {%- if "pdp-active:" in line and ns.found is none -%}
    {%- set ns.found = line -%}
  {%- endif -%}
{%- endfor -%}
```

### String pipeline inside a map function

When `ƒ_map_N` chains multiple string operations on a single value, write them as sequential `set` statements rather than a loop:

```jinja2
{%- set _s = raw_input | trim | replace("\\\n", "") -%}
{%- set ns = namespace(match=none) -%}
{%- for line in _s.split("\n") -%}
  {%- if "target-key:" in line and ns.match is none -%}
    {%- set ns.match = line -%}
  {%- endif -%}
{%- endfor -%}
{%- set result = ns.match.strip().split(":")[1].strip() | int -%}
```

### `Array.find` — first match only

`Array.find` returns the **first** element for which the predicate is true (not an array). Translate as a `for` loop with a `none`-guard:

```jinja2
{%- set ns = namespace(found=none) -%}
{%- for item in arr -%}
  {%- if "target" in item and ns.found is none -%}
    {%- set ns.found = item -%}
  {%- endif -%}
{%- endfor -%}
{# ns.found holds the first match #}
```

### `Logical.nullish` — null coalescing

`Logical.nullish(a, fallback)` is `a ?? fallback`: returns `a` unless it is null/undefined, then returns `fallback`. In Jinja2, combine an explicit `is not none` check with an equality guard on any sentinel value:

```jinja2
{# JST: nullish(element, "NA") → inequality(result, "NA") #}
{%- if elem is not none and elem != "NA" -%}
  {# element passed the filter #}
{%- endif -%}
```

### Math pipeline inside a map function

`ƒ_map_N` can chain Math operations. Translate as a single arithmetic expression:

```jinja2
{# JST: divide(n, 200) → round → add(1) → multiply(60) #}
{%- set result = ((n / 200) | round | int + 1) * 60 -%}
```

### `Array.isArray` branch

Used to choose between an array input and a fallback default list:

```jinja2
{%- set default_list = [{"Key": {}}, ...] -%}
{%- if vhapNF is sequence and vhapNF is not string -%}
  {%- set work_list = vhapNF -%}
{%- else -%}
  {%- set work_list = default_list -%}
{%- endif -%}
```

### Single-input array wrapping

When a JST wraps a single value in `new Array` purely to feed it into a `map` call, the array is scaffolding — not meaningful data structure. Recognise the pattern by: `new Array(singleInput)` → `map(ƒ_map_N)` → `getIndex(0)`. The Jinja2 equivalent applies the function logic directly to the value without wrapping:

```
JST:   new Array(smfRpcOutput) → map(ƒ_map_1) → getIndex(0)  →  result
Jinja: apply ƒ_map_1 logic directly to smfRpcOutput           →  result
```

No list, no loop — just inline the function body as sequential `set` statements on the value itself.

### Static string segments interleaved with wired slots

`String.concat` (and `String.templateLiteral`) can have literal string fragments already baked into `args` alongside `null` wired slots. Non-null entries in `args` are static — translate them as literal strings in the Jinja2 concatenation:

```json
// JST step: String.concat
// args: ["Unable to run create for ", null, ". Config already on device."]
//        ^--- static                   ^wired  ^--- static
// args[1] is wired from the device name variable
```

```jinja2
{%- set msg = "Unable to run create for " ~ device_name ~ ". Config already on device." -%}
```

When reading a `String.concat` step, always check every arg slot — not just the `null` (wired) ones. Static segments appear as non-null, non-empty strings and must be included in the output.

---

## Reading the `view` Field

Each step has a `view: { row, col }` that reflects its visual position on the canvas. This is useful for understanding execution groupings — steps in the same column tend to belong to the same logical pipeline. Read left to right (lower `col` = earlier in data flow), top to bottom within a column for parallel branches.

---

## Step-by-Step Reading Checklist

1. Read `incoming` — identify all inputs and their types.
2. Read `outgoing` — note every output you need to trace.
3. For each output, find the assign that writes to it (`"location": "outgoing"`).
4. Walk backwards through assigns to build the data flow chain.
5. For each method node, identify:
   - Which `args` slots are static vs. wired
   - What library/method it calls
6. If a method references a function name (e.g., `ƒ_query_1`), look it up in `functions` and trace it the same way.
7. Reconstruct the implied input payload shape from the key names used in `optional chaining` and `values` calls.
8. Verify your trace by checking that every outgoing variable is reachable from at least one incoming variable.

---

## Worked Example — End-to-End Trace

This synthetic JST covers every common pattern. Use it as a reference when converting real JSTs.

### Inputs / Outputs

```
incoming: payload (object), includePrimary (boolean), includeSecondary (boolean)
outgoing: orderId (string), primaryDevice (string), secondaryDevice (string),
          deviceSummary (object), activeDevices (array)
```

### Input Payload Shape

```json
{
  "payload": {
    "orderInfo": { "orderId": "ORD-001" },
    "primary":   { "svc_instance_1": [{ "device": "router-pri-01" }] },
    "secondary": { "svc_instance_1": [{ "device": "router-sec-01" }] }
  },
  "includePrimary": true,
  "includeSecondary": false
}
```

### Data Flow Trace

**orderId** — simple deep extraction:
```
optional_chaining(payload, "orderInfo", "orderId")  →  "ORD-001"
```

**primaryDevice / secondaryDevice** — opaque-key extraction (key name is unknown/variable):
```
optional_chaining(payload, "primary")               →  { "svc_instance_1": [{ "device": "router-pri-01" }] }
Object.values(...)                                  →  [ [{ "device": "router-pri-01" }] ]
getIndex(0)                                         →  [{ "device": "router-pri-01" }]
getIndex(0)                                         →  { "device": "router-pri-01" }
optional_chaining(..., "device")                    →  "router-pri-01"
```
Same chain applies to `secondary`.

**deviceSummary** — setProperty chaining:
```
setProperty({},    "primary",   "router-pri-01")  →  { "primary": "router-pri-01" }
setProperty(above, "secondary", "router-sec-01")  →  { "primary": "router-pri-01", "secondary": "router-sec-01" }
```

**activeDevices** — ternary sentinel + filter:
```
slot 0: ternary(includeSecondary, {"deviceName": "router-sec-01"}, "no")  →  "no"       (false)
slot 1: ternary(includePrimary,   {"deviceName": "router-pri-01"}, "no")  →  {"deviceName": "router-pri-01"}
new Array(slot0, slot1)                                                    →  ["no", {"deviceName": "router-pri-01"}]
filter(!= "no")                                                            →  [{"deviceName": "router-pri-01"}]
```

### Finished Jinja2 Template

```jinja2
{#
  servicePayloadMapper — generic worked example

  Inputs  : data["payload"] (object), data["includePrimary"] (boolean),
            data["includeSecondary"] (boolean)
  Outputs : orderId, primaryDevice, secondaryDevice, deviceSummary, activeDevices
#}

{#-- 1. Extract all JST inputs from the data dict --#}
{%- set payload          = data["payload"] -%}
{%- set includePrimary   = data["includePrimary"] -%}
{%- set includeSecondary = data["includeSecondary"] -%}

{#-- 2. orderId: simple deep extraction --#}
{%- set orderId = payload.orderInfo.orderId -%}

{#-- 3. primaryDevice / secondaryDevice: skip opaque service-instance key with .values() --#}
{%- set primaryDevice   = (payload.primary.values()   | list)[0][0].device -%}
{%- set secondaryDevice = (payload.secondary.values() | list)[0][0].device -%}

{#-- 4. deviceSummary: build object (mirrors setProperty chaining) --#}
{%- set deviceSummary = {
    "primary":   primaryDevice,
    "secondary": secondaryDevice
} -%}

{#-- 5. activeDevices: ternary sentinel pattern — secondary slot first, primary slot second --#}
{%- set ns = namespace(activeDevices=[]) -%}
{%- if includeSecondary -%}
  {%- set ns.activeDevices = ns.activeDevices + [{"deviceName": secondaryDevice}] -%}
{%- endif -%}
{%- if includePrimary -%}
  {%- set ns.activeDevices = ns.activeDevices + [{"deviceName": primaryDevice}] -%}
{%- endif -%}

{
  "orderId": {{ orderId | tojson(indent=2) }},
  "primaryDevice": {{ primaryDevice | tojson(indent=2) }},
  "secondaryDevice": {{ secondaryDevice | tojson(indent=2) }},
  "deviceSummary": {{ deviceSummary | tojson(indent=2) }},
  "activeDevices": {{ ns.activeDevices | tojson(indent=2) }}
}
```

### Expected Output (with the payload above)

```json
{
  "orderId": "ORD-001",
  "primaryDevice": "router-pri-01",
  "secondaryDevice": "router-sec-01",
  "deviceSummary": { "primary": "router-pri-01", "secondary": "router-sec-01" },
  "activeDevices": [{ "deviceName": "router-pri-01" }]
}
```

---

# Converting JST to Jinja2

Once you have traced the data flow (see above), write the Jinja2 template using the rules and patterns in this section.

---

## General Approach

1. Trace every outgoing variable back to its incoming sources (follow the reading checklist above).
2. Write one `{%- set -%}` variable per logical intermediate result, in dependency order.
3. Render all outputs at the bottom of the template as a JSON object using `| tojson(indent=2)`.

---

## Accessing JST Inputs — the `data` Dict

The platform passes **all JST inputs** into the Jinja2 template as a single dict named `data`. Every input, regardless of whether its `$id` contains hyphens or not, must be extracted from `data` at the top of the template.

```jinja2
{%- set device_name                 = data["device-name"] -%}
{%- set device_list                 = data["device-list"] -%}
{%- set mergedDeviceObjAvailability = data["mergedDeviceObjAvailability"] -%}
```

- Use bracket notation `data["key"]` for all inputs — especially required for hyphenated names like `device-name`, but apply it consistently to all inputs for clarity.
- After extraction, use the local variable names (with underscores) throughout the rest of the template.
- **Object field keys** inside the extracted values still use their original names and require bracket notation: `d["device-name"]`, `d["device-role"]`, etc.

Do NOT attempt to access JST inputs as bare Jinja2 variables (e.g. `device_name` without extracting from `data` first) — the platform will silently return undefined/empty rather than raising an error.

---

## Outgoing Variable Names with Hyphens

JST outgoing `$id` values can also contain hyphens (e.g. `context-id`). Jinja2 variable names cannot contain hyphens, so use an underscore-named local variable internally and render the hyphenated key only in the final JSON output block:

```jinja2
{#-- local variable uses underscore #}
{%- set context_id = ... -%}

{#-- output key preserves the original hyphenated name #}
{
  "context-id": {{ context_id | tojson(indent=2) }}
}
```

The same rule applies to any outgoing `$id` with a hyphen — the hyphen is valid in a JSON string key, just not in a Jinja2 identifier.

---

## JST → Jinja2 Mapping

### Object

| JST construct | Jinja2 equivalent |
|---|---|
| `Object.optional chaining(obj, "a", "b")` | `obj.a.b` |
| `Object.values(obj)` | `obj.values() \| list` |
| `Object.keys(obj)` | `obj.keys() \| list` |
| `Object.entries(obj)` | `obj.items() \| list` |
| `Object.fromEntries(pairs)` | `pairs \| items2dict` |
| `Object.setProperty({}, "k", v)` | `{"k": v}` (build literal) |
| `Object.setProperty(prev, "k", v)` | `prev \| combine({"k": v})` or full dict literal when all keys are known |
| `Object.getProperty(obj, "k")` | `obj["k"]` or `obj.k` |
| `Object.deleteProperty(obj, "k")` | `obj \| dict2items \| rejectattr("key","equalto","k") \| items2dict` |
| `Object.hasOwnProperty(obj, "k")` | `"k" in obj` |
| `Object.new Object()` | `{}` |
| `Object.toString(obj)` | `obj \| tojson` |

### Array

| JST construct | Jinja2 equivalent |
|---|---|
| `Array.new Array(s0, s1, ...)` | `[s0, s1, ...]` |
| `Array.getIndex(arr, i)` | `arr[i]` |
| `Array.setIndex(arr, i, v)` | `arr[:i] + [v] + arr[i+1:]` |
| `Array.filter(arr, ƒ_query_N)` | `arr \| selectattr(...)` / `rejectattr(...)` or a `for` loop with `if` |
| `Array.map(arr, ƒ_map_N)` | `arr \| map(...)` or a `for` loop |
| `Array.find(arr, ƒ_query_N)` | `for` loop with a `none`-guard namespace variable (see Patterns) |
| `Array.findIndex(arr, ƒ_query_N)` | `for` loop tracking index with a namespace variable |
| `Array.some(arr, ƒ_query_N)` | `arr \| selectattr(...) \| list \| length > 0` |
| `Array.every(arr, ƒ_query_N)` | `arr \| rejectattr(...) \| list \| length == 0` |
| `Array.reduce(arr, ƒ_map_N, init)` | `for` loop accumulating into a namespace variable |
| `Array.flat(arr, depth)` | not a built-in filter; use a recursive `for` loop or Jinja2 `\| flatten` if available |
| `Array.flatMap(arr, ƒ_map_N)` | map `for` loop, then `\| flatten` or manual concatenation |
| `Array.length(arr)` | `arr \| length` |
| `Array.isArray(val)` | `val is sequence and val is not string` |
| `Array.includes(arr, val)` | `val in arr` |
| `Array.indexOf(arr, val)` | not a built-in; use a `for` loop with index tracking |
| `Array.push(arr, val)` | `arr + [val]` |
| `Array.pop(arr)` | `arr[:-1]` |
| `Array.shift(arr)` | `arr[1:]` |
| `Array.unshift(arr, val)` | `[val] + arr` |
| `Array.concat(arr1, arr2)` | `arr1 + arr2` |
| `Array.slice(arr, start, end)` | `arr[start:end]` |
| `Array.reverse(arr)` | `arr \| reverse \| list` |
| `Array.sort(arr)` | `arr \| sort` |
| `Array.join(arr, sep)` | `arr \| join(sep)` |
| `Array.from(iterable)` | `iterable \| list` |
| `Array.toString(arr)` | `arr \| join(",")` |

### Conditional

| JST construct | Jinja2 equivalent |
|---|---|
| `Conditional.ternary(cond, a, b)` | `a if cond else b` |
| `Conditional.if...else(cond, a, b)` | `{%- if cond -%}...{%- else -%}...{%- endif -%}` |
| `Conditional.switch(val, c1, r1, ..., default)` | `{%- if val == c1 -%}r1{%- elif val == c2 -%}r2{%- else -%}default{%- endif -%}` |

### Equality

| JST construct | Jinja2 equivalent |
|---|---|
| `Equality.equality(a, b)` | `a == b` |
| `Equality.inequality(a, b)` | `a != b` |
| `Equality.identity(a, b)` | `a == b` (Jinja2 has no strict-type `===`; use `== b and a is typeof b` if type matters) |
| `Equality.nonidentity(a, b)` | `a != b` |
| `Equality.deepEquals(a, b)` | `a == b` |

### Logical

| JST construct | Jinja2 equivalent |
|---|---|
| `Logical.and(a, b)` | `a and b` |
| `Logical.or(a, b)` | `a or b` |
| `Logical.not(a)` | `not a` |
| `Logical.nullish(a, fallback)` | `a if a is not none else fallback` |

### Relational

| JST construct | Jinja2 equivalent |
|---|---|
| `Relational.greaterThan(a, b)` | `a > b` |
| `Relational.lessThan(a, b)` | `a < b` |
| `Relational.greaterThanOrEqual(a, b)` | `a >= b` |
| `Relational.lessThanOrEqual(a, b)` | `a <= b` |

### Math

| JST construct | Jinja2 equivalent |
|---|---|
| `Math.add(a, b)` | `a + b` |
| `Math.subtract(a, b)` | `a - b` |
| `Math.multiply(a, b)` | `a * b` |
| `Math.divide(a, b)` | `a / b` |
| `Math.remainder(a, b)` | `a % b` |
| `Math.exponentiation(a, b)` | `a ** b` |
| `Math.round(n)` | `n \| round \| int` |
| `Math.floor(n)` | `n \| int` (for positive numbers) or `(n // 1) \| int` |
| `Math.ceil(n)` | `(-(-n // 1)) \| int` or use `(n \| round(0, 'ceil')) \| int` |
| `Math.abs(n)` | `n \| abs` |
| `Math.min(a, b)` | `[a, b] \| min` |
| `Math.max(a, b)` | `[a, b] \| max` |
| `Math.random()` | not available in Jinja2 without a custom filter |

### String

| JST construct | Jinja2 equivalent |
|---|---|
| `String.concat(a, b, ...)` | `a ~ b ~ ...` |
| `String.split(s, delim)` | `s.split(delim)` |
| `String.trim(s)` | `s \| trim` or `s.strip()` |
| `String.trimStart(s)` | `s.lstrip()` |
| `String.trimEnd(s)` | `s.rstrip()` |
| `String.replace(s, search, repl)` | `s \| replace(search, repl)` |
| `String.includes(s, sub)` | `sub in s` |
| `String.startsWith(s, prefix)` | `s.startswith(prefix)` |
| `String.endsWith(s, suffix)` | `s.endswith(suffix)` |
| `String.indexOf(s, sub)` | `s.find(sub)` |
| `String.lastIndexOf(s, sub)` | `s.rfind(sub)` |
| `String.slice(s, start, end)` | `s[start:end]` |
| `String.substring(s, start, end)` | `s[start:end]` |
| `String.toLowerCase(s)` | `s \| lower` |
| `String.toUpperCase(s)` | `s \| upper` |
| `String.padStart(s, len, char)` | `s.rjust(len, char)` |
| `String.padEnd(s, len, char)` | `s.ljust(len, char)` |
| `String.repeat(s, n)` | `s * n` |
| `String.charAt(s, i)` | `s[i]` |
| `String.length(s)` | `s \| length` |
| `String.match(s, regex)` | use `String.match` filter if available; otherwise `re` module not available in standard Jinja2 |
| `String.search(s, regex)` | same caveat as `match` |
| `String.templateLiteral(tpl, ...vals)` | native Jinja2 `{{ }}` interpolation or `tpl \| format(*vals)` |
| `String.new String(val)` | `val \| string` |

### Number

| JST construct | Jinja2 equivalent |
|---|---|
| `Number.new Number(val)` | `val \| int` or `val \| float` |
| `Number.parseInt(s, radix)` | `s \| int(base=radix)` (radix supported) |
| `Number.parseFloat(s)` | `s \| float` |
| `Number.isNaN(val)` | `val != val` (NaN is the only value not equal to itself) |
| `Number.isInteger(val)` | `val == val \| int` |
| `Number.toFixed(n, digits)` | `"%.{digits}f" \| format(n)` |
| `Number.toString(n, radix)` | for base 10: `n \| string`; other bases not natively available |

### Boolean

| JST construct | Jinja2 equivalent |
|---|---|
| `Boolean.new Boolean(val)` | `val \| bool` or `val is truthy` |
| `Boolean.toString(bool)` | `"true" if bool else "false"` |

### Date

| JST construct | Jinja2 equivalent |
|---|---|
| `Date.now()` | not natively available; requires a custom filter or `strftime` if provided |
| `Date.toISOString(date)` | `date \| strftime("%Y-%m-%dT%H:%M:%S.000Z")` if the platform provides `strftime` |
| `Date.new Date(str)` | platform-specific; treat as opaque and pass through unless date math is needed |

### JSON

| JST construct | Jinja2 equivalent |
|---|---|
| `JSON.parse(str)` | `str \| from_json` (if available) or `str` (if already a dict in context) |
| `JSON.stringify(val)` | `val \| tojson` |
| `JSON.type of(val)` | `val \| type_name` (custom filter); use `is string`, `is number`, `is mapping`, `is sequence` tests instead |

### RegExp

| JST construct | Jinja2 equivalent |
|---|---|
| `RegExp.test(regex, str)` | `str \| regex_search(pattern)` if the platform provides it; otherwise not available in standard Jinja2 |
| `RegExp.exec(regex, str)` | same caveat as `test` |

### Set

| JST construct | Jinja2 equivalent |
|---|---|
| `Set.new Set(iterable)` | `iterable \| unique \| list` (deduplicate) |
| `Set.has(set, val)` | `val in set` |
| `Set.add(set, val)` | `(set + [val]) \| unique \| list` |
| `Set.values(set)` | `set \| list` |
| `Set.size(set)` | `set \| length` |

### Bitwise

| JST construct | Jinja2 equivalent |
|---|---|
| `Bitwise.AND(a, b)` | `a \| bitwise_and(b)` (custom filter) or not available natively |
| `Bitwise.OR(a, b)` | `a \| bitwise_or(b)` (custom filter) |
| `Bitwise.XOR(a, b)` | `a \| bitwise_xor(b)` (custom filter) |
| `Bitwise.NOT(a)` | not natively available in Jinja2 |
| `Bitwise.leftShift(a, b)` | not natively available in Jinja2 |
| `Bitwise.rightShift(a, b)` | not natively available in Jinja2 |

### WorkFlowEngine

| JST construct | Jinja2 equivalent |
|---|---|
| `WorkFlowEngine.query(task, args)` | No equivalent — this invokes a live workflow task at runtime. Cannot be replicated in a static Jinja2 template; treat the expected return value as a pre-supplied input variable instead. |

---

## Jinja2 Scoping Rule — Use `namespace()` for Mutable Lists

Jinja2 does not allow a variable set inside an `if` or `for` block to update a variable in the outer scope. Use `namespace()` whenever you need to build a list or accumulate state conditionally:

```jinja2
{%- set ns = namespace(items=[]) -%}
{%- if conditionA -%}
  {%- set ns.items = ns.items + [valueA] -%}
{%- endif -%}
{%- if conditionB -%}
  {%- set ns.items = ns.items + [valueB] -%}
{%- endif -%}
```

This is the standard replacement for the JST `ternary → new Array → filter` sentinel pattern.

---

## Output Guidelines

- **Always use `| tojson(indent=2)`** for every value rendered in the output JSON block. Never use plain `| tojson`.
- Use `{%- ... -%}` (strip-whitespace) on all setup lines so only the final JSON block produces output.
- Preserve the JST outgoing variable names exactly as keys in the output object.
- The order of keys in the output object should match the order of `outgoing` declarations in the JST.

```jinja2
{
  "outVar1": {{ outVar1 | tojson(indent=2) }},
  "outVar2": {{ outVar2 | tojson(indent=2) }}
}
```

---

## Common Pattern Translations

### Deep extraction through an opaque key

JST uses `Object.values()` when the intermediate key name is arbitrary (e.g. a service-instance ID). Jinja2:

```jinja2
{#  payload.someSection = { "anyKey": [{ "device": "hostname" }] }  #}
{%- set device = (payload.someSection.values() | list)[0][0].device -%}
```

### Conditional array (multi-device looping)

JST: `ternary(flag, value, "no") × N → new Array → filter(!= "no")`

Jinja2:
```jinja2
{%- set ns = namespace(devices=[]) -%}
{%- if flagSec -%}
  {%- set ns.devices = ns.devices + [{"deviceName": sec_device}] -%}
{%- endif -%}
{%- if flagPri -%}
  {%- set ns.devices = ns.devices + [{"deviceName": pri_device}] -%}
{%- endif -%}
```

Preserve the JST slot order: the method node at index 0 in `new Array` corresponds to the first `if` block.

### Object construction with `setProperty` chaining

When all keys are known, build the dict literal directly:

```jinja2
{%- set deviceObj = {
    "ASR1K_Primary":   pri_device,
    "ASR1K_Secondary": sec_device
} -%}
```

When keys must be added dynamically, chain with `| combine`:

```jinja2
{%- set obj = {} | combine({"key1": val1}) | combine({"key2": val2}) -%}
```

---

## Template Structure

Follow this layout for every converted template:

```jinja2
{#
  <JST name> — Jinja2 equivalent of <filename>.jst.json

  Inputs  : <list incoming variables and types>
  Outputs : <list outgoing variables>
#}

{#-- 1. Extract all JST inputs from the data dict --#}
{%- set inputA = data["inputA"] -%}
{%- set inputB = data["inputB"] -%}

{#-- 2. Extract top-level sections / intermediate values --#}
{%- set ... -%}

{#-- 3. Build output objects --#}
{%- set ... -%}

{#-- 4. Build conditional arrays or accumulate state using namespace() --#}
{%- set ns = namespace(...) -%}
{%- if ... -%} ... {%- endif -%}

{#-- 5. Render output --#}
{
  "outVar1": {{ outVar1 | tojson(indent=2) }},
  "outVar2": {{ outVar2 | tojson(indent=2) }}
}
```
