- Start Date: 2026-05-11
- PartiQL Issue: [partiql/partiql-lang#8](https://github.com/partiql/partiql-lang/issues/8)
- RFC PR: https://github.com/partiql/partiql-lang/pull/104

# Summary

This RFC proposes the addition of a first-class `MAP<K, V>` type to PartiQL. A MAP is an unordered collection of key-value pairs where every key has the same type K, every value has the same type V, keys are unique, and access is by key expression. This fills the gap between ROW (fixed schema with known fields) and STRUCT (schemaless with string keys), providing a typed dictionary that enables static type reasoning by the planner.

# Motivation

PartiQL currently has two container types for key-value data: ROW and STRUCT. Neither adequately models the "typed dictionary" pattern common in real-world data:

- **ROW** models fixed schemas (SQL tuples) — the planner knows exactly which fields exist and their types at compile time, but fields are string-only and the schema is closed.
- **STRUCT** models schemaless/semi-structured data (Ion structs) — keys are strings, values can be anything, duplicates are allowed, and the planner cannot reason about element types.

Many systems (Hive, Iceberg, Trino, Spark SQL, Presto) support a MAP type for use cases such as:

- Configuration maps (e.g., `MAP<STRING, STRING>` of settings)
- Lookup tables (e.g., `MAP<INT, STRING>` of ID-to-name mappings)
- JSON objects with dynamic keys (e.g., metrics keyed by timestamp)
- Ion structs used as dictionaries where keys are known to be homogeneous

Adding MAP as a first-class type allows PartiQL to:

1. Close the gap between PartiQL and other major SQL systems.
2. Enable type-safe operations — the planner can infer key and value types statically.
3. Enforce key uniqueness as a type-level invariant.
4. Support non-string key types (integers, dates, timestamps, etc.).

# Syntax

## Type Declaration

A MAP type is declared using angle-bracket parameterization, consistent with the existing `ARRAY<T>` syntax:

```sql
MAP<K, V>
```

Where `K` is the key type and `V` is the value type. Examples:

```sql
MAP<STRING, INT>
MAP<INT, STRUCT>
MAP<DATE, ARRAY<STRING>>
MAP<STRING, MAP<STRING, INT>>
```

## Construction

A MAP literal is constructed using the `MAP` keyword followed by curly braces containing colon-separated key-value pairs (Option A):

```sql
MAP { } -- empty map
MAP { 'a': 1, 'b': 2, 'c': 3 }
MAP { 1: 'one', 2: 'two', 3: 'three' }
MAP { DATE '2026-01-01': 100, DATE '2026-01-02': 200 }
```

The `MAP` keyword prefix disambiguates from STRUCT literal syntax (`{ ... }`).

**Alternatives considered:**

- **Option B** — `MAP(k1, v1, k2, v2)`: Function-style with positional arguments. Concise for few entries but error-prone with many entries since keys and values alternate without visual grouping.
- **Option C** — `MAP([k1, k2], [v1, v2])`: Function-style taking a key array and a value array separately. Ensures equal counts but disconnects each key from its value, making it harder to read and maintain.

## Access / Lookup

Map entries are accessed using bracket notation with a key expression, reusing the existing path index syntax:

```sql
-- Given: my_map is MAP<STRING, INT> with entries 'x' -> 10, 'y' -> 20
my_map['x']          -- returns 10
my_map['z']          -- MISSING (permissive mode) or error (strict mode)
```

```sql
-- Given: id_map is MAP<INT, STRING>
id_map[42]           -- returns the value associated with key 42
```

A function alias `MAP_GET(map, key)` is also provided for clarity in complex expressions:

```sql
MAP_GET(my_map, 'x')  -- equivalent to my_map['x']
```

### When a key is not found:

- **Permissive mode**: returns `MISSING`
- **Strict mode**: raises an error (consistent with struct field access behavior)

### Key Type Matching in Lookup

The key expression type is compared against the declared key type of the MAP. For compatible types, an implicit cast to the target key type is applied (e.g., numeric widening). For incompatible types, a type mismatch error is raised and an explicit cast is required.

```sql
-- Given: id_map is MAP<INT, STRING>
id_map[42]                 -- OK: key is INT (exact match)
id_map[42.0]               -- OK: implicit cast DECIMAL → INT applied (compatible numeric type)
id_map['42']               -- Error: key type mismatch (STRING vs INT, incompatible)
id_map[CAST('42' AS INT)]  -- OK: explicit cast to INT
```

## Grammar

```ebnf
<map type> ::= MAP '<' <data type> ',' <data type> '>'

<map constructor> ::= MAP '{' [ <map entry> ( ',' <map entry> )* ] '}'

<map entry> ::= <expr> ':' <expr>

<map access> ::= <expr> '[' <expr> ']'
```

# Constraints

## Immutability

MAP values in PartiQL are **immutable**. Once constructed, a MAP cannot be modified in place. All operations that appear to modify a MAP (e.g., `MAP_PUT`, `MAP_REMOVE`, `||`) return a **new MAP value** — the original is unchanged. This is consistent with how all PartiQL values (scalars, arrays, structs) behave as immutable data within query evaluation.

## Key Constraints

Keys must support **equality comparison** for lookup and **uniqueness enforcement**. The key type K must be a **comparable type** — a type that has well-defined equality semantics and is hashable. The exact definition of equality is implementation-dependent (e.g., whether string comparison is case-sensitive or uses a specific collation).

**NULL as key:** Disallowed. Since `NULL = NULL` evaluates to `NULL` (not `TRUE`), a NULL key cannot be reliably looked up or deduplicated. Attempting to insert a NULL key raises an error.

**MISSING as key:** Disallowed. MISSING represents the absence of a value and cannot serve as a key. Attempting to insert a MISSING key raises an error. Accessing a map with a MISSING key expression resolves to `MISSING` (missing propagation).

**Duplicate key policy:** Keys must be unique within a MAP. When a MAP is constructed with duplicate keys:

- **Strict mode**: raises an error
- **Permissive mode**: last-write-wins (the last value for a duplicate key is retained)

```sql
-- Strict mode:
MAP { 'a': 1, 'a': 2 }  -- Error: duplicate key 'a'

-- Permissive mode:
MAP { 'a': 1, 'a': 2 }  -- Results in MAP { 'a': 2 }
```

**Other consideration:** The duplicate key policy may be made configurable or left as implementation-defined behavior. This allows implementations to choose the policy that best fits their use case (e.g., an ingestion engine may prefer last-write-wins for performance, while a validation-oriented system may prefer strict error-on-duplicate).

## Value Constraints

The value type V can be **any** PartiQL type, including:

- Scalars (`INT`, `STRING`, `BOOL`, ...)
- Collections (`ARRAY`, `BAG`)
- Containers (`STRUCT`, `ROW`, `MAP`)
- `DYNAMIC` (heterogeneous values)

NULL as value is allowed — e.g., `MAP { 'a': NULL, 'b': 42 }` is valid.

MISSING as value is allowed — e.g., `MAP { 'a': MISSING, 'b': 42 }` is valid. Accessing a key whose value is MISSING returns `MISSING`. Note: this is semantically distinct from the key not existing — `CONTAINS_KEY(m, 'a')` returns `TRUE` even when the value is MISSING.

# Map Operations
## Type Check

```sql
my_map IS MAP         -- TRUE if my_map is a MAP value
NULL IS MAP           -- NULL
MISSING IS MAP        -- MISSING
```

## Measurement

```sql
SIZE(my_map)          -- returns the number of entries
CARDINALITY(my_map)   -- same as SIZE
EXISTS(my_map)        -- TRUE if the map has at least one entry
```

## Decomposition with UNPIVOT

`UNPIVOT` decomposes a MAP into a bag of key-value rows:

```sql
SELECT k, v
FROM UNPIVOT my_map AS v AT k
-- Produces: BAG<ROW(k: K, v: V)>
```

For MAP, `k` is the key (of type K, not necessarily string) and `v` is the value (of type V).

## MAP_KEYS, MAP_VALUES, and Related Functions

```sql
MAP_KEYS(my_map)                -- returns BAG<K> of all keys
MAP_VALUES(my_map)              -- returns BAG<V> of all values
CONTAINS_KEY(my_map, key)       -- returns TRUE if key exists in the map
MAP_ENTRIES(my_map)             -- returns an array of rows (key, value pairs)
```

## Iteration in MAP

A MAP is not directly iterable in the FROM clause. Use `UNPIVOT` to decompose it into key-value pairs:

```sql
SELECT k, v
FROM UNPIVOT my_map AS v AT k
```

## Modification Functions

```sql
MAP_PUT(my_map, key, value)     -- returns a new map with the entry added/updated
MAP_REMOVE(my_map, key)         -- returns a new map with the entry removed

```

## Merge

Maps can be merged using the concat `||` operator:

```sql
map1 || map2          -- entries from map2 override map1 on key conflict (last-write-wins)
```

## Casting

```sql
CAST(my_struct AS MAP<STRING, DYNAMIC>)  -- converts STRUCT to MAP
CAST(my_map AS MAP<STRING, INT>)         -- re-parameterizes a MAP (with coercion)
```

No implicit STRUCT-to-MAP coercion is defined; conversion requires explicit `CAST`.

## EXCLUDE

`EXCLUDE` on MAP entries works by key expression:

```sql
SELECT * EXCLUDE my_map['unwanted_key'] FROM ...
```

## Null/Missing Semantics

| Expression | Result |
|---|---|
| `NULL IS MAP` | `NULL` |
| `MISSING IS MAP` | `MISSING` |
| `MAP_GET(NULL, k)` | `NULL` (null propagation) |
| `MAP_GET(m, NULL)` | `NULL` (null key cannot match any entry) |
| `MAP_GET(m, MISSING)` | `MISSING` (missing propagation) |
| `MAP_GET(m, absent_key)` | `MISSING` (permissive) or error (strict) |

### Sample Data

The following examples use this common dataset — a `students` table where each row has a `name`, a `scores` map (subject to score), and a `tags` map (tag name to tag value):

``` partiql
{
  'students': <<
    {
      'name': 'Alice',
      'scores': MAP { 'math': 90, 'science': 85, 'english': 72 },
      'tags': MAP { 'status': 'active', 'priority': 'high' }
    },
    {
      'name': 'Bob',
      'scores': MAP { 'math': 75, 'science': 92 },
      'tags': MAP { 'status': 'active' }
    },
    {
      'name': 'Carol',
      'scores': MAP { 'math': 88, 'science': 79, 'english': 95 },
      'tags': MAP { 'status': 'inactive', 'priority': 'low' }
    }
  >>
}
```

## Filtering by Key/Value

MAP entries can be filtered directly using key lookup or by decomposing with UNPIVOT:

```sql
-- Filter rows where a specific key's value meets a condition.
-- Uses bracket notation to access the 'status' key in the tags MAP,
-- then compares the value to filter rows.
SELECT *
FROM students
WHERE tags['status'] = 'active'
```

Result:
```
<<
  {
    'name': 'Alice',
    'scores': MAP { 'math': 90, 'science': 85, 'english': 72 },
    'tags': MAP { 'status': 'active', 'priority': 'high' }
  },
  {
    'name': 'Bob',
    'scores': MAP { 'math': 75, 'science': 92 },
    'tags': MAP { 'status': 'active' }
  }
>>
```

```sql
-- Filter rows where a specific key exists.
-- CONTAINS_KEY checks for the presence of a key without accessing its value.
-- Useful when the key may or may not exist in a given MAP.
SELECT *
FROM students
WHERE CONTAINS_KEY(tags, 'priority')
```

Result:
```
<<
  {
    'name': 'Alice',
    'scores': MAP { 'math': 90, 'science': 85, 'english': 72 },
    'tags': MAP { 'status': 'active', 'priority': 'high' }
  },
  {
    'name': 'Carol',
    'scores': MAP { 'math': 88, 'science': 79, 'english': 95 },
    'tags': MAP { 'status': 'inactive', 'priority': 'low' }
  }
>>
```

```sql
-- Filter with unpivot — only entries with score > 80.
-- UNPIVOT decomposes each student's scores MAP into (k, v) pairs.
-- The lateral join produces one row per map entry per student,
-- then WHERE filters to only entries where the score exceeds 80.
SELECT t.name, k as subject, v as score
FROM students AS t, UNPIVOT t.scores AS v AT k
WHERE v > 80
```

Result:
```
<<
  {
    'name': 'Alice',
    'subject': 'math',
    'score': 90
  },
  {
    'name': 'Alice',
    'subject': 'science',
    'score': 85
  },
  {
    'name': 'Bob',
    'subject': 'science',
    'score': 92
  },
  {
    'name': 'Carol',
    'subject': 'math',
    'score': 88
  },
  {
    'name': 'Carol',
    'subject': 'english',
    'score': 95
  }
>>
```

## Ordering by Key/Value

Results can be ordered by accessing specific keys or by decomposing with UNPIVOT. If an implementation defines comparison ordering for MAP values, `ORDER BY` on a MAP-typed column is also supported (see Comparison Semantics).

```sql
-- Order rows by a specific map entry's value.
-- Accesses the 'math' key from each student's scores MAP,
-- then orders the entire result set by that value descending.
SELECT t.name, t.scores['math'] as math
FROM students AS t
ORDER BY t.scores['math'] DESC
```

Result:
```
<<
  {
    'name': 'Alice',
    'math': 90
  },
  {
    'name': 'Carol',
    'math': 88
  },
  {
    'name': 'Bob',
    'math': 75
  }
>>
```

```sql
-- Order map entries by key.
-- UNPIVOT flattens each student's scores MAP into rows,
-- then ORDER BY sorts all entries alphabetically by subject name.
SELECT t.name, k as subject, v as score
FROM students AS t, UNPIVOT t.scores AS v AT k
ORDER BY k ASC
```

Result:
```
<<
  {
    'name': 'Alice',
    'subject': 'english',
    'score': 72
  },
  {
    'name': 'Carol',
    'subject': 'english',
    'score': 95
  },
  {
    'name': 'Bob',
    'subject': 'math',
    'score': 75
  },
  {
    'name': 'Carol',
    'subject': 'math',
    'score': 88
  },
  {
    'name': 'Alice',
    'subject': 'math',
    'score': 90
  },
  {
    'name': 'Carol',
    'subject': 'science',
    'score': 79
  },
  {
    'name': 'Alice',
    'subject': 'science',
    'score': 85
  },
  {
    'name': 'Bob',
    'subject': 'science',
    'score': 92
  }
>>
```

```sql
-- Order map entries by value.
-- Same UNPIVOT decomposition, but sorts by score descending
-- to show highest scores first across all students and subjects.
SELECT t.name, k as subject, v as score
FROM students AS t, UNPIVOT t.scores AS v AT k
ORDER BY v DESC
```

Result:
```
<<
  {
    'name': 'Carol',
    'subject': 'english',
    'score': 95
  },
  {
    'name': 'Bob',
    'subject': 'science',
    'score': 92
  },
  {
    'name': 'Alice',
    'subject': 'math',
    'score': 90
  },
  {
    'name': 'Carol',
    'subject': 'math',
    'score': 88
  },
  {
    'name': 'Alice',
    'subject': 'science',
    'score': 85
  },
  {
    'name': 'Carol',
    'subject': 'science',
    'score': 79
  },
  {
    'name': 'Bob',
    'subject': 'math',
    'score': 75
  },
  {
    'name': 'Alice',
    'subject': 'english',
    'score': 72
  }
>>
```

## Aggregation by Key/Value

MAP entries can be aggregated by accessing specific keys or by decomposing with UNPIVOT:

```sql
-- Count entries per key across multiple maps.
-- UNPIVOT flattens all students' scores MAPs, then GROUP BY k
-- counts how many students have each subject in their scores.
SELECT k, COUNT(*) AS cnt
FROM students AS t, UNPIVOT t.scores AS v AT k
GROUP BY k
```

Result:
```
<<
  {
    'k': 'math',
    'cnt': 3
  },
  {
    'k': 'science',
    'cnt': 3
  },
  {
    'k': 'english',
    'cnt': 2
  }
>>
```

```sql
-- Sum values grouped by key.
-- Computes the total score per subject across all students
-- by summing the values (v) after grouping by key (k).
SELECT k, SUM(v) AS total
FROM students AS t, UNPIVOT t.scores AS v AT k
GROUP BY k
```

Result:
```
<<
  {
    'k': 'math',
    'total': 253
  },
  {
    'k': 'science',
    'total': 256
  },
  {
    'k': 'english',
    'total': 167
  }
>>
```

```sql
-- Aggregate by value — group students by their math score.
-- Accesses a specific key ('math') from each student's scores MAP,
-- then groups by that value to count how many students share the same score.
SELECT t.scores['math'] AS math_score, COUNT(*) AS cnt
FROM students AS t
GROUP BY t.scores['math']
```

Result:
```
<<
  {
    'math_score': 90,
    'cnt': 1
  },
  {
    'math_score': 75,
    'cnt': 1
  },
  {
    'math_score': 88,
    'cnt': 1
  }
>>
```

## Comparison Semantics

MAP comparison semantics are **implementation-defined**. Implementations may choose to support equality, ordering, or both.

**Equality:** If an implementation defines equality for MAP values, two maps are equal if and only if they contain the same set of key-value pairs, regardless of insertion order. When equality is defined, operations that depend on it are supported:

- `=` and `<>` comparison between MAP values
- `GROUP BY` on a MAP-typed column
- `DISTINCT` on a MAP-typed column
- MAP values as join keys

**Ordering:** If an implementation defines a comparison order for MAP values (e.g., by comparing sorted entries lexicographically), operations that depend on ordering are supported:

- `ORDER BY` on a MAP-typed column
- `<`, `>`, `<=`, `>=` between MAP values
- Window functions with MAP-typed `ORDER BY` expressions

Implementations that do not define ordering for MAP values should raise an error when these operations are attempted on MAP-typed columns.


# Drawbacks

- **Complexity**: Adding a new first-class type increases the surface area of the language — parser, planner, evaluator, and serialization all need MAP-aware paths.
- **Potential confusion with STRUCT**: Users may be uncertain when to use STRUCT vs MAP. The distinction (STRUCT = schemaless with string keys; MAP = typed with parameterized keys) requires clear documentation.
- **Colon overloading**: The `:` separator in MAP constructor (`MAP { k: v }`) is also used in STRUCT literals (`{ k: v }`). The `MAP` keyword prefix disambiguates at the parser level, but may cause visual confusion.
- **No SQL standard**: MAP is not part of the ISO SQL standard. This is a PartiQL extension inspired by other systems (Hive, Trino, Spark SQL).

# Rationale and alternatives

## Why this design?

- **Angle brackets `MAP<K, V>`** — Consistent with existing `ARRAY<T>` syntax in PartiQL and with precedent in Trino, Spark SQL, and Presto. Parentheses (`MAP(K, V)`) were considered but risk confusion with function calls.
- **`MAP { k: v }` construction** — Mirrors struct literal syntax with a keyword prefix for disambiguation. Alternatives considered:
  - `MAP(k1, v1, k2, v2)` — positional args are error-prone with many entries
  - `MAP [k1, v1], [k2, v2]` — verbose tuple-list syntax
  - `{ k: v } :: MAP<K,V>` — cast-style is semantically indirect (builds struct then converts)
- **Bracket access `m[expr]`** — Reuses existing path syntax; works naturally for non-string keys (`m[42]`, `m[date_var]`). Dot notation (`m.key`) only works for string keys and is ambiguous with struct field access.
- **No implicit STRUCT-MAP coercion** — STRUCT and MAP have fundamentally different semantics (duplicate keys, key types, value uniformity). Requiring explicit CAST prevents surprising behavior.

## Alternatives not chosen

| Alternative | Description | Reason for rejection |
|---|---|---|
| Extend STRUCT with type parameters | Add optional `STRUCT<K, V>` parameterization | Would break backward compatibility and conflate two distinct semantic models |
| Use function-only interface | Provide MAP operations as functions without a type | Cannot enable static type reasoning; no enforcement of key/value type uniformity |
| String-only keys (like STRUCT) | Restrict MAP keys to STRING type | Misses the primary use case of integer-keyed and date-keyed maps common in analytics |
| Allow NULL keys | Permit NULL as a valid key | NULL equality is undefined (`NULL = NULL` -> `NULL`), making lookup unreliable |

## Impact of not doing this

Without MAP, users must use STRUCT for dictionary data, losing:

- Static type reasoning (planner cannot infer element types)
- Key uniqueness guarantees
- Non-string key support
- Compatibility with external systems (Hive, Iceberg) that expose MAP-typed columns

# Prior art

## SQL Dialects with MAP Support

- **Trino/Presto**: `MAP(K, V)` type (parentheses), `MAP(ARRAY[k1,k2], ARRAY[v1,v2])` or `MAP()` constructor, `m[key]` access (throws on missing key), `element_at(m, key)` (returns NULL on missing). Functions: `map_keys()`, `map_values()`, `map_entries()`, `map_filter()`, `transform_keys()`, `transform_values()`, `map_concat()`, `map_zip_with()`.
- **Spark SQL**: `MAP<K,V>` type (angle brackets), `map(k1,v1,k2,v2)` or `map_from_arrays(keys, values)` constructor, `m[key]` access. Functions: `map_keys()`, `map_values()`, `map_entries()`, `map_filter()`, `transform_keys()`, `transform_values()`, `map_concat()`, `map_contains_key()`, `element_at()`.
- **Hive**: `MAP<K,V>` type, `map(k1,v1,k2,v2)` constructor, `m[key]` access.
- **Apache Iceberg**: `map<key_id: K, value_id: V>` in schema definition. Keys are always non-null. Values can be optional or required. Any type (including nested) is allowed as key. Maps directly to engine-native MAP types.
- **DuckDB**: `MAP(K, V)` type (parentheses), `MAP {'key1': 'value1'}` literal constructor, `m[key]` access (returns NULL on missing). Functions: `map_keys()`, `map_values()`, `map_entries()`, `map_contains()`, `map_extract()`, `map_concat()`.

## Key Design Choices Across Systems

| Feature | Trino | Spark SQL | Hive | DuckDB |
|---|---|---|---|---|
| Type syntax | `MAP(K, V)` | `MAP<K, V>` | `MAP<K, V>` | `MAP(K, V)` |
| Constructor | `MAP(ARRAY[], ARRAY[])` | `map(k, v, ...)` | `map(k, v, ...)` | `MAP {'k': 'v'}` |
| Access | `m[key]` (throws on miss) | `m[key]` | `m[key]` | `m[key]` (NULL on miss) |
| NULL keys | Disallowed | Disallowed | Allowed | Disallowed |
| Duplicate keys | Error | Configurable (error default, last-win option) | Last-write-wins | Error |
| Missing key | `element_at()` returns NULL | `element_at()` returns NULL | Returns NULL | Returns NULL |
| Equality (`=`) | Supported (order-independent) | Not supported | N/A | Supported (order-sensitive) |
| Numeric coercion on lookup | Implicit cast | Implicit cast | N/A | Implicit cast |

## Programming Languages

- **Python (`dict`)**: PartiQL's MAP design draws from Python's dictionary model — key-based lookup, bracket access syntax (`m[key]`), and the concept that any "hashable + equality-comparable" type can serve as a key. PartiQL's permissive mode uses last-write-wins for duplicate keys, matching Python's behavior. Key differences from Python: PartiQL MAPs are immutable (like SQL values), keys and values are typed (not heterogeneous by default), and missing key access returns `MISSING` rather than raising an exception.

## ISO SQL Standard

The ISO SQL standard does not define a MAP type. The closest construct is `MULTISET` (a bag of rows), which can model key-value pairs but without key-uniqueness or typed-key semantics. PartiQL's MAP design is a pragmatic extension drawing from the consensus across Trino, Spark, and Hive.

# Unresolved questions

The following questions are expected to be resolved through the RFC process:

1. **Ordering for ORDER BY**: For implementations that choose to support MAP ordering, what is the recommended algorithm? Options include: by sorted entries (lexicographic), by size then sorted entries, or left entirely to the implementation.


2. **PIVOT producing MAP**: Should `PIVOT ... AT ...` produce MAP instead of STRUCT when the key type is non-string?


# Future possibilities

- **DYNAMIC key type support**: `DYNAMIC` as a value type is allowed (e.g., `MAP<STRING, DYNAMIC>` permits heterogeneous values). Support for `DYNAMIC` as a *key* type parameter (e.g., `MAP<DYNAMIC, STRING>`, `MAP<DYNAMIC, DYNAMIC>`, bare `MAP`) is under investigation and may be added in a future revision. This would allow heterogeneous keys within a single map, similar to Python's `dict`. Key questions that need resolution include: cross-type numeric equality (is `1` INT the same key as `1.0` DECIMAL?), hashing strategy for heterogeneous keys, and whether bare `MAP` without type parameters should default to `MAP<DYNAMIC, DYNAMIC>`.
- **Collection types as keys**: Extend allowed key types to include `ARRAY` and `BAG` once well-defined equality semantics for collection types are established. This would enable use cases like composite keys (e.g., `MAP<ARRAY<INT>, STRING>`).
- **MAP Constructor**: A syntax like comprehension `MAP { k: v FOR k, v IN source }` for constructing maps from query results.
- **MAP_FILTER**: A higher-order function to filter map entries by predicate on key and/or value.
- **MAP_TRANSFORM_KEYS / MAP_TRANSFORM_VALUES**: Higher-order functions for transforming keys or values (as in Spark SQL).
- **MAP aggregation**: An aggregate function like `MAP_AGG(key_expr, value_expr)` to construct a MAP from grouped rows (similar to Trino's `map_agg`).
- **CONTAINS_KEY as operator**: Syntax like `key IN KEYS(map)`
