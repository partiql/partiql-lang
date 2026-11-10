- Start Date: 2026-11-05
- PartiQL Issue: [partiql/partiql-lang#106](https://github.com/partiql/partiql-lang/issues/106)
- RFC PR: [partiql/partiql-lang#107](https://github.com/partiql/partiql-lang/pull/107)

# Summary

This RFC adds a `JSON` type to PartiQL, with operators to parse, serialize, query and construct JSON values. It follows the JSON support in ISO/IEC 9075-2:2023 (SQL:2023), and refers to SQL:2023 for the details the two share, such as the SQL/JSON path language. The RFC focuses on how this support fits into PartiQL, such as how it works with `MISSING`, permissive and type checking mode, and expressions whose type is not known at compile time, and where PartiQL differs from SQL:2023 (see [Where PartiQL differs](#where-partiql-differs)). Parts that need further design, such as `JSON_TABLE`, are listed under [Unresolved questions](#unresolved-questions).

# Motivation

Much semi-structured data reaches a query engine as JSON text in a string column. PartiQL can navigate tuples, arrays and Ion values, but it has no way to treat a string as JSON. Given a string column `payload` that holds `{"customer": {"name": "Bob"}}`, the path `t.payload.customer` asks a string for a field, and returns `MISSING` in permissive mode or fails in type checking mode. Users fall back to engine-specific functions, which differ in name and behavior across engines.

SQL:2023 already defines JSON support, and several SQL engines implement much of it (see [Prior art](#prior-art)). Following SQL:2023 gives PartiQL:

1. A standard way to parse, read, iterate and build JSON.
2. Behavior that users of other SQL engines already know.
3. A clear mapping when a PartiQL query is translated to another SQL engine.

SQL:2023 defines JSON as a type, not as a set of functions over strings, and this RFC does the same. A value of type `JSON` can be read with PartiQL's existing path syntax, such as `JSON(o.payload).customer.name`, and passed from one operator to the next without being parsed again.

PartiQL can already query JSON documents as data, because a JSON object read from a source becomes a tuple. The `JSON` type is for the other case, JSON that a query receives as text in a column, or that a query builds and returns.

## Non-goals

- Changing how tuples, arrays or Ion values behave. Every existing query keeps its result.
- Defining storage or encoding of JSON values. That is up to each implementation.

# Specification

Each part below names the chapter of the PartiQL specification it adds to. Within a chapter, new sections come first, then changes to sections the chapter already has.

## Conventions

**Grammar.** Rules use the EBNF style of the PartiQL specification: `=` defines a rule, terminals are in quotes, `[ ]` is optional, `{ }` repeats, `|` separates alternatives, `;` ends a rule, and `(* *)` is a comment. `expression`, `query`, `identifier`, `string_literal` and `order_by_item` are the specification's existing rules. `data_type` is a type name from the Data Types chapter, such as `INT`, `STRING` or `JSON`. As in the specification, a type name is written as a terminal, and the chapter has no rule that lists them all.

**Examples.** `->` means "evaluates to", `<=>` means "is the same as", and `--` starts a comment, as in SQL. Each result starts with its type, which can differ from what its content looks like. `JSON {"c":1}` is a JSON value. `STRING '{"c":1}'` is a string that holds JSON text, as `o.payload` does. `STRING '…'` is any string, `INT …` is an integer, `true` and `false` are booleans, and `NULL` is SQL `NULL`. Serialized JSON text may differ from what is shown in whitespace.

**Example data.** Examples use one row of a table `orders`. Its column `payload` has the type `STRING`. Its value is JSON text, which is still a string, not a JSON value:

```sql
orders = << {
  'payload': '{"id": 7, "customer": {"name": "Bob", "Tier": "gold"}, "items": [{"sku": "A1", "price": 10.5}, {"sku": "B2", "price": 4}], "note": null}'
} >>
```

The same text, laid out to read:

```json
{"id": 7,
 "customer": {"name": "Bob", "Tier": "gold"},
 "items": [{"sku": "A1", "price": 10.5}, {"sku": "B2", "price": 4}],
 "note": null}
```

`o` is the `orders` table. `o.payload` is the STRING above, and `p` stands for `JSON(o.payload)`, the JSON value parsed from it.

## Typing Model: JSON when the type is not known at compile time

The specification's gradual typing rules apply to JSON as to any other type. When an expression's type is not known at compile time, for example when there is no schema, a value that is JSON at runtime follows the rules for an expression declared `JSON`. A use that would be a compile error for a declared `JSON` expression gives `MISSING` in permissive mode and an error in type checking mode.

Two rules are specific to JSON:

- A JSON value is never converted to a tuple, an array or an SQL scalar to make an operation work. Use `JSON_VALUE` or `JSON_QUERY` to read from it.
- The constructors choose their result type at compile time (§6.34), so an input whose type is not known gives a `STRING`, even if its value is JSON (see [JSON constructors](#json-constructors-json_object-json_array-json_arrayagg-json_objectagg)). Here the result depends on whether the type was known at compile time.

## Data Types: JSON Type

The type name is `JSON`, and it has no parameters (§4.8). As in SQL:2023, a query cannot write a value of type `JSON` directly, the way it writes `7` or `DATE '2024-01-01'`. A string such as `'{"a": 1}'` is a `STRING`. A JSON value comes from `JSON(text)`, `JSON_SCALAR` or a constructor.

`JSON` and the other new words in this RFC are non-reserved, so existing names such as a column called `json` still work.

**Values.** A value of type `JSON` is an *SQL/JSON item* (§4.48.3):

- an SQL/JSON scalar, which is a non-null string, number, boolean or datetime
- SQL/JSON null
- an SQL/JSON array of items
- an SQL/JSON object, whose members each pair a string key with an item

A JSON number is an exact numeric value (§9.42), so `10.5` keeps its exact value. JSON text has no datetime, so parsing never produces one. `JSON_SCALAR` on a datetime does.

An expression whose type is not known at compile time can also evaluate to a JSON value. For example, the array below mixes a JSON value and a string, so `x` has no known type at compile time:

```sql
SELECT VALUE x FROM [JSON('{"a": 1}'), 'abc'] AS x
-- Result: << JSON {"a": 1}, STRING 'abc' >>
-- x is JSON in the first row and a STRING in the second
```

A table with no schema is another case. [Typing Model](#typing-model-json-when-the-type-is-not-known-at-compile-time) gives the rules for such values.

**Absent values.** An expression of type `JSON` can also evaluate to SQL `NULL` or to `MISSING`. These are absent values, not JSON values:

| Value                                               | What it is             | `IS NULL` | `IS MISSING` |
| --------------------------------------------------- | ---------------------- | --------- | ------------ |
| SQL/JSON null, such as `p.note`                     | A JSON value           | false     | false        |
| SQL `NULL`, such as `p.age`                         | No value               | true      | false        |
| `MISSING`, such as `JSON('abc')` in permissive mode | No value, PartiQL only | true      | true         |

**No `MISSING` inside JSON.** `MISSING` is never part of a JSON value, neither as a key nor as a value, just as it is never an attribute value of a tuple. Keys are always strings, and JSON text allows only three literals, `true`, `false` and `null` ([RFC 8259 §3](https://www.rfc-editor.org/rfc/rfc8259#section-3)), so text such as `{"note": missing}` is not valid JSON. An absent key is just absent, and reading it gives SQL `NULL`, not `MISSING`:

| In the JSON text        | Example             | `p.note`                                                                                            | `JSON_EXISTS(p, 'lax $.note')`                                                             |
| ----------------------- | ------------------- | --------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| The key holds `null`    | `{"note": null}`    | JSON `null`                                                                                         | true                                                                                       |
| The key is absent       | `{"id": 7}`         | `NULL`                                                                                              | false                                                                                      |
| The text says `missing` | `{"note": missing}` | No value. `JSON(...)` fails, giving `MISSING` in permissive mode and an error in type checking mode | False on the string, because the text is not valid JSON and `ON ERROR` defaults to `FALSE` |

**Mixing with other types.** In `CASE`, `COALESCE`, `NULLIF` and the set operations, if any input is `JSON`, every input must be `JSON` (§9.5). `NULL` and `MISSING` literals are allowed. PartiQL's array, bag and tuple constructors still accept mixed values.

## Type Conversions: CAST to and from JSON

| CAST                                                      | Result                                                                                                                                                                                                                              |
| --------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `CAST(j AS JSON)`, where `j` is `JSON`                    | `j` unchanged (§6.13)                                                                                                                                                                                                               |
| `CAST(NULL AS JSON)`                                      | SQL `NULL` of type `JSON` (§6.13)                                                                                                                                                                                                   |
| `CAST(v AS JSON)`, where `v` is an Ion value | Allowed. How each Ion value maps to JSON is to be decided (see [Unresolved questions](#unresolved-questions)) |
| CAST from `JSON` to an Ion type | Not allowed |
| Any other CAST into or out of `JSON`                      | Not allowed. It is a compile error when the types are known at compile time, and otherwise `MISSING` in permissive mode and an error in type checking mode. Use `JSON(...)`, `JSON_SERIALIZE` or `JSON_VALUE ... RETURNING` instead |

## Functions and Operators

### JSON Functions and Operators

#### JSON parse: `JSON(text)`

`JSON(text)` turns a string that holds JSON text into a JSON value (§6.40). SQL:2023 does not allow a CAST from a string to JSON (§6.13), so parsing is its own operator.

**Syntax.**

```ebnf
json_parse       = "JSON" "(" json_input [ json_unique_keys ] ")" ;
json_input       = expression [ "FORMAT" "JSON" ] ;
json_unique_keys = ( "WITH" | "WITHOUT" ) "UNIQUE" [ "KEYS" ] ;
```

**Rules.**

- `json_input` is an input to a JSON operator, optionally followed by `FORMAT JSON`. `FORMAT JSON` says that a string input holds JSON text. It does not change the input's type, which stays `STRING`. `JSON(...)`, `IS JSON` and the query operators already read a string input as JSON text, so `FORMAT JSON` changes nothing for them. On a constructor input it does (see [JSON constructors](#json-constructors-json_object-json_array-json_arrayagg-json_objectagg)). `ENCODING`, which SQL:2023 allows after `FORMAT JSON` for binary strings, is under [Unresolved questions](#unresolved-questions).
- The argument must be a string. Its text must be a JSON text ([RFC 8259](https://www.rfc-editor.org/rfc/rfc8259)), parsed as §9.42 describes.
- Parsing keeps every member, including members with duplicate keys. This is `WITHOUT UNIQUE KEYS`, the default.
- With `WITH UNIQUE KEYS`, text with a duplicate key in any object is not accepted (§9.42). The result is `MISSING` in permissive mode and an error in type checking mode, as for invalid text.
- If the text is not valid JSON, the result is `MISSING` in permissive mode and an error in type checking mode. SQL:2023 raises an error here, as it does for a failed CAST such as `CAST('abc' AS INT)`. PartiQL's permissive mode turns that kind of data error into `MISSING`.
- A SQL `NULL` argument gives SQL `NULL`, and a `MISSING` argument gives `MISSING`.

**Examples.**

```sql
JSON(o.payload)               ->  JSON {"id": 7, "customer": {...}, "items": [...], "note": null}
JSON('"abc"')                 ->  JSON "abc", a JSON string
JSON('abc')                   ->  MISSING in permissive mode, an error in type checking mode  -- abc is not JSON
JSON('{"a": 1, "a": 2}').a    ->  JSON 1  -- both members are kept, and the first is returned
```

#### `JSON_SERIALIZE` and `JSON_SCALAR`

**Syntax.**

```ebnf
json_serialize = "JSON_SERIALIZE" "(" expression [ json_returning ] ")" ;
json_scalar    = "JSON_SCALAR" "(" expression ")" ;
json_returning = "RETURNING" data_type [ "FORMAT" "JSON" ] ;
```

**Rules.**

- `JSON_SERIALIZE(j)` returns the JSON text of a JSON value as a `STRING` (§6.37). Parsing the result gives an equivalent value.
- A JSON value that holds a datetime has no JSON text, so serializing it fails (§9.43). The result is `MISSING` in permissive mode and an error in type checking mode. The same applies wherever a JSON value becomes text, such as `RETURNING STRING`.
- `JSON_SCALAR(x)` turns a string, number, boolean or datetime into a JSON scalar (§6.41).
- Both return SQL `NULL` for a SQL `NULL` argument, and `MISSING` for a `MISSING` argument.
- `FORMAT JSON` may also follow a `RETURNING` type. It changes nothing there, because JSON is the only format.

**Examples.**

```sql
JSON_SERIALIZE(p.customer)    ->  STRING '{"name":"Bob","Tier":"gold"}'
JSON_SCALAR(10.5)             ->  JSON 10.5
JSON_SCALAR('Bob')            ->  JSON "Bob"
```

#### `IS JSON`

`IS JSON` is SQL:2023's predicate (§8.22). It tests content, not type. It asks whether `x` holds valid JSON of the given kind. It is how a query filters out bad rows before it parses them.

**Syntax.**

```ebnf
json_predicate = json_input "IS" [ "NOT" ] "JSON"
                 [ "VALUE" | "ARRAY" | "OBJECT" | "SCALAR" ]
                 [ json_unique_keys ] ;
json_input     = expression [ "FORMAT" "JSON" ] ;   (* as in "JSON parse" *)
```

**Rules.**

The optional keyword says what kind of JSON value must be at the top level. With no keyword, it is `VALUE`:

| Predicate | True when `x` holds | Example that is true |
|---|---|---|
| `x IS JSON`, `x IS JSON VALUE` | Any valid JSON | `'{"a": 1}'`, `'[1, 2]'`, `'7'` |
| `x IS JSON OBJECT` | A JSON object | `'{"a": 1}'` |
| `x IS JSON ARRAY` | A JSON array | `'[1, 2]'` |
| `x IS JSON SCALAR` | A string, number, boolean or `null` | `'7'`, `'"abc"'`, `'null'` |

`x IS NOT JSON …` is the negation. With `WITH UNIQUE KEYS`, the predicate is also false when any object in `x` has a duplicate key (§8.22).

| Input | Result |
|---|---|
| A string that holds valid JSON of the requested kind | true |
| A string that does not, such as `'abc'` | false |
| A JSON value | `IS JSON` and `IS JSON VALUE` are true. `IS JSON OBJECT`, `ARRAY` and `SCALAR` test the kind of value |
| SQL `NULL` | `NULL` |
| `MISSING` | `MISSING` |
| A value of another declared type, such as `1 IS JSON` | A compile error, as in SQL:2023 |
| An expression whose type is not known at compile time, and whose value is neither a string nor JSON | false |

`IS JSON` is not one of PartiQL's type predicates, such as `x IS STRING`. Those test a value's type, so `'{}' IS STRUCT` is false. `IS JSON` tests what a string holds, so `'{}' IS JSON` is true. The grammar reads `IS JSON` as this predicate in every case.

**Examples.**

```sql
o.payload IS JSON             ->  true
o.payload IS JSON OBJECT      ->  true
o.payload IS JSON ARRAY       ->  false
o.payload IS JSON SCALAR      ->  false  -- the top level is an object
'7' IS JSON SCALAR            ->  true
'null' IS JSON SCALAR         ->  true   -- null counts as a scalar here (§8.22)
'abc' IS JSON                 ->  false
JSON('[1, 2]') IS JSON ARRAY  ->  true
```

#### JSON query operators: `JSON_VALUE`, `JSON_QUERY`, `JSON_EXISTS`

Each query operator takes an input, which is a string that holds JSON or a JSON value, and a path string.

**Path strings.** The second argument of each query operator is a path string, written in the SQL/JSON path language of SQL:2023 (§9.45, §9.46). The PartiQL specification does not have this language yet, so this RFC adopts it by reference instead of defining it again.

```sql
'lax $.customer.name'
--  lax        the mode, lax or strict
--  $          the whole input value
--  .customer  a member step
--  .name      a member step
```

- *Shape.* A path starts with its mode, then `$`, then its steps, such as `.key` for a member and `[n]` for an array element, counting from 0.
- *The whole language applies.* This includes wildcards (`.*`, `[*]`), several subscripts and ranges (`[0, 2 to last]`), filters (`?(@.price > 5)`, with `like_regex` and `starts with`), arithmetic, item methods such as `.number()`, and variables such as `$i`, which `PASSING` binds.
- *Only JSON inside.* A path works only on SQL/JSON items, so `MISSING`, tuples and Ion values never occur inside it.
- *No effect from PartiQL's execution mode.* What happens when a path finds nothing or fails depends on its own mode and on the `ON EMPTY` and `ON ERROR` clauses.
- *PartiQL paths are unchanged.* Path expressions such as `t.a` and `t['a']` keep their current rules.

**Syntax.**

```ebnf
json_value           = "JSON_VALUE" "(" json_api_common
                       [ json_returning ]
                       [ json_value_behavior "ON" "EMPTY" ]
                       [ json_value_behavior "ON" "ERROR" ] ")" ;
json_value_behavior  = "ERROR" | "NULL" | "DEFAULT" expression ;

json_query           = "JSON_QUERY" "(" json_api_common
                       [ json_returning ]
                       [ json_wrapper "WRAPPER" ]
                       [ ( "KEEP" | "OMIT" ) "QUOTES" [ "ON" "SCALAR" "STRING" ] ]
                       [ json_query_behavior "ON" "EMPTY" ]
                       [ json_query_behavior "ON" "ERROR" ] ")" ;
json_wrapper         = "WITHOUT" [ "ARRAY" ]
                     | "WITH" [ "CONDITIONAL" | "UNCONDITIONAL" ] [ "ARRAY" ] ;
json_query_behavior  = "ERROR" | "NULL" | "EMPTY" "ARRAY" | "EMPTY" "OBJECT" ;

json_exists          = "JSON_EXISTS" "(" json_api_common
                       [ json_exists_behavior "ON" "ERROR" ] ")" ;
json_exists_behavior = "TRUE" | "FALSE" | "UNKNOWN" | "ERROR" ;

json_api_common      = json_input "," json_path_string [ json_passing ] ;
json_path_string     = string_literal ;   (* a path in the SQL/JSON path language *)
json_passing         = "PASSING" json_input "AS" identifier
                       { "," json_input "AS" identifier } ;
json_input           = expression [ "FORMAT" "JSON" ] ;   (* as in "JSON parse" *)
```

**Rules for all three.**

- The operators follow SQL:2023 (§6.28, §6.35, §8.23).
- A SQL `NULL` input gives SQL `NULL`. A `MISSING` input gives `MISSING`.
- An input that is neither a string nor JSON is a compile error when its type is known at compile time, and otherwise `MISSING` in permissive mode and an error in type checking mode.
- `PASSING x AS i` binds the path variable `$i` to the value of `x` (§9.46, §10.14). A JSON value is passed as it is, and a string, number, boolean or datetime becomes the matching SQL/JSON scalar. Passing SQL `NULL`, `MISSING`, or a tuple, array, bag or Ion value is to be decided (see [Unresolved questions](#unresolved-questions)).
- In lax mode, a key that does not exist is not an error. The path finds nothing, so `ON EMPTY` applies. Other errors, such as an input string that is not valid JSON, or a value that cannot convert to the `RETURNING` type, go to `ON ERROR`.

**Defaults, from SQL:2023.**

| Clause                                                  | Default                                                                                |
| ------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| `RETURNING` in `JSON_VALUE`                             | `STRING`                                                                               |
| `RETURNING` in `JSON_QUERY`                             | `JSON` if the input is `JSON`, and `STRING` if the input is a string                   |
| `ON EMPTY`, `ON ERROR` in `JSON_VALUE` and `JSON_QUERY` | `NULL`                                                                                 |
| `ON ERROR` in `JSON_EXISTS`                             | `FALSE`                                                                                |
| Wrapper in `JSON_QUERY`                                 | `WITHOUT ARRAY WRAPPER`. `WITH WRAPPER` alone means `WITH UNCONDITIONAL ARRAY WRAPPER` |
| Quotes in `JSON_QUERY` | `KEEP QUOTES`. A quotes clause cannot follow `WITH ... WRAPPER`, and `OMIT QUOTES` is not allowed when the result type is `JSON` (§6.35) |

**`JSON_VALUE`** returns one SQL scalar. SQL/JSON null comes back as SQL `NULL`. If the path finds an object, an array or more than one value, that is an error, which `ON ERROR` handles.

**`JSON_QUERY`** returns a JSON value, or its text as a `STRING`. Without a wrapper, more than one result is an error. `WITH CONDITIONAL ARRAY WRAPPER` wraps the results in an array only when there is more than one.

**`JSON_EXISTS`** returns true if the path finds at least one value, and false if it finds none. A key that holds JSON `null` still counts as found.

**Examples.**

```sql
-- JSON_VALUE returns one SQL scalar, a STRING by default
JSON_VALUE(o.payload, 'lax $.customer.name')                    ->  STRING 'Bob'
JSON_VALUE(o.payload, 'lax $.id' RETURNING INT)                 ->  INT 7
JSON_VALUE(p, 'lax $.note')                                     ->  NULL   -- JSON null comes back as SQL NULL

-- JSON_VALUE: ON EMPTY, when the path finds nothing
JSON_VALUE(p, 'lax $.age')                                      ->  NULL   -- ON EMPTY defaults to NULL
JSON_VALUE(p, 'lax $.age' RETURNING INT DEFAULT 0 ON EMPTY)     ->  INT 0
JSON_VALUE(p, 'lax $.age' ERROR ON EMPTY)                       ->  an error

-- JSON_VALUE: ON ERROR, when the path finds an object, an array or several values
JSON_VALUE(o.payload, 'lax $.customer')                         ->  NULL   -- ON ERROR defaults to NULL
JSON_VALUE(p, 'lax $.items.sku')                                ->  NULL   -- two values
JSON_VALUE(p, 'lax $.items.sku' ERROR ON ERROR)                 ->  an error

-- JSON_QUERY returns JSON for a JSON input, and a STRING for a string input
JSON_QUERY(p, 'lax $.customer')                                 ->  JSON {"name":"Bob","Tier":"gold"}
JSON_QUERY(o.payload, 'lax $.customer')                         ->  STRING '{"name":"Bob","Tier":"gold"}'

-- JSON_QUERY: wrappers
JSON_QUERY(p, 'lax $.items.price')                              ->  NULL       -- two results without a wrapper is an error
JSON_QUERY(p, 'lax $.items.price' WITH CONDITIONAL ARRAY WRAPPER)  ->  JSON [10.5, 4]
JSON_QUERY(p, 'lax $.id' WITH CONDITIONAL ARRAY WRAPPER)        ->  JSON 7     -- one result is not wrapped
JSON_QUERY(p, 'lax $.id' WITH WRAPPER)                          ->  JSON [7]   -- WITH WRAPPER is UNCONDITIONAL, so it always wraps

-- JSON_QUERY: quotes, for a JSON string returned as text
JSON_QUERY(p, 'lax $.customer.name' RETURNING STRING)              ->  STRING '"Bob"'  -- KEEP QUOTES is the default
JSON_QUERY(p, 'lax $.customer.name' RETURNING STRING OMIT QUOTES)  ->  STRING 'Bob'
JSON_QUERY(p, 'lax $.id' WITH WRAPPER OMIT QUOTES)                 ->  a compile error  -- no quotes clause after WITH ... WRAPPER

-- JSON_EXISTS
JSON_EXISTS(o.payload, 'lax $.note')                            ->  true   -- the key exists, even though its value is JSON null
JSON_EXISTS(o.payload, 'lax $.age')                             ->  false  -- the key does not exist
JSON_EXISTS(o.payload, 'strict $.age')                          ->  false  -- strict mode raises an error, and ON ERROR defaults to FALSE
```

#### JSON simplified accessor

The simplified accessor is SQL:2023's short way to read JSON (§6.36). Instead of a `JSON_QUERY` call with a path string, a query writes path steps directly after an expression of type `JSON`, using PartiQL's existing steps `.name`, `."name"` and `[n]`. It adds no new syntax.

```sql
p.customer.name      ->  JSON "Bob"
```

Each simplified accessor is the same as one `JSON_QUERY` call (§6.36). The clauses of that call give its rules:

```sql
p.customer.name
<=> JSON_QUERY(p, 'lax $.customer.name'           -- the same steps, in lax mode
               WITH CONDITIONAL ARRAY WRAPPER      -- one result is returned as it is, several are wrapped in an array
               NULL ON EMPTY                       -- a key that does not exist gives SQL NULL
               NULL ON ERROR)                      -- any other error also gives SQL NULL
```

**Rules.**

- It always uses lax mode. A member step on an array runs on each element, and an array step on a value that is not an array treats it as a one-element array.
- Keys match exact case. Arrays count from 0.
- A key that does not exist gives SQL `NULL`, not `MISSING`, in both execution modes. PartiQL uses `MISSING` only where SQL would reject a query, and here SQL:2023 defines the result.
- One result is returned as it is. Several results are wrapped in a JSON array. No result gives SQL `NULL`.
- When keys repeat in an object, a step returns the first member with that key, in text order.
- The result is a JSON value, even when it is a JSON string. To get an SQL value, use `JSON_VALUE`.
- `x['name']` is not allowed on a JSON value, because neither the simplified accessor nor the path language has a bracket key. Write `x."name"`, which also covers keys with dots or spaces. The PartiQL shorthand `x['name']` for `x.'name'` stays a rule for tuples only. When the type is not known at compile time, `x['name']` on a value that is JSON gives `MISSING` in permissive mode and an error in type checking mode.
- `x[e]`, where `e` is an expression that is not a string literal, is an array step whose index is the value of `e`. If `e` is a value of another SQL type, such as a string, the step is an error (§9.46), so the result is SQL `NULL`. When `e` is `NULL`, `MISSING`, or a tuple, array, bag or Ion value, the result is to be decided, together with `PASSING` (see [Unresolved questions](#unresolved-questions)).
- The expression before the steps can be any expression of type `JSON`, including a function call such as `JSON(o.payload)`. SQL:2023 needs parentheses there, as in `(JSON(o.payload)).customer` (§6.36), and PartiQL does not. PartiQL's field access operator on tuples and index operator on arrays already take any expression before the step, such as `COALESCE(t.a, t.b).c`.
- The simplified accessor applies only to an expression of type `JSON`. A string is not parsed, so `o.payload.customer` keeps today's tuple path behavior, and users write `JSON(o.payload).customer`.
- SQL:2023 also allows wildcard steps (`.*`, `[*]`), several subscripts and ranges, single-quoted member steps (`.'name'`) and item methods such as `.number()` in the simplified accessor (§6.36). Each one overlaps PartiQL syntax: `x.*` and `x[*]` are PartiQL path wildcards, `x.'name'` is a PartiQL tuple step, `x[0 to 2]` does not parse today, and `x.number()` reads as a function call. Which of them the simplified accessor allows is to be decided (see [Unresolved questions](#unresolved-questions)). Each works today inside a path string, such as `JSON_QUERY(p, 'lax $.items[*].sku' WITH WRAPPER)`.
- Name resolution runs first, with today's rules. In `JSON(o.payload).customer`, `o.payload` resolves as today, and the steps after `JSON(...)` follow this section.

**Examples.**

```sql
-- Member steps, and exact case
p.customer.name                ->  JSON "Bob"
p.customer.tier                ->  NULL           -- keys match exact case, and the key is "Tier"
p.customer."Tier"              ->  JSON "gold"    -- a quoted step also reaches keys with dots or spaces

-- Array steps, counting from 0
p.items[0].sku                 ->  JSON "A1"
p.items[1].price               ->  JSON 4
p.items[5]                     ->  NULL           -- out of range, which lax mode treats as no result

-- Lax mode
p.items.price                  ->  JSON [10.5, 4] -- .price runs on each element, and the two results are wrapped
p.id[0]                        ->  JSON 7         -- 7 counts as the one-element array [7]

-- One result as it is, several in an array, none as NULL
p.items[0]                     ->  JSON {"sku":"A1","price":10.5}
p.items.sku                    ->  JSON ["A1","B2"]
p.customer.age                 ->  NULL           -- the key does not exist

-- JSON null and SQL NULL
p.note                         ->  JSON null      -- a value, not SQL NULL
p.note IS NULL                 ->  false

-- Repeated keys
JSON('{"a": 1, "a": 2}').a     ->  JSON 1         -- the first member, in text order

-- An index from an expression
p.items[t.i].sku               ->  JSON "B2"      -- t.i is 1
p.items[t.s]                   ->  NULL           -- t.s is the STRING '1', and a string index is an error

-- What the simplified accessor does not do
p.customer['Tier']             ->  a compile error  -- no bracket key on JSON. Write p.customer."Tier"
o.payload.customer             ->  MISSING in permissive mode, an error in type checking mode  -- o.payload is a STRING, and it is not parsed
JSON(o.payload).customer.name  ->  JSON "Bob"     -- parse first, then read

-- The result is JSON, not an SQL value
p.customer.name = 'Bob'                          ->  a compile error  -- a JSON value and a STRING are not comparable
JSON_VALUE(p, 'lax $.customer.name') = 'Bob'     ->  true             -- JSON_VALUE returns the STRING 'Bob'
```

The last two examples show the most common mistake. `p.customer.name` is JSON `"Bob"`, not the STRING `'Bob'`, so comparing it needs `JSON_VALUE` (see [Equality](#equality)).

**How this differs from tuples.** The same path syntax works on tuples and on JSON:

| What happens                   | Tuples and Ion values (unchanged)            | JSON                                                                                     |
| ------------------------------ | -------------------------------------------- | ---------------------------------------------------------------------------------------- |
| The key does not exist         | `MISSING`, or an error in type checking mode | SQL `NULL`, in both modes                                                                |
| Unquoted `.name`               | Matches any case                             | Matches exact case                                                                       |
| A member step reaches an array | `MISSING`, or an error                       | The step runs on each element                                                            |
| `x['name']`                    | Shorthand for `x.'name'`                     | Not allowed. Write `x."name"`                                                            |
| How many results               | Exactly one                                  | Zero or more. Several are wrapped in an array                                            |
| `=`, `GROUP BY`, `ORDER BY`    | Allowed                                      | To be decided (see [Equality](#equality)) |
| `x IS STRUCT`                  | True for a tuple, such as `{'a': 1}`         | False, because a JSON object is JSON, not a tuple. Use `x IS JSON OBJECT`                |
|                                |                                              |                                                                                          |

#### JSON constructors: `JSON_OBJECT`, `JSON_ARRAY`, `JSON_ARRAYAGG`, `JSON_OBJECTAGG`

**Syntax.**

```ebnf
json_object      = "JSON_OBJECT" "("
                   [ json_member { "," json_member } [ json_null_clause ] [ json_unique_keys ] ]
                   [ json_returning ] ")" ;
json_member      = [ "KEY" ] expression "VALUE" json_input
                 | expression ":" json_input ;
json_array       = "JSON_ARRAY" "(" [ json_input { "," json_input } [ json_null_clause ] ]
                   [ json_returning ] ")"
                 | "JSON_ARRAY" "(" query [ json_null_clause ] [ json_returning ] ")" ;
json_arrayagg    = "JSON_ARRAYAGG" "(" json_input [ "ORDER" "BY" order_by_item { "," order_by_item } ]
                   [ json_null_clause ] [ json_returning ] ")" ;
json_objectagg   = "JSON_OBJECTAGG" "(" json_member [ json_null_clause ] [ json_unique_keys ]
                   [ json_returning ] ")" ;
json_null_clause = "NULL" "ON" "NULL" | "ABSENT" "ON" "NULL" ;
json_input       = expression [ "FORMAT" "JSON" ] ;   (* as in "JSON parse" *)
```

**Rules.** The constructors follow SQL:2023 (§6.34, §10.11).

- *Result type.* With no `RETURNING` clause, the result is `JSON` if any input value has the declared type `JSON`, and a `STRING` otherwise. Only values count, not keys. `JSON_ARRAY` over a query has no input values, so it returns a `STRING`. An input whose type is not known at compile time is not declared `JSON`, so it gives a `STRING` even if its value is JSON. `RETURNING JSON` gives `JSON`.
- *`NULL` values.* `JSON_OBJECT` and `JSON_OBJECTAGG` default to `NULL ON NULL`, so a `NULL` value becomes JSON `null`. `JSON_ARRAY` and `JSON_ARRAYAGG` default to `ABSENT ON NULL`, so `NULL` values are left out.
- *`MISSING` values.* A `MISSING` value leaves its member or element out, whichever null clause is used, as a `MISSING` attribute is left out of a tuple.
- *Keys.* A `NULL` key is an error. A `MISSING` key, such as `t.k` when `t` has no attribute `k`, follows PartiQL's rule for tuple attribute names. It leaves its member out in permissive mode, and is an error in type checking mode. This applies to `JSON_OBJECT` and `JSON_OBJECTAGG`. By default (`WITHOUT UNIQUE KEYS`), `JSON_OBJECT` and `JSON_OBJECTAGG` keep members with the same key. With `WITH UNIQUE KEYS`, a duplicate key is an error (§6.34, §10.11), which is `MISSING` in permissive mode.
- *Inputs.* A JSON input is nested as JSON. A string, number or boolean input becomes the matching JSON scalar. A datetime input is cast to `STRING` and becomes a JSON string (§6.34). A string input is not parsed. A string input followed by `FORMAT JSON` is parsed as JSON text, and the parsed value is nested.
- *`FORMAT JSON` and `RETURNING`.* `FORMAT JSON` after an input says how to read that input. `RETURNING` says what type the result has. The two are independent. `FORMAT JSON` does not change the result type, and `RETURNING` does not change how an input is read.
- *Order.* Without `ORDER BY`, `JSON_ARRAYAGG` puts the rows of a group in an array in no defined order. With `ORDER BY`, the elements follow the sort keys, which follow the rules of PartiQL's `ORDER BY`. A sort key of type `JSON` is to be decided, with comparison (see [Equality](#equality)).

**Examples.**

```sql
-- Result type: JSON only if an input value is declared JSON, or with RETURNING JSON
JSON_OBJECT('id' : 7)                          ->  STRING '{"id":7}'                 -- no input is JSON
JSON_OBJECT('id' : 7 RETURNING JSON)           ->  JSON {"id":7}
JSON_OBJECT('c' : p.customer)                  ->  JSON {"c":{"name":"Bob","Tier":"gold"}}  -- p.customer is declared JSON
JSON_ARRAY(SELECT VALUE x FROM <<1, 2>> AS x)  ->  STRING '[1,2]'                    -- a query has no input values, so the result is a STRING. The order is not defined

-- Member values: what each input becomes
JSON_ARRAY(1, true, 'x')                       ->  STRING '[1,true,"x"]'             -- a number, a boolean and a JSON string
JSON_OBJECT('d' : DATE '2024-01-01')           ->  STRING '{"d":"2024-01-01"}'       -- a datetime is cast to a string
JSON_OBJECT('c' : JSON('{"a": 1}'))            ->  JSON {"c":{"a":1}}                -- a JSON input is nested as JSON
JSON_OBJECT('c' : '{"a": 1}')                  ->  STRING '{"c":"{\"a\": 1}"}'      -- a string input is not parsed
JSON_OBJECT('c' : '{"a": 1}' FORMAT JSON)      ->  STRING '{"c":{"a":1}}'            -- FORMAT JSON parses it. The input is still declared STRING
JSON_OBJECT('c' : '{"a": 1}' RETURNING JSON)   ->  JSON {"c":"{\"a\": 1}"}          -- RETURNING JSON changes the result type, not the member value

-- NULL values: kept by the object constructors, left out by the array constructors
JSON_OBJECT('id' : 7, 'note' : NULL)                  ->  STRING '{"id":7,"note":null}'  -- NULL ON NULL is the default
JSON_OBJECT('id' : 7, 'note' : NULL ABSENT ON NULL)   ->  STRING '{"id":7}'
JSON_ARRAY(1, NULL, 'x')                              ->  STRING '[1,"x"]'               -- ABSENT ON NULL is the default
JSON_ARRAY(1, NULL, 'x' NULL ON NULL)                 ->  STRING '[1,null,"x"]'

-- MISSING values: always left out, whichever null clause is used
JSON_OBJECT('id' : 7, 'note' : MISSING)               ->  STRING '{"id":7}'
JSON_OBJECT('id' : 7, 'note' : MISSING NULL ON NULL)  ->  STRING '{"id":7}'
JSON_ARRAY(1, MISSING, 'x' NULL ON NULL)              ->  STRING '[1,"x"]'

-- Keys
JSON_OBJECT(KEY 'id' VALUE 7)                         ->  STRING '{"id":7}'              -- the same as 'id' : 7
JSON_OBJECT(NULL : 7)                                 ->  an error
JSON_OBJECT(MISSING : 7, 'b' : 2)                     ->  STRING '{"b":2}' in permissive mode, an error in type checking mode
JSON_OBJECT('a' : 1, 'a' : 2)                         ->  STRING '{"a":1,"a":2}'         -- WITHOUT UNIQUE KEYS is the default
JSON_OBJECT('a' : 1, 'a' : 2 WITH UNIQUE KEYS)        ->  MISSING in permissive mode, an error in type checking mode
```

The aggregates, over the items of the example row:

```sql
-- JSON_ARRAYAGG with no ORDER BY: the order is not defined
SELECT JSON_ARRAYAGG(JSON_VALUE(i, 'lax $.sku')) AS skus
FROM orders AS o, JSON(o.payload).items AS i
-- Result: <<{'skus': '["A1","B2"]'}>>
-- skus is a STRING, with the elements in some order

-- JSON_ARRAYAGG with ORDER BY, and JSON_OBJECTAGG
SELECT JSON_ARRAYAGG(JSON_VALUE(i, 'lax $.sku') ORDER BY JSON_VALUE(i, 'lax $.price' RETURNING DECIMAL)) AS skus,
       JSON_OBJECTAGG(JSON_VALUE(i, 'lax $.sku') : i) AS by_sku
FROM orders AS o, JSON(o.payload).items AS i
-- Result: <<{'skus': '["B2","A1"]', 'by_sku': JSON {"A1":{"sku":"A1","price":10.5},"B2":{"sku":"B2","price":4}}}>>
-- skus is a STRING, sorted by price. by_sku is JSON, because its value input i is declared JSON
```

### Changes to existing sections

#### Type Mismatch

A function that is not defined for `JSON` treats a JSON argument as a wrongly typed input. It is a compile error when the type is known at compile time, and otherwise `MISSING` in permissive mode and an error in type checking mode. `IS NULL`, `IS MISSING` and `COUNT` accept JSON.

```sql
UPPER(p.customer.name)                             ->  a compile error  -- UPPER is not defined for JSON
UPPER(JSON_VALUE(p, 'lax $.customer.name'))        ->  STRING 'BOB'
p.id + 1                                           ->  a compile error
JSON_VALUE(p, 'lax $.id' RETURNING INT) + 1        ->  INT 8
p.note IS NULL                                     ->  false            -- JSON null is a value
```

#### Absent Value Inputs

Each JSON operator returns SQL `NULL` for a SQL `NULL` input and `MISSING` for a `MISSING` input, unless its section says otherwise. The constructors are the exception: they leave a `MISSING` value out (see [JSON constructors](#json-constructors-json_object-json_array-json_arrayagg-json_objectagg)).

```sql
JSON_VALUE(NULL, 'lax $.a')            ->  NULL
JSON(MISSING)                          ->  MISSING
JSON_OBJECT('a' : MISSING, 'b' : 1)    ->  STRING '{"b":1}'   -- the constructor exception
```

#### Equality

To be decided. SQL:2023 lets JSON values take part in `=`, `<`, `IN`, `DISTINCT`, `GROUP BY`, `ORDER BY`, the set operations, `MIN`, `MAX` and joins, as the optional features T879, T880 and T881 (§4.8.2, §9.11, §9.12, §9.14). How PartiQL compares JSON values is under [Unresolved questions](#unresolved-questions). A JSON value and a value of another type, such as a `STRING`, are not comparable. To compare JSON with an SQL value, use `JSON_VALUE` or `JSON_SERIALIZE`.

```sql
p.id = 7                                         ->  a compile error  -- a JSON value and an INT are not comparable
JSON_VALUE(p, 'lax $.id' RETURNING INT) = 7      ->  true
```

#### Aggregate Functions

`MIN` and `MAX` on JSON are to be decided, with comparison (see [Equality](#equality)). `COUNT` accepts JSON. This RFC adds `JSON_ARRAYAGG` and `JSON_OBJECTAGG`, which are defined with the other constructors (see [JSON constructors](#json-constructors-json_object-json_array-json_arrayagg-json_objectagg)).

#### Indexing Operator

On a value of type `JSON`, `x[n]` is a step of the simplified accessor and follows its rules (see [JSON simplified accessor](#json-simplified-accessor)). On arrays it is unchanged.

#### Field Access Operator

On a value of type `JSON`, `x.name` and `x."name"` are steps of the simplified accessor and follow its rules (see [JSON simplified accessor](#json-simplified-accessor)). The forms `x['name']` and `x[CAST(e AS STRING)]` are not allowed on JSON. On tuples, all forms are unchanged.

#### Type Checking Functions

Type predicates such as `x IS STRUCT` are false for a JSON value, because a JSON object is JSON, not a tuple. A type predicate for JSON itself is an unresolved question. `IS JSON` is not a type predicate (see [`IS JSON`](#is-json)).

```sql
p.customer IS STRUCT         ->  false   -- a JSON object is not a tuple
p.customer IS JSON OBJECT    ->  true
```

## PartiQL Query Syntax

### `JSON_TABLE`

To be decided. SQL:2023 defines `JSON_TABLE` (§7.11) as a table in `FROM`, which turns JSON into rows and columns, such as `JSON_TABLE(p, 'lax $.items[*]' COLUMNS (sku STRING PATH 'lax $.sku'))`. How it fits PartiQL's `FROM`, `MISSING` and execution modes is under [Unresolved questions](#unresolved-questions).

### Changes to existing sections

#### FROM Clause: Ranging Over Collections

A JSON array in `FROM` gives one row per element. This is a PartiQL extension. SQL:2023 turns a JSON array into rows only through `JSON_TABLE`.

**Rules.**

- Each element has type `JSON`.
- `AT` gives the element's position, counting from 0, as for PartiQL arrays.
- A JSON value that is not an array follows PartiQL's rule for a value that is not a collection. In permissive mode it is treated as a one-element bag, and in type checking mode it is an error.

**Example.**

```sql
SELECT JSON_VALUE(i, 'lax $.sku') AS sku, n
FROM orders AS o, JSON(o.payload).items AS i AT n
-- Result: <<{'sku': 'A1', 'n': 0}, {'sku': 'B2', 'n': 1}>>
-- sku is a STRING, and n is an INT
```

#### WHERE Clause

A JSON value in `WHERE`, `HAVING` or `ON` is not a boolean, so a JSON `true` does not count as true. Use `JSON_VALUE(... RETURNING BOOLEAN)` to test a JSON boolean.

#### GROUP BY Clause and ORDER BY Clause

A JSON value as a grouping key or a sort key is to be decided, with comparison (see [Equality](#equality)).

# Relation to SQL:2023

This RFC follows the JSON support in SQL:2023 Part 2: §4.8 and §4.48 (the type and data model), §6.28 and §6.34 to §6.41 (the operators, constructors, parse, serialize and scalar), §8.22 and §8.23 (`IS JSON` and `JSON_EXISTS`), §9.42 to §9.46 (parsing, and the path language) and §10.11 (the aggregates). Where SQL:2023 defines a behavior, PartiQL uses it.

## Where PartiQL differs

1. **`MISSING`.** SQL:2023 has no `MISSING`. This RFC defines how `MISSING` behaves as an input to each operator.
2. **Permissive mode.** SQL:2023 has only errors. This RFC defines permissive mode for invalid JSON text and wrongly typed input.
3. **Types not known at compile time.** SQL:2023 decides by declared type. PartiQL is gradually typed, so this RFC defines what happens when an expression's type is not known at compile time.
4. **Iteration in `FROM`.** SQL:2023 turns a JSON array into rows only through `JSON_TABLE`. PartiQL also allows a JSON array in `FROM`.
5. **Ion values.** SQL:2023 has no Ion type. This RFC allows a CAST from an Ion value to JSON.
6. **Steps after a function call.** SQL:2023 starts a simplified accessor only at a value expression primary, so `JSON(o.payload).customer` needs parentheses there. PartiQL's field access and index operators already take any expression before the step, so no parentheses are needed.
7. **Keywords.** SQL:2023 reserves `JSON`, the JSON operator names, `ABSENT`, `EMPTY` and `OMIT`. PartiQL keeps them non-reserved, so existing names keep working.

# Drawbacks

- JSON and tuples now answer some paths differently. A missing key is `NULL` on JSON and `MISSING` on a tuple, and unquoted keys match case only on JSON. Users who move data between the two must learn both.
- Keeping duplicate keys and returning the first differs from engines that return the last, such as PostgreSQL `json`, or that reject duplicates by default, such as Spark. A query translated to those engines can give a different answer for rows with duplicate keys.
- The constructors return a `STRING` by default, as SQL:2023 says. Users of PostgreSQL and DuckDB, which return JSON, may expect JSON.

# Rationale and alternatives

- **A separate `JSON` type, not reusing the Ion type.** Ion has types JSON lacks, such as timestamps and symbols, and JSON has a null that SQL does not. Changing Ion values to behave like JSON would change existing queries.
- **SQL `NULL` for a missing key, not `MISSING`.** SQL:2023 defines this result, every engine checked returns `NULL`, and rows keep the same columns. `MISSING` would match tuples, but would change a result that SQL defines.
- **Exact case for keys.** SQL:2023 requires it, and every engine checked except Redshift SUPER's default does the same.
- **Keep duplicates and return the first, rather than the last or an error.** SQL:2023 allows each. Keeping them loses no data, and returning the first matches how PartiQL reads a tuple whose order the schema defines. An error would fail a whole query because of one row.
- **No bracket key on JSON.** SQL:2023 has none in the path language or the simplified accessor, and `j."key"` writes any key, including keys with dots or spaces. Adding `j['key']` would be a PartiQL extension with no new capability.
- **`MISSING` for invalid text in permissive mode, rather than always an error.** One bad row should not fail a permissive query. This matches how PartiQL's permissive mode treats other data errors, and type checking mode keeps SQL:2023's error.
- **`IS JSON` as SQL:2023's content test, not a type predicate.** Every engine that has `IS JSON` uses it to test content, and engines that also test types give that test another name.
- **JSON arrays in `FROM`.** It reuses PartiQL's existing iteration, needs no new syntax, and covers the common case without `JSON_TABLE`.

# Prior art

| Topic | [PostgreSQL 17](https://www.postgresql.org/docs/17/functions-json.html) | [Redshift](https://docs.aws.amazon.com/redshift/latest/dg/r_SUPER_type.html) | [Spark 4.3](https://spark.apache.org/docs/latest/sql-ref-datatypes.html) | [Trino 484](https://trino.io/docs/current/functions/json.html) | [DuckDB 2.0](https://duckdb.org/docs/2.0/data/json/overview.html) |
|---|---|---|---|---|---|
| JSON type | `json`, `jsonb` | `SUPER` | `VARIANT` | `json` | `JSON` |
| Missing key | `NULL` | `NULL` | `NULL` | `NULL` | `NULL` |
| Key case | Exact | Ignored by default | Exact | Exact | Exact |
| Duplicate keys, lookup | Last for `json`. `jsonb` keeps only the last | Error by default | Error by default | Returns both | First |
| JSON null | A value | Not documented | A value | A value | A value |
| Lax unwrap of arrays | Yes | No | No | Yes | No |
| `j['key']` | `jsonb` only | Not documented | No | No | Yes |
| Constructor default | `json` | No `JSON_OBJECT` | No `json_object` | String | `JSON` |
| `IS JSON` | Yes | No, but [`CAN_JSON_PARSE`](https://docs.aws.amazon.com/redshift/latest/dg/CAN_JSON_PARSE.html) | No, but [`try_parse_json`](https://spark.apache.org/docs/latest/api/sql/variant-functions/#try_parse_json) | No | No, but [`json_valid`](https://duckdb.org/docs/2.0/data/json/json_functions.html) |
| `JSON_TABLE` | Yes, without `PLAN` | No | Basic form | Yes | No |
| JSON in `=`, `GROUP BY`, `ORDER BY` | No for `json`, yes for `jsonb` | Yes | No | `=` and `GROUP BY` only | Compares text |

Redshift results come from the AWS documentation, and the Spark `IS JSON` row from the Spark documentation. The others were run on the versions shown. Oracle, Snowflake, Databricks, MySQL, BigQuery and SQL Server were checked in their documentation for duplicate keys, and Oracle and Snowflake also for key case. Their results vary in the same way. Oracle has [`IS JSON`](https://docs.oracle.com/en/database/oracle/oracle-database/23/adjsn/conditions-is-json-and-is-not-json.html) as a content test, as SQL:2023 does.

[RFC 9535](https://www.rfc-editor.org/rfc/rfc9535) (JSONPath) is a separate standard from the SQL/JSON path language. It writes `$['name']`, and its dot form is shorthand for that. This RFC follows SQL/JSON, not [RFC 9535](https://www.rfc-editor.org/rfc/rfc9535).

# Unresolved questions

These parts of SQL:2023's JSON support need PartiQL-specific design. The specification will cover them in later RFCs or revisions of this one.

| Feature | SQL:2023 | What needs deciding |
|---|---|---|
| `JSON_TABLE` | §7.11 | How it fits PartiQL's `FROM`, `MISSING` and execution modes |
| Comparison, grouping and ordering of JSON | T879 to T881, §9.11 to §9.14 | Equality and ordering of JSON values, including scalars of different types and objects (IA224, IA225) |
| More steps in the simplified accessor: wildcards `.*` and `[*]`, several subscripts and ranges, `.'name'`, and item methods such as `j.a.number()` | §6.36, T862, T863, IL064 | Each overlaps PartiQL syntax: path wildcards, the tuple step `.'name'`, and function calls. Which to allow, and how they parse |
| `PASSING` SQL `NULL`, `MISSING`, or a tuple, array, bag or Ion value | §9.46, §10.14 | What each becomes inside the path. SQL:2023 casts other types to a string (IV165). The same answer applies to the index `e` in the simplified accessor step `x[e]` |
| `ENCODING` in `FORMAT JSON` | §6.34, §10.14 | Which binary string types PartiQL accepts as JSON input |
| PartiQL tuples and arrays as constructor inputs, such as `JSON_OBJECT('c' : {'a': 1})` | Not in SQL:2023 | Whether to accept them, and how they map to JSON |
| How `CAST(v AS JSON)` maps an Ion value to JSON | Not in SQL:2023 | How each Ion value maps, such as an Ion null, a typed null, a timestamp, a symbol, a blob or an annotated value. Depends on how the specification defines its Ion type |
| A type predicate for JSON, to tell a JSON value from a string that holds JSON text. This only matters when the type is not known at compile time | Not in SQL:2023 | Its name, which must not be `IS JSON`, and how it fits PartiQL's other type predicates |

# Future possibilities

- A binary JSON representation, as PostgreSQL's [`jsonb`](https://www.postgresql.org/docs/17/datatype-json.html), for faster access. This would not change the semantics in this RFC.
- Functions from other engines, such as Redshift's [`JSON_EXTRACT_PATH_TEXT`](https://docs.aws.amazon.com/redshift/latest/dg/JSON_EXTRACT_PATH_TEXT.html), offered as extensions rather than as part of the language.

# Appendix

Reference tables for implementers and conformance tests.

## Choices SQL:2023 leaves to implementations

SQL:2023 leaves some choices to each implementation. An implementation-defined choice must be documented, and its code starts with I. An implementation-dependent choice needs no documentation, and its code starts with U. PartiQL makes these choices:

| Code | Choice | PartiQL |
|---|---|---|
| UA057 | Whether parsing drops members with duplicate keys | Keeps every member |
| UA075 | Which member a path step returns when keys repeat | The first, in text order. A JSON object keeps the order of its text, so the first member is well-defined |
| IA071 | Whether a path step raises an error for duplicate keys | No error |
| ID074, ID101 | The string type that `JSON_VALUE` returns by default, and that `JSON_QUERY` returns for a string input | `STRING` |
| ID098, ID102 | The string type of a constructor or `JSON_SERIALIZE` result | `STRING` |
| ID100 | The string type that a constructor casts a datetime input to | `STRING` |
| IA073 | How a subscript that is not an integer, such as `[1.5]`, is truncated or rounded | To be decided, with the other path language questions |
| IL064 | The maximum length of variable-length strings returned by item methods in the simplified accessor | Unresolved, with item methods in the simplified accessor |
| IA224, IA225 | The order of scalars of different types, and of objects | Unresolved, with comparison |

## Grammar names

Each rule in this RFC corresponds to a rule in SQL:2023:

| Rule here | SQL:2023 | Section |
|---|---|---|
| `json_parse` | `<JSON parse>` | §6.40 |
| `json_serialize` | `<JSON serialize>` | §6.37 |
| `json_scalar` | `<JSON scalar>` | §6.41 |
| `json_predicate` | `<JSON predicate>` | §8.22 |
| `json_value` | `<JSON value function>` | §6.28 |
| `json_query` | `<JSON query>` | §6.35 |
| `json_exists` | `<JSON exists predicate>` | §8.23 |
| `json_path_string` | `<JSON path specification>` | §9.46 |
| `json_object` | `<JSON object constructor>` | §6.34 |
| `json_array` | `<JSON array constructor>` | §6.34 |
| `json_arrayagg` | `<JSON array aggregate constructor>` | §10.11 |
| `json_objectagg` | `<JSON object aggregate constructor>` | §10.11 |
| `json_api_common` | `<JSON API common syntax>` | §10.14 |
| `json_passing` | `<JSON passing clause>` | §10.14 |
| `json_input` | `<JSON input expression>`, `<JSON value expression>` | §6.34, §10.14 |
| `json_null_clause` | `<JSON constructor null clause>` | §6.34 |
| `json_returning` | `<JSON returning clause>`, `<JSON output clause>` | §6.28, §6.34 |
| `json_unique_keys` | `<JSON key uniqueness constraint>` | §6.40, §8.22 |

## Conformance

PartiQL claims these SQL:2023 features once implemented: T801 (JSON data type), T803 (string-based JSON), T811 (basic constructors), T812 (`JSON_OBJECTAGG`), T813 (`JSON_ARRAYAGG` with `ORDER BY`), T814 (colon in `JSON_OBJECT`), T821 (basic query operators) in part, without `JSON_TABLE`, T822 (`IS JSON WITH UNIQUE KEYS`), T823 (`PASSING`), T825 (`ON EMPTY` and `ON ERROR`), T826 (general value expression in `ON EMPTY` and `ON ERROR`), T828 (`JSON_QUERY`), T829 (array wrapper options), T830 (unique keys in constructors), T831 to T837 and T840 (the path language: strict mode, item methods, several subscripts, wildcards, filters, `starts with`, `like_regex`, hex integer literals), T865 to T878 (item methods in path strings), T860, T861 and T864 (simplified accessor). It does not claim the `JSON_TABLE` features (T824, T827, T838), the comparison features (T879, T880, T881), or T862 and T863, which are under [Unresolved questions](#unresolved-questions). Conformance tests for each rule in this RFC are added to the PartiQL conformance test suite.

# References

- ISO/IEC 9075-2:2023, Information technology, Database languages SQL, Part 2: Foundation (SQL/Foundation).
- [RFC 8259](https://www.rfc-editor.org/rfc/rfc8259), The JavaScript Object Notation (JSON) Data Interchange Format.
- [RFC 9535](https://www.rfc-editor.org/rfc/rfc9535), JSONPath: Query Expressions for JSON.
- [PartiQL specification](https://github.com/partiql/partiql-lang): tuple path navigation, execution modes, and the typing model.
- [partiql/partiql-lang#102](https://github.com/partiql/partiql-lang/pull/102), the specification revamp whose chapters this RFC follows.
