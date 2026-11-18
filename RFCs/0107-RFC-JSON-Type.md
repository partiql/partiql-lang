- Start Date: 2026-11-05
- PartiQL Issue: [partiql/partiql-lang#106](https://github.com/partiql/partiql-lang/issues/106)
- RFC PR: [partiql/partiql-lang#107](https://github.com/partiql/partiql-lang/pull/107)

# Summary

This RFC adds a `JSON` type to PartiQL. It implements SQL:2023's `JSON` type (feature T801, ISO/IEC 9075-2:2023) and follows SQL:2023 wherever SQL:2023 decides a behavior. Where PartiQL has its own rules, such as `MISSING`, its two execution modes and types not known at compile time, PartiQL's rules apply.

# Motivation

Much semi-structured data reaches a query engine as JSON text in a string column. PartiQL can navigate tuples, arrays and Ion values, but it has no way to treat a string as JSON. Given a string column `payload` that holds `{"customer": {"name": "Bob"}}`, the path `t.payload.customer` asks a string for a field, and returns `MISSING` in permissive mode or fails in type checking mode. Users fall back to engine-specific functions, which differ in name and behavior across engines.

SQL:2023 defines JSON as a type, and this RFC does the same. Following SQL:2023 gives PartiQL a standard way to parse, read and build JSON, behavior that users of other SQL engines already know, and a clear mapping when a PartiQL query is translated to another SQL engine.

## Non-goals

- Changing how tuples, arrays or Ion values behave. Every existing query keeps its result.
- Defining how values of type `JSON` are stored or encoded. That is up to each implementation.

# Specification

## Conventions

**Grammar.** Rules use the EBNF style of the PartiQL specification: `=` defines a rule, terminals are in quotes, `[ ]` is optional, `{ }` repeats, `|` separates alternatives, and `;` ends a rule.

**Terms.** This RFC uses SQL:2023's terms, which keep text and parsed values apart:

- *JSON text* is a string that conforms to [RFC 8259](https://www.rfc-editor.org/rfc/rfc8259) (§3.5.30), such as the `STRING` `'{"a": 1}'`. A *JSON object*, *JSON array*, *JSON string* or *JSON null* is a piece of JSON text.
- An *SQL/JSON item* is what JSON text becomes when it is parsed (§4.48.3). Its kinds are an *SQL/JSON scalar*, *SQL/JSON null*, an *SQL/JSON array* and an *SQL/JSON object*.
- A value of type `JSON` is an SQL/JSON item. An expression of type `JSON` evaluates to an SQL/JSON item, SQL `NULL` or `MISSING`.

**Examples.** `->` means "evaluates to", and `--` starts a comment. Each result starts with its type. `JSON {"c":1}` is a value of type `JSON`, shown as the JSON text it serializes to. `STRING '{"c":1}'` is a string that holds JSON text. `NULL` is SQL `NULL`. Serialized JSON text is shown with members in the order of the input text. The exact text is not specified yet.

**Example data.** Examples use one row of a table `orders`. Its column `payload` has the type `STRING`, and its value is JSON text:

```sql
orders = << {
  'payload': '{"id": 7, "customer": {"name": "Bob", "Tier": "gold"}, "items": [{"sku": "A1", "price": 10.5}, {"sku": "B2", "price": 4}], "note": null}'
} >>
```

`o` is the `orders` table. `o.payload` is JSON text, and `p` stands for `JSON(o.payload)`, the SQL/JSON item parsed from it, whose type is `JSON`.

## The JSON type

The type name is `JSON`, and it has no parameters (§4.8). A query cannot write a value of type `JSON` directly, the way it writes `7` or `DATE '2024-01-01'` (§5.3). A string such as `'{"a": 1}'` is a `STRING`. A value of type `JSON` comes from `JSON(text)`, `JSON_SCALAR`, a constructor, `JSON_QUERY`, or a path step on another value of type `JSON`.

`JSON` and the other new words in this RFC are non-reserved, so existing names such as a column called `json` still work.

**Values.** A value of type `JSON` is an SQL/JSON item (§4.48.3), one of:

1. A scalar: a non-null string, number, boolean or datetime (`DATE`, `TIME` or `TIMESTAMP`, with or without time zone).
2. SQL/JSON null.
3. An array of items.
4. An object, whose members each pair a string key with an item.

A JSON number in JSON text becomes an SQL/JSON scalar, read the way SQL reads the same digits as a literal (§9.42). Without an exponent, such as `10.5`, it is an exact number, like `DECIMAL`. With an exponent, such as `1e3`, it is an approximate number, a binary floating point like `FLOAT`. PartiQL's literals already follow this rule:

```sql
0.1 + 0.2        ->  0.3                    -- exact
0.1e0 + 0.2e0    ->  0.30000000000000004    -- approximate, rounded in binary
```

JSON text has no datetime, so parsing never produces one. `JSON_SCALAR` on a datetime does.

```sql
JSON('"2024-01-01"')              ->  JSON "2024-01-01"     -- an SQL/JSON scalar that is a string, not a date
JSON_SCALAR(DATE '2024-01-01')    ->  an SQL/JSON scalar that is DATE '2024-01-01'
```

**Absent values.** An expression of type `JSON` can also evaluate to SQL `NULL` or to `MISSING`. These are absent values, not SQL/JSON items:

| Value                                               | What it is             | `IS NULL` | `IS MISSING` |
| --------------------------------------------------- | ---------------------- | --------- | ------------ |
| SQL/JSON null, such as `p.note`                     | An SQL/JSON item       | false     | false        |
| SQL `NULL`, such as `p.age`                         | No value               | true      | false        |
| `MISSING`, such as `JSON('abc')` in permissive mode | No value, PartiQL only | true      | true         |

`MISSING` is never part of an SQL/JSON item, neither as a key nor as a value, as it is never an attribute value of a tuple.

**Mixing with other types.** In `CASE`, `COALESCE`, `NULLIF` and the set operations, if any input has type `JSON`, every input must have type `JSON` (§9.5). `NULL` and `MISSING` literals are allowed. PartiQL's array, bag and tuple constructors still accept a mix, such as `[JSON('1'), 2]`.

```sql
COALESCE(p.note, 'none')                          ->  a compile error  -- p.note has type JSON, 'none' is a STRING
COALESCE(JSON_VALUE(p, 'lax $.note'), 'none')     ->  STRING 'none'
```

## JSON when the type is not known at compile time

The specification's gradual typing rules apply to JSON as to any other type. When an expression's type is not known at compile time, for example when there is no schema, a value of type `JSON` at runtime follows the rules for an expression declared `JSON`. A use that would be a compile error for a declared `JSON` expression gives `MISSING` in permissive mode and an error in type checking mode.

Two rules are specific to JSON:

1. A value of type `JSON` is never converted to a tuple, an array or an SQL scalar to make an operation work. A query reads from it with the JSON operators.

```sql
-- x is the SQL/JSON scalar 5, which is not converted to INT 5
SELECT VALUE x + 1 FROM [JSON('5')] AS x
-- MISSING in permissive mode, an error in type checking mode
```

2. A constructor whose input has no known type also has no known result type. Each row gives a value of type `JSON` when an input value has type `JSON`, and a `STRING` otherwise. When every input type is known, the constructor follows SQL:2023 (§6.34). `JSON_QUERY` decides for each row in the same way, and the aggregates for each group (see [Querying JSON](#querying-json) and [the constructors](#building-arrays-and-objects-the-constructors)).

```sql
-- The type of x is not known at compile time, so each row decides
SELECT VALUE JSON_OBJECT('c' : x) FROM [JSON('{"a": 1}'), 'abc'] AS x
-- << JSON {"c":{"a":1}}, STRING '{"c":"abc"}' >>
```

## Converting to and from JSON

Each kind of SQL/JSON item has its own way in. JSON text can be parsed into any kind:

| SQL/JSON item | Made by                                        | Example                                                   |
| ------------- | ---------------------------------------------- | --------------------------------------------------------- |
| A scalar      | `JSON_SCALAR`, or parsing                      | `JSON_SCALAR(10.5)`, `JSON('10.5')`                       |
| SQL/JSON null | Parsing, or an object constructor given `NULL` | `JSON('null')`, the member in `JSON_OBJECT('a' : NULL)`   |
| An array      | `JSON_ARRAY`, `JSON_ARRAYAGG`, or parsing      | `JSON_ARRAY(1, 2 RETURNING JSON)`, `JSON('[1, 2]')`       |
| An object     | `JSON_OBJECT`, `JSON_OBJECTAGG`, or parsing    | `JSON_OBJECT('a' : 1 RETURNING JSON)`, `JSON('{"a": 1}')` |

`JSON_SCALAR(NULL)` is SQL `NULL`, not SQL/JSON null (§6.41). `JSON_SERIALIZE` goes the other way, from an item to JSON text.

### Parsing JSON text: `JSON(text)`

`JSON(text)` turns a string that holds JSON text into an SQL/JSON item of type `JSON` (§6.40).

```ebnf
json_parse       = "JSON" "(" json_input [ json_unique_keys ] ")" ;
json_input       = expression [ "FORMAT" "JSON" ] ;
json_unique_keys = ( "WITH" | "WITHOUT" ) "UNIQUE" [ "KEYS" ] ;
```

**The input must be a string.** Its text must be JSON text, parsed as §9.42 describes. An input of another declared type is a compile error. An input whose type is not known at compile time, and that is not a string at runtime, gives `MISSING` in permissive mode and an error in type checking mode. `FORMAT JSON` says that a string input holds JSON text. It is the default here, so it changes nothing.

```sql
JSON(o.payload)       ->  JSON {"id": 7, "customer": {...}, "items": [...], "note": null}
JSON('"abc"')         ->  JSON "abc"         -- an SQL/JSON scalar that is a string
JSON(42)              ->  a compile error    -- 42 is an INT, not a string
```

**Invalid text.** If the text is not valid JSON text, the result is `MISSING` in permissive mode and an error (22032) in type checking mode.

```sql
JSON('abc')           ->  MISSING in permissive mode, an error in type checking mode
```

**Numbers.** A JSON number becomes an SQL/JSON scalar of the PartiQL type that a literal written the same way has:

| In JSON text | PartiQL type |
|---|---|
| `7` | `INT` |
| `10.5` | `DECIMAL` |
| `12345678901234567890123` | `DECIMAL` |
| `1e3` | `FLOAT` |

**A number out of range.** A number too large for its type, such as `1e400`, is not accepted. The result is `MISSING` in permissive mode and an error in type checking mode, as for invalid text.

```sql
JSON('{"x": 1e400}')  ->  MISSING in permissive mode, an error in type checking mode
```

**Duplicate keys.** Parsing keeps every member, including members with duplicate keys. This is `WITHOUT UNIQUE KEYS`, the default. A path step on a repeated key returns the first member, in the order of the text, in both modes.

```sql
JSON('{"a": 1, "a": 2}').a    ->  JSON 1
```

**`WITH UNIQUE KEYS`.** Text with a duplicate key in any object is not accepted (§9.42). The result is `MISSING` in permissive mode and an error in type checking mode.

```sql
JSON('{"a": 1, "a": 2}' WITH UNIQUE KEYS)    ->  MISSING in permissive mode, an error in type checking mode
```

**Absent inputs.** SQL `NULL` gives SQL `NULL`, and `MISSING` gives `MISSING`.

### Writing JSON text: `JSON_SERIALIZE`

`JSON_SERIALIZE(j)` returns the JSON text of an SQL/JSON item (§6.37).

```ebnf
json_serialize = "JSON_SERIALIZE" "(" expression [ json_returning ] ")" ;
json_returning = "RETURNING" data_type [ "FORMAT" "JSON" ] ;
```

**The input must have type `JSON`.** A string is not accepted, even one that holds JSON text (§6.38).

```sql
JSON_SERIALIZE(p.customer)     ->  STRING '{"name":"Bob","Tier":"gold"}'
JSON_SERIALIZE('{"a": 1}')     ->  a compile error    -- the input is a STRING
```

**The result.** The result is a `STRING`, or the string type named in `RETURNING` (ID102). `FORMAT JSON` after the type changes nothing, because JSON is the only format. Parsing the result gives an equivalent SQL/JSON item (§9.43). Apart from that, the exact text is not specified yet: member order, whitespace, and how a number is spelled (see [Unresolved questions](#unresolved-questions)).

**A datetime.** An SQL/JSON item that holds a datetime has no JSON text, so serializing it fails (§9.43). The result is `MISSING` in permissive mode and an error in type checking mode. The same applies wherever an SQL/JSON item becomes text.

```sql
JSON_SERIALIZE(JSON_SCALAR(DATE '2024-01-01'))    ->  MISSING in permissive mode, an error in type checking mode
```

**Absent inputs.** SQL `NULL` gives SQL `NULL`, and `MISSING` gives `MISSING`.

### Making a scalar: `JSON_SCALAR`

`JSON_SCALAR(x)` turns an SQL value into an SQL/JSON scalar (§6.41).

```ebnf
json_scalar = "JSON_SCALAR" "(" expression ")" ;
```

**Inputs.** The input must be a string, a number, a boolean or a datetime. Any other declared type, such as an interval, a binary value, a tuple, an array or `JSON`, is a compile error. An input whose type is not known at compile time, and that is none of these at runtime, gives `MISSING` in permissive mode and an error in type checking mode.

```sql
JSON_SCALAR(10.5)                  ->  JSON 10.5
JSON_SCALAR('Bob')                 ->  JSON "Bob"
JSON_SCALAR(DATE '2024-01-01')     ->  an SQL/JSON scalar that is DATE '2024-01-01'
JSON_SCALAR(INTERVAL '1' DAY)      ->  a compile error
```

**Absent inputs.** SQL `NULL` gives SQL `NULL`, not SQL/JSON null (§6.41). `MISSING` gives `MISSING`.

```sql
JSON_SCALAR(NULL)                  ->  NULL
```

### Building arrays and objects: the constructors

`JSON_OBJECT` and `JSON_ARRAY` build an SQL/JSON object or array from values in one row (§6.34). `JSON_OBJECTAGG` and `JSON_ARRAYAGG` build one from the rows of a group (§10.11).

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
```

**Result type.** With no `RETURNING` clause, the result has type `JSON` if any input value has type `JSON`, and is the JSON text as a `STRING` otherwise (§6.34, §10.11). Only values count, not keys. `RETURNING JSON` always gives `JSON`. `JSON_ARRAY` over a query always gives a `STRING`, because a query is not an input value in SQL:2023's grammar.

```sql
JSON_OBJECT('id' : 7)                          ->  STRING '{"id":7}'
JSON_OBJECT('id' : 7 RETURNING JSON)           ->  JSON {"id":7}
JSON_OBJECT('c' : p.customer)                  ->  JSON {"c":{"name":"Bob","Tier":"gold"}}   -- p.customer has type JSON
JSON_ARRAY(SELECT VALUE x FROM <<1, 2>> AS x)  ->  STRING '[1,2]'
```

**When an input's type is not known.** `JSON_OBJECT` and `JSON_ARRAY` decide for each row (see [JSON when the type is not known at compile time](#json-when-the-type-is-not-known-at-compile-time)). `JSON_ARRAYAGG` and `JSON_OBJECTAGG` decide for each group: the result has type `JSON` if any value in the group has type `JSON`, and is a `STRING` otherwise.

```sql
SELECT g, JSON_ARRAYAGG(t.v) AS arr FROM t GROUP BY t.g AS g
-- t has no schema
-- group 1 holds 'a', 'b'          ->  arr is STRING '["a","b"]'
-- group 2 holds JSON('1'), 'c'    ->  arr is JSON [1,"c"]
```

**Inputs.** A value of type `JSON` is nested as it is. A string, number or boolean becomes an SQL/JSON scalar. A datetime is cast to `STRING` and becomes an SQL/JSON string scalar (§6.34). A string is not parsed, unless it is followed by `FORMAT JSON`, or it is the result of a constructor or of `JSON_QUERY`, which implies `FORMAT JSON` (§10.12). Other inputs, such as an interval, a binary value, a tuple, an array, a bag or an Ion value, are not specified yet (see [Unresolved questions](#unresolved-questions)).

```sql
JSON_ARRAY(1, true, 'x')                       ->  STRING '[1,true,"x"]'
JSON_OBJECT('c' : JSON('{"a": 1}'))            ->  JSON {"c":{"a":1}}
JSON_OBJECT('c' : '{"a": 1}')                  ->  STRING '{"c":"{\"a\": 1}"}'     -- a string is not parsed
JSON_OBJECT('c' : '{"a": 1}' FORMAT JSON)      ->  STRING '{"c":{"a":1}}'          -- FORMAT JSON parses it
JSON_OBJECT('c' : JSON_OBJECT('a' : 1))        ->  STRING '{"c":{"a":1}}'          -- the inner result implies FORMAT JSON
JSON_OBJECT('c' : JSON_QUERY('{"a": [1]}', 'lax $.a'))  ->  STRING '{"c":[1]}'     -- so does a JSON_QUERY result
JSON_OBJECT('d' : DATE '2024-01-01')           ->  STRING '{"d":"2024-01-01"}'     -- a datetime becomes a string
JSON_OBJECT('d' : JSON_SCALAR(DATE '2024-01-01'))  ->  JSON, whose member d holds DATE '2024-01-01'
```

**`NULL` values.** `JSON_OBJECT` and `JSON_OBJECTAGG` default to `NULL ON NULL`, so a `NULL` value becomes SQL/JSON null. `JSON_ARRAY` and `JSON_ARRAYAGG` default to `ABSENT ON NULL`, so a `NULL` value is left out (§6.34, §10.11).

```sql
JSON_OBJECT('id' : 7, 'note' : NULL)                  ->  STRING '{"id":7,"note":null}'
JSON_OBJECT('id' : 7, 'note' : NULL ABSENT ON NULL)   ->  STRING '{"id":7}'
JSON_ARRAY(1, NULL, 'x')                              ->  STRING '[1,"x"]'
JSON_ARRAY(1, NULL, 'x' NULL ON NULL)                 ->  STRING '[1,null,"x"]'
```

**`MISSING` values.** A `MISSING` value leaves its member or element out, whichever null clause is used, as a `MISSING` attribute is left out of a tuple.

```sql
JSON_OBJECT('id' : 7, 'note' : MISSING)               ->  STRING '{"id":7}'
JSON_ARRAY(1, MISSING, 'x' NULL ON NULL)              ->  STRING '[1,"x"]'
```

**Keys.** A key must be a string. A `NULL` key is an error (§6.34). A `MISSING` key leaves its member out in permissive mode, and is an error in type checking mode, as for a tuple attribute name.

```sql
JSON_OBJECT(KEY 'id' VALUE 7)                         ->  STRING '{"id":7}'   -- the same as 'id' : 7
JSON_OBJECT(NULL : 7)                                 ->  an error
JSON_OBJECT(t.k : 1, 'b' : 2)                         ->  STRING '{"b":2}' in permissive mode, an error in type checking mode   -- t has no attribute k
```

**Duplicate keys.** By default, `WITHOUT UNIQUE KEYS`, members with the same key are all kept. With `WITH UNIQUE KEYS`, a duplicate key is not accepted: the result is `MISSING` in permissive mode and an error in type checking mode (see the open question under [`WITH UNIQUE KEYS`](#parsing-json-text-jsontext)).

```sql
JSON_OBJECT('a' : 1, 'a' : 2)                         ->  STRING '{"a":1,"a":2}'
JSON_OBJECT('a' : 1, 'a' : 2 WITH UNIQUE KEYS)        ->  MISSING in permissive mode, an error in type checking mode
```

**Aggregates.** Over no rows, `JSON_ARRAYAGG` and `JSON_OBJECTAGG` give SQL `NULL` (§10.11). Without `ORDER BY`, `JSON_ARRAYAGG` puts the rows of a group in no defined order. With `ORDER BY`, the elements follow the sort keys, as in PartiQL's `ORDER BY`. A sort key of type `JSON` waits on comparison (see [Unresolved questions](#unresolved-questions)).

```sql
SELECT JSON_ARRAYAGG(JSON_VALUE(i, 'lax $.sku') ORDER BY JSON_VALUE(i, 'lax $.price' RETURNING DECIMAL)) AS skus,
       JSON_OBJECTAGG(JSON_VALUE(i, 'lax $.sku') : i) AS by_sku
FROM orders AS o, JSON(o.payload).items AS i
-- << {'skus': '["B2","A1"]', 'by_sku': JSON {"A1":{"sku":"A1","price":10.5},"B2":{"sku":"B2","price":4}}} >>
-- skus is a STRING, sorted by price. by_sku is JSON, because its value input i has type JSON
```

### CAST

| CAST | Result |
|---|---|
| `CAST(j AS JSON)`, where `j` has type `JSON` | `j` unchanged (§6.13) |
| `CAST(NULL AS JSON)` | SQL `NULL` of type `JSON` (§6.13) |
| `CAST(v AS JSON)`, where `v` is an Ion value | Allowed. How each Ion value maps to an SQL/JSON item is to be decided (see [Unresolved questions](#unresolved-questions)) |
| CAST from `JSON` to an Ion type | Not allowed |
| Any other CAST into or out of `JSON` | Not allowed. It is a compile error when the types are known at compile time, and otherwise `MISSING` in permissive mode and an error in type checking mode. Use `JSON(...)`, `JSON_SERIALIZE` or `JSON_VALUE ... RETURNING` instead |

## Testing JSON

PartiQL has no `IS JSON` predicate, in any form. SQL:2023's `IS JSON` (§8.22) tests content, and two functions replace it, each named for what it tests:

- `is_valid_json` tests content: whether a string holds valid JSON text, of a given shape.
- `is_json_type` tests type: whether a value has type `JSON`.

### `is_valid_json`

```ebnf
is_valid_json = "IS_VALID_JSON" "(" expression [ "," json_shape [ "," ( "TRUE" | "FALSE" ) ] ] ")" ;
json_shape    = "VALUE" | "OBJECT" | "ARRAY" | "SCALAR" ;
```

`is_valid_json(x)` is true when `x` holds valid JSON text. It has the meaning of SQL:2023's `IS JSON` (§8.22).

**The shape.** The optional second argument says what must be at the top level of the text. It is a keyword, not a string. With no shape, it is `VALUE`:

| Call | True when `x` holds | Example that is true |
|---|---|---|
| `is_valid_json(x)`, `is_valid_json(x, VALUE)` | Any valid JSON text | `'{"a": 1}'`, `'[1, 2]'`, `'7'` |
| `is_valid_json(x, OBJECT)` | A JSON object | `'{"a": 1}'` |
| `is_valid_json(x, ARRAY)` | A JSON array | `'[1, 2]'` |
| `is_valid_json(x, SCALAR)` | A JSON string, number, boolean or null | `'7'`, `'"abc"'`, `'null'` |

```sql
is_valid_json('[1, 2]')               ->  true
is_valid_json('[1, 2]', OBJECT)       ->  false    -- valid, but an array
is_valid_json('null', SCALAR)         ->  true     -- null counts as a scalar here (§8.22)
is_valid_json('abc')                  ->  false    -- not valid JSON text
```

**Unique keys.** The optional third argument is `TRUE` or `FALSE`, and `FALSE` is the default. With `TRUE`, the result is also false when any object in the text, at any depth, has a duplicate key. This is SQL:2023's `IS JSON WITH UNIQUE KEYS` (§8.22).

```sql
is_valid_json('{"a": 1, "a": 2}')                     ->  true
is_valid_json('{"a": 1, "a": 2}', OBJECT, TRUE)       ->  false
is_valid_json('{"a": {"b": 1, "b": 2}}', VALUE, TRUE) ->  false    -- the duplicate is nested
```

**Inputs.** A string is tested as JSON text. A value of type `JSON` is always valid, and the shape and unique keys test the item itself. An input of another declared type, such as `is_valid_json(1)`, is a compile error.

```sql
is_valid_json(JSON('[1, 2]'), ARRAY)   ->  true
is_valid_json(1)                       ->  a compile error
```

**An input whose type is not known.** When the type is not known at compile time and the value is neither a string nor a value of type `JSON` at runtime, the result is false.

```sql
SELECT VALUE is_valid_json(x) FROM [JSON('{}'), '{}', 'abc', 5] AS x
-- << true, true, false, false >>
```

**Absent inputs.** SQL `NULL` gives SQL `NULL`, and `MISSING` gives `MISSING`.

### `is_json_type`

```ebnf
is_json_type = "IS_JSON_TYPE" "(" expression ")" ;
```

`is_json_type(x)` is true when `x` has type `JSON`, including an SQL/JSON null. It follows PartiQL's type predicates, such as `x IS STRING`:

| Input | Result |
|---|---|
| A value of type `JSON` | true |
| A value of any other type, including a string that holds JSON text | false |
| SQL `NULL` | `NULL` |
| `MISSING` | `MISSING` |

```sql
is_json_type(JSON(o.payload))     ->  true
is_json_type(o.payload)           ->  false     -- a STRING that holds JSON text
is_json_type(JSON('null'))        ->  true      -- an SQL/JSON null
```

The result only gives new information when the type of `x` is not known at compile time. Otherwise the compiler already knows it.

## Querying JSON

Three query operators read part of an SQL/JSON item: `JSON_VALUE` returns one SQL value, `JSON_QUERY` returns JSON, and `JSON_EXISTS` tests whether something is there. Each takes an input and a path string.

### Path strings

A path string is written in the SQL/JSON path language, which SQL:2023 defines in §9.45 and §9.46. This RFC adopts that language as SQL:2023 defines it, and does not restate it.

```sql
'lax $.customer.name'
--  lax        the mode, lax or strict
--  $          the whole input
--  .customer  a member step
--  .name      a member step
```

- A path starts with its mode, then `$`, then its steps, such as `.key` for a member and `[n]` for an array element, counting from 0.
- The whole language applies, including wildcards, filters, arithmetic and item methods such as `.date()`.
- A path works only on SQL/JSON items, so `MISSING`, tuples and Ion values never occur inside it.
- PartiQL's own path expressions, such as `t.a` on a tuple, keep their current rules.

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
```

### Rules for all three

**Inputs.** The input is a string that holds JSON text, or a value of type `JSON` (§10.14). A string is parsed first. An input of another declared type is a compile error. An input whose type is not known at compile time, and that is neither at runtime, gives `MISSING` in permissive mode and an error in type checking mode.

```sql
JSON_VALUE(o.payload, 'lax $.id')     ->  STRING '7'    -- a string input is parsed first
JSON_VALUE(p, 'lax $.id')             ->  STRING '7'    -- the same, from a value of type JSON
JSON_VALUE(7, 'lax $')                ->  a compile error
```

**Absent inputs.** SQL `NULL` gives SQL `NULL`, and `MISSING` gives `MISSING`.

**Errors go to `ON ERROR`.** An error inside a query operator is handled by its `ON ERROR` clause, in both execution modes (§9.47). This includes input text that is not valid JSON, a path that finds several items where one is needed, and a value that cannot convert to the `RETURNING` type. The default is `NULL`, and `FALSE` for `JSON_EXISTS`. To get an error, write `ERROR ON ERROR`.

```sql
JSON_VALUE('abc', 'lax $.a')                      ->  NULL       -- in both modes
JSON_VALUE('abc', 'lax $.a' ERROR ON ERROR)       ->  an error   -- in both modes
```

**Lax and strict.** In lax mode, a member that does not exist is not an error: the path finds nothing, and `ON EMPTY` applies. In strict mode, it is an error, and `ON ERROR` applies (§9.46).

**Defaults, from SQL:2023.**

| Clause | Default |
|---|---|
| `RETURNING` in `JSON_VALUE` | `STRING` (ID074) |
| `RETURNING` in `JSON_QUERY` | `JSON` for an input of type `JSON`, and `STRING` for a string input (ID101) |
| `ON EMPTY`, `ON ERROR` in `JSON_VALUE` and `JSON_QUERY` | `NULL` |
| `ON ERROR` in `JSON_EXISTS` | `FALSE` |
| Wrapper in `JSON_QUERY` | `WITHOUT ARRAY WRAPPER`. `WITH WRAPPER` alone means `WITH UNCONDITIONAL ARRAY WRAPPER` |
| Quotes in `JSON_QUERY` | `KEEP QUOTES` |

**`PASSING`.** `PASSING x AS i` binds the path variable `$i` to the value of `x` (§9.46). A value of type `JSON` is passed as it is, and a string, number, boolean or datetime becomes an SQL/JSON scalar of the same type. A datetime is kept as a datetime. Other values, such as SQL `NULL`, `MISSING`, a tuple or an Ion value, are not specified yet (see [Unresolved questions](#unresolved-questions)).

```sql
JSON_VALUE(p, 'lax $.items[$i].sku' PASSING 1 AS i)               ->  STRING 'B2'
JSON_VALUE(JSON('{}'), 'lax $d.type()' PASSING DATE '2024-01-01' AS d)  ->  STRING 'date'
```

### `JSON_VALUE`

`JSON_VALUE` returns one SQL value (§6.28). The `RETURNING` type is a string, numeric, boolean or datetime type, and the item is cast to it with an ordinary `CAST`. An SQL/JSON null gives SQL `NULL`. An object, an array or several items is an error, which `ON ERROR` handles.

```sql
JSON_VALUE(p, 'lax $.customer.name')                       ->  STRING 'Bob'
JSON_VALUE(p, 'lax $.id' RETURNING INT)                    ->  INT 7
JSON_VALUE(p, 'lax $.note')                                ->  NULL       -- an SQL/JSON null
JSON_VALUE(p, 'lax $.age' RETURNING INT DEFAULT 0 ON EMPTY) ->  INT 0      -- no member age
JSON_VALUE(p, 'lax $.customer')                            ->  NULL       -- an object, and ON ERROR defaults to NULL
JSON_VALUE(p, 'lax $.items[*].sku')                        ->  NULL       -- two items
```

### `JSON_QUERY`

`JSON_QUERY` returns JSON (§6.35). With no `RETURNING`, the result is a value of type `JSON` for an input of type `JSON`, and the JSON text as a `STRING` for a string input. When the input's type is not known at compile time, each row follows its own input.

```sql
JSON_QUERY(p, 'lax $.customer')            ->  JSON {"name":"Bob","Tier":"gold"}
JSON_QUERY(o.payload, 'lax $.customer')    ->  STRING '{"name":"Bob","Tier":"gold"}'
SELECT VALUE JSON_QUERY(x, 'lax $.a') FROM [JSON('{"a": [1]}'), '{"a": [1]}'] AS x
-- << JSON [1], STRING '[1]' >>
```

**Wrappers.** Without a wrapper, several items are an error. `WITH CONDITIONAL ARRAY WRAPPER` wraps the items in an array only when there are several, and `WITH WRAPPER` always wraps.

```sql
JSON_QUERY(p, 'lax $.items[*].price')                                ->  NULL        -- two items, no wrapper
JSON_QUERY(p, 'lax $.items[*].price' WITH CONDITIONAL ARRAY WRAPPER) ->  JSON [10.5, 4]
JSON_QUERY(p, 'lax $.id' WITH CONDITIONAL ARRAY WRAPPER)             ->  JSON 7      -- one item, not wrapped
JSON_QUERY(p, 'lax $.id' WITH WRAPPER)                               ->  JSON [7]    -- always wrapped
```

**Quotes.** With a string result type, an SQL/JSON string scalar keeps its quotes by default. `OMIT QUOTES` returns the string itself. A quotes clause cannot follow `WITH ... WRAPPER`, and `OMIT QUOTES` is not allowed with `RETURNING JSON` (§6.35).

```sql
JSON_QUERY(p, 'lax $.customer.name' RETURNING STRING)              ->  STRING '"Bob"'
JSON_QUERY(p, 'lax $.customer.name' RETURNING STRING OMIT QUOTES)  ->  STRING 'Bob'
```

**A datetime.** A result that holds a datetime has no JSON text, so `RETURNING` a string type is an error, which `ON ERROR` handles (§9.43, §9.49).

```sql
JSON_QUERY(JSON('{"d": "2024-01-01"}'), 'lax $.d.date()' RETURNING STRING)   ->  NULL
```

### `JSON_EXISTS`

`JSON_EXISTS` is true when the path finds at least one item, and false when it finds none (§8.23). A member that holds SQL/JSON null is found.

```sql
JSON_EXISTS(p, 'lax $.note')       ->  true    -- the member exists, and holds SQL/JSON null
JSON_EXISTS(p, 'lax $.age')        ->  false   -- no member age
JSON_EXISTS(p, 'strict $.age')     ->  false   -- an error in strict mode, and ON ERROR defaults to FALSE
```

## Reading JSON with path steps

PartiQL's path steps, such as `.name` and `[0]`, also work on a value of type `JSON`. This is SQL:2023's simplified accessor (§6.36): each step after a value of type `JSON` reads from it, without a path string.

```sql
p.customer.name      ->  JSON "Bob"
```

Each chain of steps is the same as one `JSON_QUERY` call (§6.36), and that call gives its rules:

```sql
p.customer.name
<=> JSON_QUERY(p, 'lax $.customer.name'       -- the same steps, in lax mode
               WITH CONDITIONAL ARRAY WRAPPER  -- one result as it is, several in an array
               NULL ON EMPTY                   -- a member that does not exist gives SQL NULL
               NULL ON ERROR)                  -- any other error gives SQL NULL
```

**Member steps.** `.name` and `."name"` read a member. Keys match exact case, in both forms (§6.36).

```sql
p.customer.name        ->  JSON "Bob"
p.customer.tier        ->  NULL           -- exact case, and the key is "Tier"
p.customer."Tier"      ->  JSON "gold"
```

**A member that does not exist.** The result is SQL `NULL`, not `MISSING`, in both execution modes.

```sql
p.customer.age         ->  NULL
```

**Bracket keys.** PartiQL's other field access forms, `x['name']` and `x[CAST(e AS STRING)]`, also work on a value of type `JSON`. Each means `x."name"`, a member step with exact case.

```sql
p.customer['Tier']     ->  JSON "gold"    -- the same as p.customer."Tier"
p.customer['tier']     ->  NULL           -- exact case
```

**Array steps.** `[n]` reads an element, counting from 0. An index out of range gives SQL `NULL`. `x[e]`, where `e` is an expression, uses the value of `e` as the index. An index that is not a number is an error, so the result is SQL `NULL`.

```sql
p.items[0].sku         ->  JSON "A1"
p.items[5]             ->  NULL           -- out of range
p.items[t.i].sku       ->  JSON "B2"      -- t.i is 1
p.items[t.s]           ->  NULL           -- t.s is the STRING '1'
```

**Lax mode.** The steps always use lax mode (§6.36). A member step on an array runs on each element, and an array step on a value that is not an array treats it as a one-element array.

```sql
p.items.price          ->  JSON [10.5, 4]   -- .price on each element, and the two results wrapped
p.id[0]                ->  JSON 7           -- 7 counts as the one-element array [7]
```

**Results.** One result is returned as it is, several are wrapped in an SQL/JSON array, and none gives SQL `NULL`. The result has type `JSON`, even when it is a scalar. To get an SQL value, use `JSON_VALUE`.

```sql
p.items[0]                                   ->  JSON {"sku":"A1","price":10.5}
p.items.sku                                  ->  JSON ["A1","B2"]
p.customer.name = 'Bob'                      ->  a compile error   -- a value of type JSON and a STRING
JSON_VALUE(p, 'lax $.customer.name') = 'Bob' ->  true
```

**Repeated keys.** A step returns the first member with that key, in the order of the text (see [Duplicate keys](#parsing-json-text-jsontext)).

**What comes before the steps.** The steps apply only to a value of type `JSON`. A string is not parsed, so `o.payload.customer` keeps today's behavior on a string. The value before the steps can be any expression of type `JSON`, including a function call, with or without parentheses.

```sql
o.payload.customer               ->  MISSING in permissive mode, an error in type checking mode   -- a STRING
JSON(o.payload).customer.name    ->  JSON "Bob"
(JSON(o.payload)).customer.name  ->  JSON "Bob"
```

**Name resolution.** Name resolution runs first, with today's rules. In `JSON(o.payload).customer`, `o.payload` resolves as today, and only the steps after a value of type `JSON` follow this section.

**How this differs from tuples.**

| Step | Tuples and Ion values, unchanged | JSON |
|---|---|---|
| A member that does not exist | `MISSING`, or an error in type checking mode | SQL `NULL`, in both modes |
| Unquoted `.name` | Matches any case | Matches exact case |
| A member step on an array | `MISSING`, or an error | Runs on each element |
| An index out of range | `MISSING`, or an error | SQL `NULL` |
| How many results | Exactly one | Zero or more. Several are wrapped in an array |

## JSON in PartiQL's clauses and functions

### `FROM`: iterating an SQL/JSON array

A value of type `JSON` that is an SQL/JSON array can be used in `FROM`, and gives one row per element. Each element has type `JSON`. `AT` gives its position, counting from 0, as for a PartiQL array.

```sql
SELECT JSON_VALUE(i, 'lax $.sku') AS sku, n
FROM orders AS o, JSON(o.payload).items AS i AT n
-- << {'sku': 'A1', 'n': 0}, {'sku': 'B2', 'n': 1} >>
```

A value of type `JSON` that is not an array follows PartiQL's rule for a value that is not a collection: it is treated as a one-element bag in permissive mode, and is an error in type checking mode. Run on PartiQL today, `FROM {'a': 1} AS x` gives one row in permissive mode and an error in type checking mode.

### `WHERE`, `HAVING` and `ON`

A value of type `JSON` is not a boolean, even when it is the SQL/JSON scalar `true`. PartiQL treats a condition that is not a boolean as not true, so the row is left out, in both modes. To test an SQL/JSON boolean, read it with `JSON_VALUE ... RETURNING BOOLEAN`.

```sql
SELECT VALUE o FROM orders AS o WHERE JSON('true')                                          -- no rows
SELECT VALUE o FROM orders AS o WHERE JSON_VALUE(o.payload, 'lax $.id' RETURNING INT) = 7  -- the row
```

### Aggregate functions

`COUNT` accepts a value of type `JSON`. An SQL/JSON null is a value, not SQL `NULL`, so `COUNT` counts it. `SUM`, `AVG` and the other numeric aggregates do not accept JSON: an input of type `JSON` is a compile error, and an input whose type is not known gives `MISSING` in permissive mode and an error in type checking mode. `MIN` and `MAX` need comparison, which is not specified yet (see [Unresolved questions](#unresolved-questions)). The JSON aggregates are under [the constructors](#building-arrays-and-objects-the-constructors).

```sql
SELECT COUNT(x) AS c FROM [JSON('1'), NULL, JSON('null')] AS x
-- << {'c': 2} >>      -- the SQL/JSON null counts, SQL NULL does not
```

### Comparison, grouping and ordering

Comparing values of type `JSON` is not specified yet: `=`, `<`, `IN`, `DISTINCT`, `GROUP BY`, `ORDER BY`, `UNION` and the other set operations that remove duplicates, and joins on a JSON key (see [Unresolved questions](#unresolved-questions)). A value of type `JSON` and a value of another type are not comparable. To compare JSON with an SQL value, read it with `JSON_VALUE`.

```sql
p.id = 7                                         ->  a compile error   -- a value of type JSON and an INT
JSON_VALUE(p, 'lax $.id' RETURNING INT) = 7      ->  true
```

### Type predicates

A value of type `JSON` is not a tuple, an array or a string, so PartiQL's type predicates such as `x IS STRUCT`, `x IS LIST` and `x IS STRING` are false for it. To test for JSON, use `is_json_type`, and to test the shape of an SQL/JSON item, use `is_valid_json` (see [Testing JSON](#testing-json)).

```sql
p.customer IS STRUCT                   ->  false
is_json_type(p.customer)               ->  true
is_valid_json(p.customer, OBJECT)      ->  true
```

# Relation to SQL:2023

This RFC implements SQL:2023's `JSON` type (T801) and follows SQL:2023 wherever SQL:2023 decides a behavior: §4.8 and §4.48 (the type and data model), §6.28 and §6.34 to §6.41 (the operators), §8.23 (`JSON_EXISTS`), §9.42 to §9.49 (parsing, serializing, the path language and the query operators) and §10.11 to §10.14 (the aggregates and shared clauses).

## Where PartiQL differs

1. **`MISSING`.** SQL:2023 has no `MISSING`. This RFC defines how `MISSING` behaves as an input to each operator, following PartiQL's rules for tuples.
2. **Permissive mode.** SQL:2023 has only errors. This RFC defines permissive mode for invalid JSON text and wrongly typed input. Errors inside a query operator still go to `ON ERROR`, as in SQL:2023.
3. **Types not known at compile time.** SQL:2023 decides result types from declared types. PartiQL is gradually typed, so the constructors decide for each row, and the aggregates for each group.
4. **No `IS JSON`.** SQL:2023's `IS JSON` becomes the function `is_valid_json`, and PartiQL adds `is_json_type` as a type test.
5. **Bracket keys on JSON.** `x['name']` works on a value of type `JSON`, as it does on a tuple. SQL:2023 has no bracket key.
6. **Steps after a function call.** A path step can follow `JSON(...)` or `JSON_SCALAR(...)` without parentheses. This is an open question.
7. **Iteration in `FROM`.** An SQL/JSON array can be iterated in `FROM`. SQL:2023 turns JSON into rows only through `JSON_TABLE`.
8. **Ion values.** SQL:2023 has no Ion type. This RFC allows a CAST from an Ion value to JSON.
9. **Keywords.** SQL:2023 reserves `JSON` and the JSON operator names. PartiQL keeps them non-reserved, so existing names keep working.

## Conformance

PartiQL claims these SQL:2023 features once implemented:

- T801 (JSON data type), T802 (`JSON(...)` with unique keys), T803 (string-based JSON).
- T811 to T814 (constructors, `JSON_OBJECTAGG`, `JSON_ARRAYAGG` with `ORDER BY`, colon in `JSON_OBJECT`), T830 (unique keys in constructors).
- T821 (basic query operators) in part: `JSON_VALUE`, `JSON_QUERY` and `JSON_EXISTS`, without `IS JSON` and `JSON_TABLE`.
- T823 (`PASSING`), T825 and T826 (`ON EMPTY`, `ON ERROR`), T828 (`JSON_QUERY`), T829 (array wrappers).
- T831 to T837 and T840 (the path language), T865 to T878 (item methods).
- T851 (`FORMAT JSON` where it is the default).
- T860, T861 and T864 (the simplified accessor).

It does not claim T822 (`IS JSON WITH UNIQUE KEYS`, replaced by `is_valid_json`), the `JSON_TABLE` features (T824, T827, T838), T862 and T863 (wildcards and item methods in the simplified accessor), or the comparison features (T879 to T882), which are under [Unresolved questions](#unresolved-questions).

# Drawbacks

- **JSON and tuples answer some paths differently.** A member that does not exist is SQL `NULL` on JSON and `MISSING` on a tuple. Unquoted keys match exact case on JSON and any case on a tuple. When an expression's type is not known at compile time, rows of the two kinds give different results for the same path.
- **JSON and Ion values behave differently.** An Ion null is SQL `NULL`, an Ion value is converted to make an operation work, and Ion values mix with other types in `CASE` and `COALESCE`. A value of type `JSON` does none of these.
- **A datetime in JSON cannot be written as JSON text.** `JSON_SERIALIZE` on it fails, as SQL:2023 says. None of the engines run does this, so queries moved from them can fail here.
- **The constructors return a `STRING` by default**, as SQL:2023 says. Users of PostgreSQL, where they return `json`, may expect JSON.
- **Duplicate keys return the first member.** PostgreSQL returns the last, and some engines reject duplicates. A query translated to them can give a different answer for rows with duplicate keys.
- **In type checking mode, a query operator can return `NULL` where `JSON(...)` raises an error,** because errors inside a query operator go to `ON ERROR`.

# Rationale and alternatives

- **A separate `JSON` type, not the Ion type.** Ion has types JSON lacks, such as symbols and annotations, and JSON has a null that is not SQL `NULL`. Changing Ion values to behave like JSON would change existing queries.
- **SQL:2023's `JSON` type in full (T801), including datetime.** The alternative was PostgreSQL's model, where JSON holds only JSON's own kinds. PostgreSQL does not implement T801.
- **PartiQL's rules where SQL:2023 has none.** `MISSING`, permissive mode and types not known at compile time follow the rules PartiQL already uses for tuples, so one query behaves the same way whatever kind of value a row holds.
- **Two functions instead of `IS JSON`.** `x IS JSON` would read like PartiQL's type predicates, such as `x IS STRING`, but test content.
- **Keep every member for a duplicate key, and return the first.** It loses no data, and matches PartiQL's tuples and Ion structs.
- **Results decided per row, or per group, when a type is not known.** Deciding at compile time would give a `STRING` even for rows whose input is JSON.
- **The SQL/JSON path language by reference.** This RFC changes nothing in it.

# Prior art

| Topic | [PostgreSQL 17](https://www.postgresql.org/docs/17/functions-json.html) | [Trino 484](https://trino.io/docs/current/functions/json.html) | [MySQL 8.4](https://dev.mysql.com/doc/refman/8.4/en/json.html) | [Oracle 23ai](https://docs.oracle.com/en/database/oracle/oracle-database/23/adjsn/) | [DuckDB 2.0](https://duckdb.org/docs/2.0/data/json/overview.html) |
|---|---|---|---|---|---|
| JSON type | `json` keeps the text as written. `jsonb` stores a parsed binary form | `json`. Keeps member order and duplicate keys | `JSON`, a binary form | `JSON`, a binary form that adds types such as date | `JSON`, a logical type stored as text |
| Claims to the SQL standard | Its feature table lists T803 as supported and T801 as not implemented | Its docs say "according to the SQL standard, there shall be no JSON data type", which is SQL:2016's string-based model. No feature table | Its docs define `JSON` by RFC 8259, and do not mention the SQL standard | Its feature table lists support for T811 to T835, the SQL:2016 functions, with no row for T801 or T803 | Its docs do not mention the SQL standard |
| A date held in JSON | No. `JSON_SCALAR(DATE …)` is a string | Only inside a path | Yes | Yes | No |
| A step on a duplicate key | The last | Both | The last | The first, for a string input | The first |
| A member that does not exist | `NULL` | `NULL` | `NULL` | `NULL` | `NULL` |
| Testing JSON text | `IS JSON` | None | `JSON_VALID` | `IS JSON` | `json_valid` |
| Bracket key | `jsonb['a']` | No | No | No | Yes |
| Constructor default | `json` | A string | | | `JSON` |

All results were run on the versions shown, except the row of claims, which comes from each engine's documentation. Spark 4.0.1 was run for maps and dates: its `VARIANT` keeps a date, and a duplicate map key is an error by default.

[RFC 9535](https://www.rfc-editor.org/rfc/rfc9535) (JSONPath) is a separate language from the SQL/JSON path language, and this RFC does not follow it.

# Unresolved questions

## Questions for this review

Each has a proposal in the RFC text, and an alternative, in an author comment marked **Open question, team input needed**. The team's answers decide the rule.

| Question | Where | Proposal | Alternative |
|---|---|---|---|
| A JSON number too large for its type, such as `1e400` | [Parsing JSON text](#parsing-json-text-jsontext), "A number out of range" | Not accepted: `MISSING` in permissive mode, an error in type checking mode | An SQL/JSON scalar that is `FLOAT` `Infinity`, as PartiQL's literal `1e400` |
| Duplicate keys when parsing | [Parsing JSON text](#parsing-json-text-jsontext), "Duplicate keys" | Keep every member, as PostgreSQL's `json`, and return the first | Keep only the last, as PostgreSQL's `jsonb` |
| A duplicate key with `WITH UNIQUE KEYS`, in permissive mode | [Parsing JSON text](#parsing-json-text-jsontext), "`WITH UNIQUE KEYS`" | `MISSING`, like invalid text | An error in both modes |
| `is_valid_json(x)` when `x` has no known type and is neither a string nor JSON | [Testing JSON](#testing-json), "An input whose type is not known" | False, as a type predicate gives | `MISSING` in permissive mode, an error in type checking mode |
| A path step directly after `JSON(...)` or `JSON_SCALAR(...)` | [Reading JSON with path steps](#reading-json-with-path-steps), "What comes before the steps" | Allowed with or without parentheses | Parentheses required, as SQL:2023 does |

## Open for later

These need further design, and are not needed by the rules above. They are not questions for this review.

| Question | SQL:2023 | What needs deciding |
|---|---|---|
| The exact text `JSON_SERIALIZE` writes: member order, whitespace, and how a number is spelled, such as `1e3` or `1000` | UV097 | Whether to fix the text. SQL:2023 calls an SQL/JSON object unordered (§4.48.3) and does not require an implementation to document its text. PostgreSQL's `jsonb` sorts keys by length, then by bytes, and writes `1e3` as `1000` |
| `RETURNING` a binary string type, and `ENCODING` after `FORMAT JSON` | §6.37, §10.12 | Which binary string types PartiQL accepts, as input and as a result |
| More steps in the simplified accessor: wildcards `.*` and `[*]`, several subscripts and ranges, `.'name'`, and item methods such as `j.a.number()` | §6.36, T862, T863 | Each overlaps PartiQL syntax: path wildcards, the tuple step `.'name'`, and function calls. Which to allow, and how they parse |
| What `PASSING` gives for SQL `NULL`, `MISSING`, a tuple, an array, a bag or an Ion value | §9.46, §10.14 | What each becomes inside the path. SQL:2023 casts types it doesn't list to a string (IV165) |
| Constructor inputs of other types: an interval, a binary value, a tuple, an array, a bag or an Ion value | §6.34 casts other SQL types to a string | Whether to accept each, and what it becomes. PostgreSQL gives `"1 day"` for an interval and `"\\x4142"` for binary |
| Comparison, grouping and ordering of values of type `JSON`, including a `JSON` sort key in `JSON_ARRAYAGG` | T879 to T882 | Equality and ordering of SQL/JSON items (IA224, IA225) |
| `JSON_TABLE` | §7.11, T824, T827, T838 | How it fits PartiQL's `FROM`, `MISSING` and execution modes |
| How `CAST(v AS JSON)` maps an Ion value to an SQL/JSON item | Not in SQL:2023 | How each Ion value maps, such as an Ion null, a typed null, a timestamp, a symbol, a blob or an annotated value |

# Future possibilities

- **A binary representation,** as PostgreSQL's `jsonb` has, for faster access. It would not change any result in this RFC, as long as it keeps every member in the order of the text.
- **The questions under [Unresolved questions](#unresolved-questions),** such as comparison and `JSON_TABLE`.
- **An explicit conversion from JSON to tuples and arrays,** as PartiQL lowers Ion values today. Because duplicate keys return the first member, as tuples do, it would not change which member a path returns.
- **Functions from other engines,** such as Redshift's `JSON_EXTRACT_PATH_TEXT`, as extensions rather than as part of the language.

# Appendix

## Where each section goes in the specification

Each section of this RFC adds to a chapter of the specification revamp in [#102](https://github.com/partiql/partiql-lang/pull/102):

| Section in this RFC | Chapter in #102 |
|---|---|
| [The JSON type](#the-json-type) | Data Types |
| [JSON when the type is not known at compile time](#json-when-the-type-is-not-known-at-compile-time) | Typing Model |
| [Parsing JSON text](#parsing-json-text-jsontext), [Writing JSON text](#writing-json-text-json_serialize), [Making a scalar](#making-a-scalar-json_scalar), [the constructors](#building-arrays-and-objects-the-constructors) | Functions and Operators |
| [CAST](#cast) | Type Conversions |
| [Testing JSON](#testing-json) | Functions and Operators |
| [Querying JSON](#querying-json) | Functions and Operators |
| [Reading JSON with path steps](#reading-json-with-path-steps) | Functions and Operators: Field Access and Indexing Operators |
| [`FROM`](#from-iterating-an-sqljson-array), [`WHERE`](#where-having-and-on) | PartiQL Query Syntax: FROM Clause, WHERE Clause |
| [Aggregate functions](#aggregate-functions), [Comparison](#comparison-grouping-and-ordering), [Type predicates](#type-predicates) | Functions and Operators |

## Choices SQL:2023 leaves to implementations

| Code | Choice | PartiQL |
|---|---|---|
| IV048, IV119 | The type of a numeric literal, and so of a JSON number | The type of the same PartiQL literal: `INT`, `DECIMAL` or `FLOAT` |
| UA057 | Whether parsing drops members with duplicate keys | Keeps every member |
| US039 | The order parsing reads members in | The order of the text |
| UA075 | Which member a path step returns when keys repeat | The first, in the order of the text |
| IA071 | Whether a path step raises an error for duplicate keys | No error, in either mode |
| ID074 | The string type `JSON_VALUE` returns by default | `STRING` |
| ID101 | The string type `JSON_QUERY` returns for a string input | `STRING` |
| ID098, ID134 | The string type a constructor or aggregate returns by default | `STRING` |
| ID100 | The string type a constructor casts a datetime input to | `STRING` |
| ID102 | The string type of a `JSON_SERIALIZE` result | `STRING` |
| IA050 | When a query operator parses a string input | Before the path runs. An invalid input goes to `ON ERROR` either way |
| IV185 | The unique-keys check a query operator applies when it parses a string input | `WITHOUT UNIQUE KEYS`, as `JSON(...)` |
| IV165 | The string type `PASSING` casts other types to | Not specified yet |
| UV097 | The exact text of serialized JSON | Not specified yet |

# References

- ISO/IEC 9075-2:2023, Information technology, Database languages SQL, Part 2: Foundation (SQL/Foundation).
- [RFC 8259](https://www.rfc-editor.org/rfc/rfc8259), The JavaScript Object Notation (JSON) Data Interchange Format.
- [partiql/partiql-lang#102](https://github.com/partiql/partiql-lang/pull/102), the specification revamp whose chapters this RFC follows.
