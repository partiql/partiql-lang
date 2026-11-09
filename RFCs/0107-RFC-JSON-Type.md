- Start Date: 2026-11-05
- PartiQL Issue: [partiql/partiql-lang#106](https://github.com/partiql/partiql-lang/issues/106)
- RFC PR: [partiql/partiql-lang#107](https://github.com/partiql/partiql-lang/pull/107)

# Summary

This RFC adds a `JSON` type to PartiQL, following the JSON support in ISO/IEC 9075-2:2023 (SQL:2023): parsing and serializing JSON, the query operators `JSON_VALUE`, `JSON_QUERY` and `JSON_EXISTS`, the `IS JSON` predicate, the constructors, and the simplified accessor `j.a.b`. It defines how JSON works with PartiQL's `MISSING`, execution modes and expressions whose type is not known at compile time, and adds one extension: iterating a JSON array in `FROM`. Parts that need more PartiQL-specific design, such as `JSON_TABLE`, are listed under "Unresolved questions".

# Motivation

Much semi-structured data reaches a query engine as JSON text in a string column. PartiQL can navigate tuples, arrays and Ion values, but it has no way to treat a string as JSON. Given a string column `payload` that holds `{"customer": {"name": "Bob"}}`, the path `t.payload.customer` asks a string for a field, and returns `MISSING` in permissive mode or fails in type checking mode. Users fall back to engine-specific functions, which differ in name and behavior across engines.

SQL:2023 already defines JSON support, and several SQL engines implement much of it (see "Prior art"). Following SQL:2023 gives PartiQL:

1. A standard way to parse, read, iterate and build JSON.
2. Behavior that users of other SQL engines already know.
3. A clear mapping when a PartiQL query is translated to another SQL engine.

A type, rather than a set of functions over strings, is what makes this work. It keeps JSON null separate from SQL `NULL`, lets PartiQL's path syntax read JSON, and tells a string apart from the JSON it holds.

This RFC is about a JSON *type* inside the language. It is separate from querying JSON documents as PartiQL data, which PartiQL already does: a JSON object read from a source is a tuple. The JSON type is for JSON that arrives as text, or that a query builds.

## Non-goals

- Changing how tuples, arrays or Ion values behave. Every existing query keeps its result.
- Defining storage or encoding of JSON values. That is up to each implementation.

# Guide-level explanation

Each part below names the chapter of the PartiQL specification it adds to, so that it can move into the specification as written. The chapter names are those of the specification revamp in [partiql/partiql-lang#102](https://github.com/partiql/partiql-lang/pull/102), which is still in review. If its layout changes, these parts move with it.

## Conventions

**Grammar.** Rules use the EBNF style of the PartiQL specification: `=` defines a rule, terminals are in quotes, `[ ]` is optional, `{ }` repeats, `|` separates alternatives, `;` ends a rule, and `(* *)` is a comment. `expression`, `query`, `string_literal` and `data_type` are the specification's existing rules.

**Examples.** `->` means "evaluates to", and `<=>` means "is the same as". Each result says its type: `JSON …` is a JSON value, `STRING '…'` is a string, `INT …` is an integer, `true` and `false` are booleans, and `NULL` is SQL `NULL`. Serialized JSON text may differ from what is shown in whitespace.

**Example data.** Examples use one row of a table `orders`. Its column `payload` has the type `STRING`. Its value is JSON text, which is still a string, not a JSON value:

```sql
orders = << {
  'payload': '{"id": 7, "customer": {"name": "Bob", "Tier": "gold"}, "items": [{"sku": "A1", "price": 10.5}, {"sku": "B2", "price": 4}], "note": null}'
} >>
```

The same text, laid out to read:

```
{"id": 7,
 "customer": {"name": "Bob", "Tier": "gold"},
 "items": [{"sku": "A1", "price": 10.5}, {"sku": "B2", "price": 4}],
 "note": null}
```

`o` is the `orders` table. `o.payload` is the STRING above, and `p` stands for `JSON(o.payload)`, the JSON value parsed from it.

## Typing Model: JSON when the type is not known at compile time

PartiQL is gradually typed. An expression's type may not be known at compile time, for example when there is no schema. Each value still has its type at runtime, and a value that is JSON follows the same rules as an expression declared `JSON`: paths, `FROM`, CAST, comparison, and the input of the query operators. A query gives the same result whether or not the type was known at compile time. An operation that is a compile error for an expression declared `JSON` gives, at runtime, `MISSING` in permissive mode and an error in type checking mode.

## Data Types: JSON Type (new section)

**Syntax.**

```ebnf
data_type = ... | "JSON" ;   (* adds JSON to the existing types *)
```

`JSON` has no parameters (§4.8). It and the other new words in this RFC are non-reserved, so existing names such as a column called `json` still work.

**Values.** A value of type `JSON` is an *SQL/JSON item* (§4.48.3):

- an SQL/JSON scalar: a non-null string, number, boolean or datetime
- SQL/JSON null
- an SQL/JSON array of items
- an SQL/JSON object, whose members each pair a string key with an item

A JSON number is an exact numeric value (§9.42), so `10.5` keeps its exact value. JSON text has no datetime, so parsing never produces one; `JSON_SCALAR` on a datetime does.

**Absent values.** An expression of type `JSON` can also evaluate to SQL `NULL` or to `MISSING`. These are absent values, not JSON values:

| Value | What it is | `IS NULL` | `IS MISSING` |
|---|---|---|---|
| SQL/JSON null, such as `p.note` | A JSON value | false | false |
| SQL `NULL`, such as `p.age` | No value | true | false |
| `MISSING` | No value, PartiQL only | true | true |

**No `MISSING` inside JSON.** `MISSING` is never inside a JSON value, just as it is never an attribute value of a tuple. JSON text allows only three literals, `true`, `false` and `null` (RFC 8259 §3), so text such as `{"note": missing}` is not valid JSON. A key is either present, possibly with the value `null`, or absent:

| In the JSON text | Example | `p.note` | `JSON_EXISTS(p, 'lax $.note')` |
|---|---|---|---|
| The key holds `null` | `{"note": null}` | JSON `null` | true |
| The key is absent | `{"id": 7}` | `NULL` | false |
| The text says `missing` | `{"note": missing}` | No value: `JSON(...)` fails, giving `MISSING` in permissive mode and an error in type checking mode | On the string, false: the text is not valid JSON, and `ON ERROR` defaults to `FALSE` |

## Type Conversions: CAST to and from JSON

| CAST | Result |
|---|---|
| `CAST(j AS JSON)`, where `j` is `JSON` | `j` unchanged (§6.13) |
| `CAST(NULL AS JSON)` | SQL `NULL` of type `JSON` (§6.13) |
| `CAST(v AS JSON)`, where `v` is an Ion value | The JSON value with the same content. An Ion null inside `v` becomes JSON `null`. An Ion value with no JSON form, such as a timestamp, a blob, a symbol or an annotated value, is an error. The exact rules depend on how the specification defines its Ion type |
| Any other CAST into or out of `JSON` | Not allowed: a compile error when the types are known at compile time, and otherwise `MISSING` in permissive mode and an error in type checking mode. Use `JSON(...)`, `JSON_SERIALIZE` or `JSON_VALUE ... RETURNING` instead |

## Functions and Operators

### JSON Functions and Operators (new section)

#### JSON parse: `JSON(text)`

`JSON(text)` turns a string that holds JSON text into a JSON value (§6.40). SQL:2023 does not allow a CAST from a string to JSON (§6.13), so parsing is its own operator.

**Syntax.**

```ebnf
json_parse = "JSON" "(" expression [ json_unique_keys ] ")" ;
json_unique_keys = ? see "Unresolved questions" ? ;
```

**Rules.**

- The argument must be a string. Its text must be a JSON text (RFC 8259), parsed as §9.42 describes.
- Parsing keeps every member, including members with duplicate keys.
- If the text is not valid JSON, the result is `MISSING` in permissive mode and an error in type checking mode. SQL:2023 raises an error here, as it does for a failed CAST such as `CAST('abc' AS INT)`. PartiQL's permissive mode turns that kind of data error into `MISSING`.
- A SQL `NULL` argument gives SQL `NULL`, and a `MISSING` argument gives `MISSING`.

**Examples.**

```
JSON(o.payload)               ->  JSON {"id": 7, "customer": {...}, "items": [...], "note": null}
JSON('"abc"')                 ->  JSON "abc", a JSON string
JSON('abc')                   ->  MISSING in permissive mode, an error in type checking mode: abc is not JSON
JSON('{"a": 1, "a": 2}').a    ->  JSON 1: both members are kept, and the first is returned (see "JSON simplified accessor")
```

#### `JSON_SERIALIZE` and `JSON_SCALAR`

**Syntax.**

```ebnf
json_serialize = "JSON_SERIALIZE" "(" expression [ json_returning ] ")" ;
json_scalar    = "JSON_SCALAR" "(" expression ")" ;
json_returning = "RETURNING" data_type ;
```

**Rules.**

- `JSON_SERIALIZE(j)` returns the JSON text of a JSON value as a `STRING` (§6.37). Parsing the result gives an equivalent value.
- `JSON_SCALAR(x)` turns a string, number, boolean or datetime into a JSON scalar (§6.41).
- Both return SQL `NULL` for a SQL `NULL` argument, and `MISSING` for a `MISSING` argument.

**Examples.**

```
JSON_SERIALIZE(p.customer)    ->  STRING '{"name":"Bob","Tier":"gold"}'
JSON_SCALAR(10.5)             ->  JSON 10.5
JSON_SCALAR('Bob')            ->  JSON "Bob"
```

#### `IS JSON`

`IS JSON` is SQL:2023's predicate (§8.22). It tests content, not type: does `x` hold valid JSON of the given kind? It is how a query filters out bad rows before it parses them.

**Syntax.**

```ebnf
json_predicate = expression "IS" [ "NOT" ] "JSON"
                 [ "VALUE" | "ARRAY" | "OBJECT" | "SCALAR" ]
                 [ json_unique_keys ] ;
```

**Rules.**

The optional keyword says what kind of JSON value must be at the top level. With no keyword, it is `VALUE`:

| Predicate | True when `x` holds | Example that is true |
|---|---|---|
| `x IS JSON`, `x IS JSON VALUE` | Any valid JSON | `'{"a": 1}'`, `'[1, 2]'`, `'7'` |
| `x IS JSON OBJECT` | A JSON object | `'{"a": 1}'` |
| `x IS JSON ARRAY` | A JSON array | `'[1, 2]'` |
| `x IS JSON SCALAR` | A string, number, boolean or `null` | `'7'`, `'"abc"'`, `'null'` |

`x IS NOT JSON …` is the negation.

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

```
o.payload IS JSON             ->  true
o.payload IS JSON OBJECT      ->  true
o.payload IS JSON ARRAY       ->  false
'abc' IS JSON                 ->  false
JSON('[1, 2]') IS JSON ARRAY  ->  true
```

#### JSON query operators: `JSON_VALUE`, `JSON_QUERY`, `JSON_EXISTS`

Each query operator takes an input, which is a string that holds JSON or a JSON value, and a path string.

**Path strings.** A path string, such as `'lax $.customer.name'`, uses the SQL/JSON path language of SQL:2023 (§9.45, §9.46). It starts with its mode, `lax` or `strict`, then `$` for the whole value, then one step per level: `.key` for a member and `[n]` for an array element, counting from 0. The PartiQL specification does not have this language yet. This RFC adopts `$`, member steps and single array steps by reference, without defining them again. The rest of the path language is listed under "Unresolved questions". This RFC does not change PartiQL's own path expressions such as `t.a` and `t['a']`. PartiQL's execution mode does not change how a path string behaves; the path's mode and the `ON EMPTY` and `ON ERROR` clauses decide.

**Syntax.**

```ebnf
json_value           = "JSON_VALUE" "(" expression "," json_path_string
                       [ json_returning ]
                       [ json_value_behavior "ON" "EMPTY" ]
                       [ json_value_behavior "ON" "ERROR" ] ")" ;
json_value_behavior  = "ERROR" | "NULL" | "DEFAULT" expression ;

json_query           = "JSON_QUERY" "(" expression "," json_path_string
                       [ json_returning ]
                       [ json_wrapper "WRAPPER" ]
                       [ ( "KEEP" | "OMIT" ) "QUOTES" [ "ON" "SCALAR" "STRING" ] ]
                       [ json_query_behavior "ON" "EMPTY" ]
                       [ json_query_behavior "ON" "ERROR" ] ")" ;
json_wrapper         = "WITHOUT" [ "ARRAY" ]
                     | "WITH" [ "CONDITIONAL" | "UNCONDITIONAL" ] [ "ARRAY" ] ;
json_query_behavior  = "ERROR" | "NULL" | "EMPTY" "ARRAY" | "EMPTY" "OBJECT" ;

json_exists          = "JSON_EXISTS" "(" expression "," json_path_string
                       [ json_exists_behavior "ON" "ERROR" ] ")" ;
json_exists_behavior = "TRUE" | "FALSE" | "UNKNOWN" | "ERROR" ;

json_path_string     = string_literal ;   (* a path in the SQL/JSON path language *)
```

**Rules for all three.**

- The operators follow SQL:2023 (§6.28, §6.35, §8.23).
- A SQL `NULL` input gives SQL `NULL`. A `MISSING` input gives `MISSING`.
- An input that is neither a string nor JSON is a compile error when its type is known at compile time, and otherwise `MISSING` in permissive mode and an error in type checking mode.
- In lax mode, a key that does not exist is not an error: the path finds nothing, so `ON EMPTY` applies. Other errors, such as an input string that is not valid JSON, or a value that cannot convert to the `RETURNING` type, go to `ON ERROR`.

**Defaults, from SQL:2023.**

| Clause | Default |
|---|---|
| `RETURNING` in `JSON_VALUE` | `STRING` |
| `RETURNING` in `JSON_QUERY` | `JSON` if the input is `JSON`, and `STRING` if the input is a string |
| `ON EMPTY`, `ON ERROR` in `JSON_VALUE` and `JSON_QUERY` | `NULL` |
| `ON ERROR` in `JSON_EXISTS` | `FALSE` |
| Wrapper in `JSON_QUERY` | `WITHOUT ARRAY WRAPPER`. `WITH WRAPPER` alone means `WITH UNCONDITIONAL ARRAY WRAPPER` |
| Quotes in `JSON_QUERY` | `KEEP QUOTES`. A wrapper clause and a quotes clause cannot both be given |

**`JSON_VALUE`** returns one SQL scalar. SQL/JSON null comes back as SQL `NULL`. If the path finds an object, an array or more than one value, that is an error, which `ON ERROR` handles.

**`JSON_QUERY`** returns a JSON value, or its text as a `STRING`. Without a wrapper, more than one result is an error. `WITH CONDITIONAL ARRAY WRAPPER` wraps the results in an array only when there is more than one.

**`JSON_EXISTS`** returns true if the path finds at least one value, and false if it finds none. A key that holds JSON `null` still counts as found.

**Examples.**

```
JSON_VALUE(o.payload, 'lax $.customer.name')             ->  STRING 'Bob'
JSON_VALUE(o.payload, 'lax $.id' RETURNING INT)          ->  INT 7
JSON_VALUE(o.payload, 'lax $.customer')                  ->  NULL: the path finds an object
JSON_QUERY(p, 'lax $.customer')                          ->  JSON {"name":"Bob","Tier":"gold"}
JSON_QUERY(o.payload, 'lax $.customer')                  ->  STRING '{"name":"Bob","Tier":"gold"}': the input is a STRING
JSON_QUERY(p, 'lax $.items.price' WITH CONDITIONAL ARRAY WRAPPER)  ->  JSON [10.5, 4]
JSON_EXISTS(o.payload, 'lax $.note')                     ->  true: the key exists, even though its value is JSON null
JSON_EXISTS(o.payload, 'lax $.age')                      ->  false: the key does not exist
JSON_EXISTS(o.payload, 'strict $.age')                   ->  false: strict mode raises an error, and ON ERROR defaults to FALSE
```

#### JSON simplified accessor

The simplified accessor reads a JSON value with PartiQL's existing path steps: `.name`, `."name"` and `[n]`. It adds no new syntax. A chain of steps after an expression of type `JSON` is a simplified accessor (§6.36), and it is shorthand for `JSON_QUERY`:

```
p.customer.name
<=> JSON_QUERY(p, 'lax $.customer.name' WITH CONDITIONAL ARRAY WRAPPER NULL ON EMPTY NULL ON ERROR)
->  JSON "Bob"
```

**Rules.**

- It always uses lax mode. A member step on an array runs on each element, and an array step on a value that is not an array treats it as a one-element array.
- Keys match exact case. Arrays count from 0.
- A key that does not exist gives SQL `NULL`, not `MISSING`, in both execution modes. PartiQL uses `MISSING` only where SQL would reject a query, and here SQL:2023 defines the result.
- One result is returned as it is. Several results are wrapped in a JSON array. No result gives SQL `NULL`.
- When keys repeat in an object, a step returns the first member with that key, in text order.
- The result is a JSON value, even when it is a JSON string. To get an SQL value, use `JSON_VALUE`.
- `x['name']` is not allowed on a JSON value, because neither the simplified accessor nor the path language has a bracket key. Write `x."name"`, which also covers keys with dots or spaces. The PartiQL shorthand `x['name']` for `x.'name'` stays a rule for tuples only. When the type is not known at compile time, `x['name']` on a value that is JSON gives `MISSING` in permissive mode and an error in type checking mode.
- `x[e]`, where `e` is an expression that is not a string literal, is an array step whose index is the value of `e`. If `e` is not a number, the step is an error (§9.46), so the result is SQL `NULL`.
- The simplified accessor applies only to an expression of type `JSON`. A string is not parsed: `o.payload.customer` keeps today's tuple path behavior, and users write `JSON(o.payload).customer`.
- Name resolution runs first, with today's rules. In `JSON(o.payload).customer`, `o.payload` resolves as today, and the steps after `JSON(...)` follow this section.

**Examples.**

```
p.customer.name     ->  JSON "Bob"
p.customer.tier     ->  NULL: keys match exact case, and the key is "Tier"
p.customer."Tier"   ->  JSON "gold"
p.items[0].sku      ->  JSON "A1"
p.items.price       ->  JSON [10.5, 4]: lax mode runs .price on each element
p.note              ->  JSON null, which is a value
p.age               ->  NULL: the key does not exist
p.customer['Tier']  ->  a compile error: write p.customer."Tier"
```

`p.customer.name` is JSON `"Bob"`, not the STRING `'Bob'`. So `p.customer.name = 'Bob'` is not allowed (see "Equality, grouping and ordering"), and `JSON_VALUE(p, 'lax $.customer.name') = 'Bob'` is.

**How this differs from tuples.** The same path syntax works on tuples and on JSON:

| What happens | Tuples and Ion values (unchanged) | JSON |
|---|---|---|
| The key does not exist | `MISSING`, or an error in type checking mode | SQL `NULL`, in both modes |
| Unquoted `.name` | Matches any case | Matches exact case |
| A member step reaches an array | `MISSING`, or an error | The step runs on each element |
| `x['name']` | Shorthand for `x.'name'` | Not allowed. Write `x."name"` |
| How many results | Exactly one | Zero or more. Several are wrapped in an array |
| `=`, `GROUP BY`, `ORDER BY` | Allowed | Not allowed yet (see "Equality, grouping and ordering") |
| `x IS STRUCT` | True for a tuple, such as `{'a': 1}` | False: a JSON object is JSON, not a tuple. Use `x IS JSON OBJECT` |

#### JSON constructors: `JSON_OBJECT`, `JSON_ARRAY`, `JSON_ARRAYAGG`

**Syntax.**

```ebnf
json_object      = "JSON_OBJECT" "("
                   [ json_member { "," json_member } [ json_null_clause ] [ json_unique_keys ] ]
                   [ json_returning ] ")" ;
json_member      = [ "KEY" ] expression "VALUE" expression
                 | expression ":" expression ;
json_array       = "JSON_ARRAY" "(" [ expression { "," expression } [ json_null_clause ] ]
                   [ json_returning ] ")"
                 | "JSON_ARRAY" "(" query [ json_null_clause ] [ json_returning ] ")" ;
json_arrayagg    = "JSON_ARRAYAGG" "(" expression [ json_null_clause ] [ json_returning ] ")" ;
json_null_clause = "NULL" "ON" "NULL" | "ABSENT" "ON" "NULL" ;
```

**Rules.** The constructors follow SQL:2023 (§6.34, §10.11).

- *Result type.* With no `RETURNING` clause, the result is `JSON` if any input value has the declared type `JSON`, and a `STRING` otherwise. Only values count, not keys. `JSON_ARRAY` over a query has no input values, so it returns a `STRING`. An input whose type is not known at compile time is not declared `JSON`, so it gives a `STRING` even if its value is JSON. `RETURNING JSON` gives `JSON`.
- *`NULL` values.* `JSON_OBJECT` defaults to `NULL ON NULL`, so a `NULL` value becomes JSON `null`. `JSON_ARRAY` and `JSON_ARRAYAGG` default to `ABSENT ON NULL`, so `NULL` values are left out.
- *`MISSING` values.* A `MISSING` value leaves its member or element out, whichever null clause is used, as a `MISSING` attribute is left out of a tuple.
- *Keys.* A `NULL` key is an error.
- *Inputs.* A JSON input is nested as JSON. A string input becomes a JSON string; it is not parsed.
- *Order.* `JSON_ARRAYAGG` puts the rows of a group in an array, in no defined order.

**Examples.**

```
JSON_OBJECT('id' : 7, 'note' : NULL)      ->  STRING '{"id":7,"note":null}'
JSON_OBJECT('id' : 7 RETURNING JSON)      ->  JSON {"id":7}
JSON_OBJECT('c' : p.customer)             ->  JSON {"c":{"name":"Bob","Tier":"gold"}}: an input value is JSON
JSON_OBJECT('id' : 7, 'note' : MISSING)   ->  STRING '{"id":7}': a MISSING value leaves its member out
JSON_ARRAY(1, NULL, 'x')                  ->  STRING '[1,"x"]': NULL is left out by default
```

```sql
SELECT JSON_ARRAYAGG(JSON_VALUE(i, 'lax $.sku')) AS skus
FROM orders AS o, JSON(o.payload).items AS i
-- <<{'skus': '["A1","B2"]'}>>: skus is a STRING, with the elements in some order
```

### Changes to existing sections

#### Type Mismatch

A function that is not defined for `JSON` treats a JSON argument as a wrongly typed input: a compile error when the type is known at compile time, and otherwise `MISSING` in permissive mode and an error in type checking mode. `IS NULL`, `IS MISSING` and `COUNT` accept JSON.

A JSON value in `WHERE`, `HAVING` or `ON` is not a boolean, so a JSON `true` does not count as true.

#### Equality, grouping and ordering

A JSON value is not allowed in `=`, `<`, `IN`, `BETWEEN`, `DISTINCT`, `GROUP BY`, `ORDER BY`, the set operations, `MIN`, `MAX` or a join condition, and neither is a value that contains JSON. This is what SQL:2023 specifies for an implementation without features T879, T880 and T881. It is a compile error when the type is known at compile time, and an error in both modes otherwise. To compare, use `JSON_VALUE` results or `JSON_SERIALIZE` text. How to compare JSON values is an unresolved question.

#### Result types of `CASE`, `COALESCE`, `NULLIF` and set operations

In `CASE`, `COALESCE`, `NULLIF` and the set operations, if any input is `JSON`, every input must be `JSON` (§9.5). `NULL` and `MISSING` literals are allowed. PartiQL's array, bag and tuple constructors still accept mixed values.

#### Type Checking Functions

Type predicates such as `x IS STRUCT` are false for a JSON value: a JSON object is JSON, not a tuple. A type predicate for JSON itself is an unresolved question. `IS JSON` is not a type predicate (see "`IS JSON`").

## PartiQL Query Syntax: FROM Clause, ranging over a JSON array

A JSON array in `FROM` gives one row per element. This is a PartiQL extension: SQL:2023 turns a JSON array into rows only through `JSON_TABLE`.

**Rules.**

- Each element has type `JSON`.
- `AT` gives the element's position, counting from 0, as for PartiQL arrays.
- A JSON value that is not an array follows PartiQL's rule for a value that is not a collection: in permissive mode it is treated as a one-element bag, and in type checking mode it is an error.

**Example.**

```sql
SELECT JSON_VALUE(i, 'lax $.sku') AS sku, n
FROM orders AS o, JSON(o.payload).items AS i AT n
-- <<{'sku': 'A1', 'n': 0}, {'sku': 'B2', 'n': 1}>>, where sku is a STRING and n an INT
```

# Reference-level explanation

This section relates the RFC to SQL:2023. The RFC follows the JSON support in SQL:2023 Part 2: §4.8 and §4.48 (the type and data model), §6.28 and §6.34 to §6.41 (the operators, constructors, parse, serialize and scalar), §8.22 and §8.23 (`IS JSON` and `JSON_EXISTS`), §9.42 to §9.46 (parsing, and the path language) and §10.11 (the aggregates). Where SQL:2023 defines a behavior, PartiQL uses it.

## Where PartiQL differs

1. **`MISSING`.** SQL:2023 has no `MISSING`. This RFC defines how `MISSING` behaves as an input to each operator.
2. **Permissive mode.** SQL:2023 has only errors. This RFC defines permissive mode for invalid JSON text and wrongly typed input.
3. **Types not known at compile time.** SQL:2023 decides by declared type. PartiQL is gradually typed, so this RFC defines what happens when an expression's type is not known at compile time.
4. **Iteration in `FROM`.** SQL:2023 turns a JSON array into rows only through `JSON_TABLE`. PartiQL also allows a JSON array in `FROM`.
5. **Ion values.** SQL:2023 has no Ion type. This RFC allows a CAST from an Ion value to JSON.

## Choices SQL:2023 leaves to implementations

SQL:2023 marks some choices as implementation-defined (the implementation must document its choice; the code starts with I) or implementation-dependent (no documentation needed; the code starts with U). PartiQL makes these choices:

| Code | Choice | PartiQL |
|---|---|---|
| UA057 | Whether parsing drops members with duplicate keys | Keeps every member |
| UA075 | Which member a path step returns when keys repeat | The first, in text order. This matches the specification's rule that `t.a` returns the first attribute `a` |
| IA071 | Whether a path step raises an error for duplicate keys | No error |
| ID074, ID101 | The string type that `JSON_VALUE` returns by default, and that `JSON_QUERY` returns for a string input | `STRING` |
| ID098, ID102 | The string type of a constructor or `JSON_SERIALIZE` result | `STRING` |
| IL064 | The maximum length of variable-length strings returned by item methods | Unresolved, with item methods |
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
| `json_null_clause` | `<JSON constructor null clause>` | §6.34 |
| `json_returning` | `<JSON returning clause>`, `<JSON output clause>` | §6.28, §6.34 |
| `json_unique_keys` | `<JSON key uniqueness constraint>` | §6.40, §8.22 |

## Conformance

PartiQL claims these SQL:2023 features once implemented: T801 (JSON data type), T803 (string-based JSON), T811 (basic constructors), T814 (colon in `JSON_OBJECT`), T821 (basic query operators) in part, without `JSON_TABLE`, T825 (`ON EMPTY` and `ON ERROR`), T828 (`JSON_QUERY`), T860 and T864 (simplified accessor). Conformance tests for each rule in this RFC are added to the PartiQL conformance test suite.

# Drawbacks

- JSON and tuples now answer some paths differently: a missing key is `NULL` on JSON and `MISSING` on a tuple, and unquoted keys match case only on JSON. Users who move data between the two must learn both.
- Keeping duplicate keys and returning the first differs from engines that return the last, such as PostgreSQL `json`, or that reject duplicates by default, such as Spark. A query translated to those engines can give a different answer for rows with duplicate keys.
- The constructors return a `STRING` by default, as SQL:2023 says. Users of PostgreSQL and DuckDB, which return JSON, may expect JSON.

# Rationale and alternatives

- **A separate `JSON` type, not reusing the Ion type.** Ion has types JSON lacks, such as timestamps and symbols, and JSON has a null that SQL does not. Changing Ion values to behave like JSON would change existing queries.
- **SQL `NULL` for a missing key, not `MISSING`.** SQL:2023 defines this result, every engine checked returns `NULL`, and rows keep the same columns. `MISSING` would match tuples, but would change a result that SQL defines.
- **Exact case for keys.** SQL:2023 requires it, and every engine checked except Redshift SUPER's default does the same.
- **Keep duplicates and return the first, rather than the last or an error.** SQL:2023 allows each. Keeping them loses no data and matches how PartiQL tuples already behave. An error would fail a whole query because of one row.
- **No bracket key on JSON.** SQL:2023 has none in the path language or the simplified accessor, and `j."key"` writes any key, including keys with dots or spaces. Adding `j['key']` would be a PartiQL extension with no new capability.
- **`MISSING` for invalid text in permissive mode, rather than always an error.** One bad row should not fail a permissive query. This matches how PartiQL's permissive mode treats other data errors, and type checking mode keeps SQL:2023's error.
- **`IS JSON` as SQL:2023's content test, not a type predicate.** Every engine that has `IS JSON` uses it to test content, and engines that also test types give that test another name.
- **JSON arrays in `FROM`.** It reuses PartiQL's existing iteration, needs no new syntax, and covers the common case without `JSON_TABLE`.

# Prior art

| Topic | PostgreSQL 17 | Redshift | Spark 4.3 | Trino 484 | DuckDB 2.0 |
|---|---|---|---|---|---|
| JSON type | `json`, `jsonb` | `SUPER` | `VARIANT` | `json` | `JSON` |
| Missing key | `NULL` | `NULL` | `NULL` | `NULL` | `NULL` |
| Key case | Exact | Ignored by default | Exact | Exact | Exact |
| Duplicate keys, lookup | `json`: last. `jsonb`: keeps the last | Error by default | Error by default | Returns both | First |
| JSON null | A value | Not documented | A value | A value | A value |
| Lax unwrap of arrays | Yes | No | No | Yes | No |
| `j['key']` | `jsonb` only | Not documented | No | No | Yes |
| Constructor default | `json` | No `JSON_OBJECT` | No `json_object` | String | `JSON` |
| `IS JSON` | Yes | No; `CAN_JSON_PARSE` | No; `try_parse_json` | No | No; `json_valid` |
| `JSON_TABLE` | Yes, without `PLAN` | No | Basic form | Yes | No |
| JSON in `=`, `GROUP BY`, `ORDER BY` | `json`: no. `jsonb`: yes | Yes | No | `=` and `GROUP BY` only | Compares text |

Redshift results come from the AWS documentation, and the Spark `IS JSON` row from the Spark documentation. The others were run on the versions shown. Oracle, Snowflake, Databricks, MySQL, BigQuery and SQL Server were checked in their documentation for duplicate keys, and Oracle and Snowflake also for key case. Their results vary in the same way. Oracle has `IS JSON` as a content test, as SQL:2023 does.

RFC 9535 (JSONPath) is a separate standard from the SQL/JSON path language. It writes `$['name']`, and its dot form is shorthand for that. This RFC follows SQL/JSON, not RFC 9535.

# Unresolved questions

These parts of SQL:2023's JSON support need PartiQL-specific design. The specification will cover them in later RFCs or revisions of this one.

| Feature | SQL:2023 | What needs deciding |
|---|---|---|
| `JSON_TABLE` | §7.11 | How it fits PartiQL's `FROM`, `MISSING` and execution modes |
| Comparison, grouping and ordering of JSON | T879 to T881, §9.11 to §9.14 | Equality and ordering of JSON values, including scalars of different types and objects (IA224, IA225) |
| Item methods, such as `j.a.number()` | §6.36, §9.46 | How `t.j.a.number()` relates to PartiQL's qualified function calls |
| Path string features beyond `$`, member steps and array steps: filters, wildcards, several subscripts, ranges, `last`, arithmetic and `PASSING` variables | §9.46, §10.14 | How they interact with PartiQL values and `MISSING` |
| `WITH UNIQUE KEYS` (`json_unique_keys`) | §6.40, §8.22 | Its interaction with keeping duplicate keys |
| `JSON_OBJECTAGG`, and `JSON_ARRAYAGG` with `ORDER BY` | §10.11 | Depends on comparison and ordering |
| PartiQL tuples and arrays as constructor inputs, such as `JSON_OBJECT('c' : {'a': 1})` | Not in SQL:2023 | Whether to accept them, and how they map to JSON |
| CAST from JSON to an Ion value | Not in SQL:2023 | Whether to allow it, once the specification defines its Ion type |
| `x.'name'` on JSON | T863 | Whether to support the single-quoted member step |
| A type predicate for JSON, testing whether a value has type `JSON` | Not in SQL:2023 | Its name, which must not be `IS JSON`, and how it fits PartiQL's other type predicates |
| `IS JSON` on an expression whose type is not known at compile time, when the value is neither a string nor JSON | Not in SQL:2023 | Whether the result is false, or follows PartiQL's rule for wrongly typed input |

# Future possibilities

- A binary JSON representation, as PostgreSQL's `jsonb`, for faster access. This would not change the semantics in this RFC.
- Functions from other engines, such as Redshift's `JSON_EXTRACT_PATH_TEXT`, offered as extensions rather than as part of the language.

# References

- ISO/IEC 9075-2:2023, Information technology, Database languages SQL, Part 2: Foundation (SQL/Foundation).
- RFC 8259, The JavaScript Object Notation (JSON) Data Interchange Format.
- RFC 9535, JSONPath: Query Expressions for JSON.
- PartiQL specification: tuple path navigation, execution modes, and the typing model.
