- Start Date: 2022-09-15
- PartiQL Issue [partiql/partiql-docs/#27](https://github.com/partiql/partiql-docs/issues/27)
- RFC PR: [partiql/partiql-docs/#30](https://github.com/partiql/partiql-docs/pull/30)

# Summary
[summary]: #summary

This RFC adds specification of PartiQL's syntax and semantics for insert-or-update—also known as `UPSERT`—and insert-or-replace as part of PartiQL data manipulation language (DML). The semantics is equivalent to `INSERT INTO ... ON CONFLICT DO UPDATE/REPLACE EXCLUDED` as outlined in [RFC-0011](https://github.com/partiql/partiql-docs/blob/main/RFCs/0011-partiql-insert.md).

# Motivation
[motivation]: #motivation

PartiQL already has the specification of the same semantics using `INSERT INTO ... ON CONFLICT DO UPDATE/REPLACE EXCLUDE`, here we propose to use designated syntax for `UPSERT` and `REPLACE`. This is to provide more ergonomic choices for implementations that require to adopt the corresponding semantics as separate statements.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The keywords “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this format specification are to be interpreted as described in [Key Words RFC2119](https://datatracker.ietf.org/doc/html/rfc2119).

## Terminology
[terminology]: #terminology

* **Attribute**: The column of a table (or schema) are named by attributes in relational model. In non-relational model they’re equivalent to column headers; for example in NoSQL model they are keys in key-value pairs.
* **Closed schema**: A schema where all attributes are declared, and undeclared attributes are not permitted.
* **DDL**: A data definition language defines SQL statements used for defining or altering various database entities like tables and views.
* **DML**: A data manipulation language defines SQL statements used for adding (inserting), deleting, and modifying (updating) data in a database.
* **Declared attributes**: Any attribute that is defined in the Schema with a name, type, whether the attribute is required or optional, and any additional properties; all declared attributes are either required or optional.
* **Item**: A collection of attributes and their corresponding values in a non-relational (E.g. NoSQL) data model.
* **NoSQL data model**: A non-relational data model that represents the data in a non-tabular way, for example in document or key-value form.
* **Open schema**: A Schema where undeclared attributes are permitted.
* **Optional attributes**: A set, comprising the declared attributes that are not necessarily required for a successful data insertion or update to occur.
* **PartiQL value**: A PartiQL value is a value defined by PartiQL specification that can be one of absent, missing, scalar, tuple, array, or bag. (see PartiQL Specification Figure 1 for more details).
* **Relation**: A two-dimensional representation of data in rows and columns.
* **Relational model**: A data model that represents data as relations; with this model, data is organized into relations where each relation is an unordered collection of tuples.
* **Required attributes**: A set, comprising the declared attributes that must exist for a data item to conform to a Schema.
* **Schema**: Schema is an abstract entity that is used as a blueprint to describe data along with its associated constraints.
* **Tuple**: The rows of relations which are collection of column values. In a non-relational data model (E.g. NoSQL/semi-structured model) denotes a collection of name/value pairs also known as an Item; tuples can be ordered or unordered depending on the database system.

### Out of scope

* Returning output from `UPSERT`/`REPLACE` statement—the specification of returning output is considered as an extension to this RFC which is planned to be defined in the future.
* Type coercion implementations—the type coercion when inserting data from a source schema to a target schema with an expression (E.g. a sub-select statement) is considered as a database system implementation concern.
* Security privileges for execution of DML statements—the data access and manipulation security specification is considered as a database system specific feature, hence is considered out of scope for this RFC.

## 2. Proposed grammar and semantics
[proposed-grammar-and-semantics]: #proposed-grammar-and-semantics

Figure 1. illustrates the BNF grammar corresponding to PartiQL’s `UPSERT` and `REPLACE` statements. See Appendix (1) for the Grammar conventions.

Note:

* All the definitions that are missing from the grammar refer to the definitions that are already in SQL-92 standard.

```EBNF
<upsert statement> ::= UPSERT INTO <table name> [ AS <alias> ] 
    [  ( <attr name> [, <attr name> ]... ) ]
        <values>
        
<replace statement> ::= REPLACE INTO <table name> [ AS <alias> ] 
    [  ( <attr name> [, <attr name> ]... ) ]
        <values>

<values> ::= DEFAULT VALUES | <values clause>
      | <bag value> | <sub-select>
 
<values clause> ::= VALUES <value> [, <value>]...

<value> ::= ( { <value expr> | DEFAULT } [, { <value expr> | DEFAULT } ]...)

<value expr> ::= <partiql value>
     | <sql value expr>
     
<partiql value> ::= NULL| <ion value>
    | <tuple value>
    | <collection value>

<sql value expr> ::= <sql expression>

<ion value> ::= <backtick> <ion literal> <backtick>

<tuple value> ::= <l curly brace> <r curly brace>
    | <l curly brace> <string value> : <partiql value> [, <string value> : <partiql value>]... <r curly brace>

<collection value> ::= <array value>
    | <bag value>

<array value> ::= <left bracket> { <partiql value> [, <partiql value>]... } <right bracket>

<bag value> ::= <opening bag> { <partiql value> [, <partiql value>]... } <closing bag>
<sub-select> ::= <sfw query>

(* See sfw_query in PartiQL spec. figure 3: BNF Grammar for PartiQL Queries:https://partiql.org/assets/PartiQL-Specification.pdf *)
<sfw query>

(* Literals *)
<l curly brace> ::=
    “{”
<r curly brace> ::=
    “}”
<left bracket> ::=
    “[”
<right bracket> ::=
    “]”
<backtick> ::=
    `
<opening bag> ::=
    <<
<closing bag> ::=
    >>
<left paren> ::=
  “(”
<right paren> ::=
  “)”
```

## 3. Description

Implementation based on this specification MUST lead to atomic insertions/replacements of PartiQL values into a Schema (E.g. a database table). Atomic insertion/replacement means that either all or none of the insertions/replacements specified by the `UPSERT`/`REPLACE` statement are performed—E.g. when there is an error in insertion/replacement of one or more data items.

`UPSERT` or `REPLACE` clause MUST be a “deterministic” statement. This means that the command MUST not be allowed to affect any single existing row or item more than once; a `SemanticError` (E.g. cardinality violation error) MUST be raised when this situation arises. Rows resulting from insertion after conflict, should not duplicate each other in terms of attributes constrained by an arbiter index or constraint. 

For both `UPSERT` and `REPLACE`, when conflict occurs on a primary key or a constraint and the proposed value for `UPDATE` or `REPLACE` is a `<tuple value>` — attributes comprising the primary key, or the constraint MUST exist in the proposed `<tuple value>`, otherwise a `SemanticError` MUST be raised.

### 3.1 Attribute names

As providing attribute names is optional, when attribute names are omitted and values are provided by `VALUES`, a `<sub-select>`, or bag of lists (E.g. `<< ['v1', 'v2'], ['v3', 'v4'] >>`):

* (a) If the number of values in a data item is equal to the number of declared attributes, the values are assumed to refer to the declared attributes, in order of their definition in the table schema, and
* (b) If the number of values in data item (M) is fewer than the number of declared attributes (N) the values are assumed to refer to the first M declared attributes, in order, and
* (c) If the number of values in a data item (M) is greater than the number of declared attributes (N) it is an error, and
* (d) If a value is not specified for every required attribute, it is an error.

It’s worth emphasizing that, the implementing database MUST provide an ordering for the declared attributes that appear in the Schema.

When attribute names are omitted and bag of tuples is provided, each `<tuple value>` within the `<bag value>` represents an attribute-value pair with attribute corresponding to the Schema.
If a `<tuple value>` for an attribute-value pair is not specified in the bag, default value or null value (in case of attribute being nullable) for the corresponding attribute will be used. 
The values for all not null attributes SHALL be specified if a default value is not specified in the Schema.

When attribute names are provided with bag of tuples, it's a `SematicError`.

When attribute names are provided with `VALUES` or bag of lists, they MAY be listed in any order. Each attribute—required or optional—that is omitted in the provided attributes, MAY be filled with a default value, either its declared default value or an implementation of `NULL`—in case attribute is nullable. If the expression for any attribute is not of the correct data type, automatic type conversion i.e. type coercion MAY be attempted.

Section 3.3 provides more clarification with examples.

### 3.2 Query

`SELECT` query expression can be used for inserting/replacing the resulting values from the query to the target table. The resulting tuples (E.g. rows) MUST have the same attribute name and type as the target schema’s required attributes.

Optional attributes in target schema that have the same name in the resulting tuples from the source schema, SHOULD have the same type. If the implementing database provides type coercion rule for the types involved in source and destination, the `SELECT` MAY lead to a successful outcome, otherwise it's a `SemanticError`. See examples for more clarification.

## 4. Parameters

`<table name>` is the name of an existing table.

`<alias>` is a substitute name for `<table name>`. When an alias is provided, it SHOULD hide the actual name of the table. 

`<attr name>` is the name of an attribute in the table named by `<table name>`.

`DEFAULT VALUES` denotes that all required attributes MUST be filled with their default values or `NULL` in case of attributes being nullable. This is as if `DEFAULT` were explicitly specified for each attribute. If the target database system has no concept or implementation of default values, this should return a `SemanticError` exception.

`<value expr>` is a value expression (E.g. `1` or `2+2`) that can get assigned to the corresponding attribute.

`DEFAULT` denotes the corresponding attribute gets filled with its default value. For a generated attribute (E.g. auto-increments), specifying this is permitted which SHOULD result in computing the attribute value from its data-generation expression.

## 5. Examples

### `UPSERT`/`REPLACE` - INSERT (No violation on unique constraint)

In the following examples, `Films` table has a closed schema. In the table `len` attribute is implicitly defined as nullable.

In the following examples `CREATE TABLE` including `SCHEMA OPEN` and `SCHEMA CLOSED` syntax is arbitrary since the PartiQL DDL is yet to be defined—see the "Terminology" section for open and closed schema definitions.

```SQL
-- The following `CREATE TABLE` syntax is arbitrary since the PartiQL DDL is yet to be defined:
CREATE TABLE Films /* SCHEMA CLOSED */
(
   code  VARCHAR(40) PRIMARY KEY DEFAULT '1',
   title VARCHAR(100) DEFAULT 'Default Film',
   did   INTEGER DEFAULT 10,
   date_prod DATE DEFAULT NOW()::TIMESTAMP,
   kind VARCHAR(50) DEFAULT 'Comedy',
   len VARCHAR(50)
);
```

#### Example 5.1

The following statements insert items while the `len` attribute is omitted in the target attributes and the implementing database allows using default value or `NULL` for the omitted attributes in the target attributes. In addition, implementing database sets the value of `len` to `NULL` — this is because based on the DDL for `Films`, `len` is nullable.

```SQL
-- Same example is applicable to `REPLACE`
UPSERT INTO Films VALUES ('UA502', 'Bananas', 105, '1971-07-13', 'Comedy', '82 minutes');

-- Same example is applicable to `REPLACE`
UPSERT INTO Films (code, title, did, date_prod, kind)
VALUES ('UA502', 'Bananas', 105, '1971-07-13', 'Comedy', '82 minutes');
```

#### Example 5.2

The following examples uses the `DEFAULT` clause for the `date` and `len` attributes rather than specifying a value.

For the second query, the `len` attribute is omitted in the target attributes and the implementing database allows using default value or `NULL` for the omitted attributes. Therefore, implementing database sets the value of `len` to `NULL` — this is because based on the DDL for `Films` `len` is nullable.

For the third query, the `kind` attribute is omitted in the target attributes and the order of attributes has changed. In this case, the implementing database allows using a different order and default value or `NULL` for the omitted attributes. Therefore, implementing database sets the value of `kind` to its default value 'Comedy':

```SQL
-- Same example is applicable to `REPLACE`
UPSERT INTO Films
VALUES ('UA503', 'Bananas', 105, DEFAULT, 'Comedy', DEFAULT);

-- Same example is applicable to `REPLACE`
UPSERT INTO Films (code, title, did, date_prod, kind)
VALUES ('T_603', 'Yojimbo', 106, DEFAULT, 'Drama');

-- Same example is applicable to `REPLACE`
UPSERT INTO films (title, code, did, date_prod, len)
VALUES ('MyTitle', 'MyCode', 108, '1961-06-16', '180 minutes');

SELECT * FROM Films;<<
{
   'code': 'UA503',
   'title': 'Bananas',
   'did': 105,
   'date_prod': 2022-08-10T,
   'kind': 'Comedy',
   'len': NULL
},
{
   'code': 'T_603',
   'title': 'Yojimbo',
   'did': 106,
   'date_prod': 2022-08-10T,
   'kind': 'Drama',
   'len': NULL
},
{
   'code': 'MyCode',
   'title': 'MyTitle',
   'did': 108,
   'date_prod': 1961-06-16T,
   'kind': 'Comedy',
   'len': '180 minutes'
}>>
```
The following example statement inserts a row or item consisting entirely of default values:

```SQL
-- Same example is applicable to `REPLACE`
UPSERT INTO Films DEFAULT VALUES;
```

#### Example 5.3

The following example statement inserts multiple rows using the multi-row VALUES syntax:

```SQL
-- Same example is applicable to `REPLACE`
UPSERT INTO Films (code, title, did, date_prod, kind)
VALUES ('B6717', 'Tampopo', 110, '1985-02-10', 'Comedy'),
       ('HG120', 'The Dinner Game', 140, DEFAULT, 'Comedy');
```

#### Example 5.4

In the following statement, we insert values to `Music` table, `Music` table has a closed schema with `Artist` and `SongTitle` as required attributes:

```SQL
-- The following `CREATE TABLE` syntax is arbitrary since the PartiQL DDL is yet to be defined:
CREATE TABLE Music /* SCHEMA CLOSED */
(
   Artist     VARCHAR(20) NOT NULL,
   SongTitle  VARCHAR(30) NOT NULL,
   PRIMARY KEY (Artist, SongTitle)
);

-- Same example is applicable to `REPLACE`
UPSERT INTO Music
<<
{'Artist' : 'Acme Band', 'SongTitle' : 'PartiQL Rocks'},
{'Artist' : 'Emca Band', 'SongTitle' : 'PartiQL Rocks'}
>>;
```

#### Example 5.5

In the following statement we insert multiple person items as a bag value to `Person` table, `Person` table has an open schema with `LastName` and `DOB` as required attributes and `FirtName` as optional attribute.

```SQL
-- The following `CREATE TABLE` syntax is arbitrary since the PartiQL DDL is yet to be defined:CREATE TABLE Person /* SCHEMA OPEN */
(
   LastName    VARCHAR(50) NOT NULL,
   FirstName   VARCHAR(20),    
   DOB        DATE NOT NULL,
   PRIMARY KEY (LastName)
);

-- Same example is applicable to `REPLACE`
UPSERT INTO Person<<
{'FirstName' : 'Raul','LastName' : 'Lewis','DOB' : 1963-08-19T,'GovId' : 'LEWISR261LL','GovIdType' : 'Driver License',
},{'LastName' : 'Logan','DOB' : 1967-07-03T,'Address' : '43 Stockert Hollow Road, Everett, WA, 98203'
},{'LastName' : 'Pena','DOB' : 1974-02-10T,'GovId' : '744 849 301','GovIdType' : 'SSN','Address' : '4058 Melrose Street, Spokane Valley, WA, 99206'
}>>;

SELECT * FROM Person;<<
{'LastName' : 'Lewis','FirstName' : 'Raul','DOB' : 1963-08-19T,'GovId' : 'LEWISR261LL','GovIdType' : 'Driver License',
},
{'LastName' : 'Logan','FirstName' : NULL,'DOB' : 1967-07-03T,'Address' : '43 Stockert Hollow Road, Everett, WA, 98203'
},
{'LastName' : 'Pena','FirstName' : NULL,'DOB' : 1974-02-10T,'GovId' : '744 849 301','GovIdType' : 'SSN','Address' : '4058 Melrose Street, Spokane Valley, WA, 99206'
}>>;
```

In the following statement, we insert multiple items as a bag value to `Foo` table, `Foo` table has an open schema:

```SQL
-- The following `CREATE TABLE` syntax is arbitrary since the PartiQL DDL is yet to be defined:
CREATE TABLE Foo /* SCHEMA OPEN */
(
   id         INT     NOT NULL PRIMARY KEY,
   is_deleted BOOLEAN NOT NULL DEFAULT FALSE,
   title      VARCHAR(50),
   bar        VARCHAR(10)      DEFAULT 'baz',
);

-- In the following, inserting `{ 'id': 1 }` goes through because the rest of the attributes either have default value or
-- are nullable. In addition, because `Foo` has open-schema we can add `value` as a new attribute to the table.

-- Same example is applicable to `REPLACE`
UPSERT INTO Foo
<<
{ 'id': 1 },
{ 'id': 2, 'title': 'some-name' },
{ 'id': 3, 'is_deleted': true, 'bar': '10'},
{ 'id': 4, 'title': 'some-other-name', 'value': '10'}
>>;

SELECT * FROM Foo;
<<
{ 'id': 1, 'is_deleted': false, 'title': NULL, 'bar': 'baz' },
{ 'id': 2, 'is_deleted': false, 'title': 'some-name', 'bar': 'baz' },
{ 'id': 3, 'is_deleted': true, 'title': NULL, 'bar': '10' },
{ 'id': 4, 'is_deleted': false, 'title': 'some-other-name', 'bar': 'baz', 'value': '10'},
>>;
```

#### Example 5.6

The following example statement inserts items from `RockAlbums` table into `Music` table. Both tables have the same layout for required and optional attributes and `Music` table has open schema, therefore the first `UPSERT/REPLACE` statement can insert items using a `SELECT *` query. Furthermore, the second `UPSERT/REPLACE` leads to a `SemanticError` because `RockAlbums` has closed schema.

```SQL
-- The following `CREATE TABLE` is arbitrary since the PartiQL DDL is yet to be defined:
CREATE TABLE Music /* SCHEMA OPEN */
(
   Artist     VARCHAR(20) NOT NULL,
   SongTitle  VARCHAR(30) NOT NULL,
   AlbumTitle VARCHAR(25) NOT NULL,
   Year       INT,
   Price      FLOAT,
   Genre      VARCHAR(10),
   PRIMARY KEY (Artist, SongTitle)
);

-- The following `CREATE TABLE` syntax is arbitrary since the PartiQL DDL is yet to be defined:
CREATE TABLE RockAlbums /* SCHEMA CLOSED */
(
    Artist     VARCHAR(20) NOT NULL,
    SongTitle  VARCHAR(30) NOT NULL,
    AlbumTitle VARCHAR(25) NOT NULL,
    Year       INT,
    RockGenre  VARCHAR(10),
    PRIMARY KEY (Artist, SongTitle)
);

-- The following query goes through because `Price` attribute is nullable.
-- Same example is applicable to `REPLACE`
UPSERT INTO Music
SELECT *
FROM RockAlbums
WHERE RockGenre IN ('Alternative', 'SpaceRock');

-- The following leads to a `SemanticError` because `RockAlbums` has closed schema.
-- Same example is applicable to `REPLACE`
UPSERT INTO RockAlbums
SELECT *
FROM Music;
```

The following statement shows that the values can be specified with a sub-select. The INSERT goes through because the values that are not `SELECT` default to `NULL`.

```SQL
-- Same example is applicable to `REPLACE`
UPSERT INTO Music (Artist, SongTitle, AlbumTitle)
SELECT Artist, SongTitle, AlbumTitle
FROM RockAlbums
WHERE RockGenre IN ('Alternative', 'SpaceRock');
```

#### Example 5.7

The following example statement attempts to insert items from `RockAlbums` table to `Music` table. Both tables have the same layout for required attributes but have different types for `Year` optional attribute, therefore—if the implementing database system does not implement a type coercion rule from `DATE` to `INT` — the `INSERT` statement using a `SELECT` query leads to a `SemanticError`.

```SQL
-- The following `CREATE TABLE` syntax is arbitrary since the PartiQL DDL is yet to be defined:
CREATE TABLE Music /* SCHEMA OPEN */
(
   Artist     VARCHAR(20) NOT NULL,
   SongTitle  VARCHAR(30) NOT NULL,
   AlbumTitle VARCHAR(25) NOT NULL,
   Year       INT,
   Price      FLOAT,
   Genre      VARCHAR(10),
   PRIMARY KEY (Artist, SongTitle)
);

-- The following `CREATE TABLE` syntax is arbitrary since the PartiQL DDL is yet to be defined:
CREATE TABLE RockAlbums /* SCHEMA CLOSED */
(
    Artist     VARCHAR(20) NOT NULL,
    SongTitle  VARCHAR(30) NOT NULL,
    AlbumTitle VARCHAR(25) NOT NULL,
    Year       DATE,
    RockGenre  VARCHAR(10),
    PRIMARY KEY (Artist, SongTitle)
);

-- Same example is applicable to `REPLACE`
UPSERT INTO Music
SELECT * FROM RockAlbums;
```

#### Example 5.8

The following example statement, inserts some rows into table `Films`:

```SQL
-- The following `CREATE TABLE` syntax is arbitrary since the PartiQL DDL is yet to be defined:
CREATE TABLE Foo /* SCHEMA CLOSED */
(
   id         INT     NOT NULL PRIMARY KEY,
   is_deleted BOOLEAN NOT NULL DEFAULT FALSE,
   title      VARCHAR(50),
   bar        VARCHAR(10) DEFAULT 'baz'
);

-- Same example is applicable to `REPLACE`
UPSERT INTO Foo (id, title)
<<
[2, 'some-name'],
>>;

SELECT * FROM Foo;
<<
{ 'id': 2, 'is_deleted': false, 'title': 'some-name', 'bar': 'baz' }
>>;

UPSERT INTO Foo
<<
[3, true],
[4, true],
>>;

SELECT * FROM Foo;
<<
{ 'id': 3, 'is_deleted': true, 'title': NULL, 'bar': 'baz' },
{ 'id': 4, 'is_deleted': true, 'title': NULL, 'bar': 'baz' }
>>;
```

#### Example 5.9

The following statements lead to a `SemanticError`:

```SQL
-- The following `CREATE TABLE` syntax is arbitrary since the PartiQL DDL is yet to be defined:
CREATE TABLE Foo /* SCHEMA OPEN */
(
   id         int     NOT NULL PRIMARY KEY,
   is_deleted boolean NOT NULL DEFAULT FALSE,
   title      varchar(50),
   bar        varchar(10) DEFAULT 'baz'
);

-- `SemanticError` because of the presence of target attribute names with bag of tuples.
-- In this case, the behavior for adding or not adding attributes like `is_deleted` is undefined.
-- Same example is applicable to `REPLACE`
UPSERT INTO Foo (id, title)
<<
{ 'id': 1 },
{ 'id': 2, 'title': 'some-name' },
{ 'id': 3, 'is_deleted': true, 'title': NULL, 'bar': '10'}
>>;


-- `SemanticError` because not all <bag value> items (row values) can get mapped to the target attributes set.
-- Same example is applicable to `REPLACE`
UPSERT INTO Foo (id, title)
<<
[2, 'some-name'],1,'some-other-name'
>>;

-- `SemanticError` because UPSERT/REPLACE has more target attributes than row values specified by some of the `<bag value>` elements (E.g. `[1]`).
-- Same example is applicable to `REPLACE`
UPSERT INTO Foo (id, title)
<<
[1],
[1, 'some_name']
>>;

-- `SemanticError` because usage of `DEFAULT` outside `VALUES(...)` is unsupported.
-- Same example is applicable to `REPLACE`
UPSERT INTO Foo (id, title)
<<
[1, DEFAULT],
[2, 'some-name']
>>;

-- `SemanticError` because usage of `DEFAULT` outside `VALUES(...)` is unsupported.
-- Same example is applicable to `REPLACE`
UPSERT INTO Foo
<<
{'id': 1, 'is_deleted': DEFAULT},
{'id': 2, 'is_deleted': true}
>>;
```

### `UPSERT` - DO UPDATE

#### Example 5.10

`UPSERT`(Insert or update) an item into a NoSQL database. Assumes a unique constraint e.g. a primary key has been violated for an existing item. In this case, the statement updates the item with additional attributes.

```SQL
-- The following `CREATE TABLE` syntax is arbitrary since the PartiQL DDL is yet to be defined:
CREATE TABLE Customers /* SCHEMA OPEN */
(
    HK      INT     NOT NULL PARTITION KEY,
    RK      INT     NOT NULL SORT KEY,
);
```

```SQL
-- Existing Item with HK as primary key: {'HK': 1, 'RK': 1, 'otherAttr1': 5}
-- Item after the update:  {'HK': 1, 'RK': 1, 'otherAttr': 5, 'myAttr1': 1, 'myAttr2': 2}

UPSERT INTO Customers <<
{'HK': 1, 'RK': 1, 'myAttr1': 1, 'myAttr2': 2}
>>
```

#### Example 5.11

Same table schema as in the Example 5.10. In this case, the statement merges the item with additional attributes.

```SQL
-- Existing Item with HK as primary key:
--  {'HK': 1, 'RK': 1, 'myAttr': 10}
-- Item after the update:
--  {'HK': 1, 'RK': 1, 'myAttr': 12, 'anotherAttr': 'hello'}

UPSERT INTO Customers <<
{'HK': 1, 'RK': 1, 'myAttr1': 12, 'anotherAttr': 'hello'}
>>
```

#### Example 5.12

Same table schema as in the Example 5.10. The following example returns `SemanticError` due to missing composite primary key (hash key and range key) from the statement. 

```SQL
-- Existing Item is:
--  {'HK': 1, 'RK': 1, 'myAttr': 12 }
-- Outcome is:
--  SemanticError

UPSERT INTO Customers
<<{'HK': 1, 'thirdAttr': 'world'}>>

UPSERT INTO Customers
<<{'RK': 1, 'thirdAttr': 'world'}>>

UPSERT INTO Customers
<<{'thirdAttr': 'world'}>>
```

#### Example 5.13

`UPSERT`(Insert or update) an item into a NoSQL database. Assumes a unique constraint e.g. a primary key has been violated for an existing item. In this case, `UPSERT` statement includes attributes not belonging to the table schema, which leads to `SemanticError`.

```SQL
-- The following `CREATE TABLE` syntax is arbitrary since the PartiQL DDL is yet to be defined:
CREATE TABLE Customers /* SCHEMA CLOSED */
(
    HK      INT     NOT NULL PARTITION KEY,
    RK      INT     NOT NULL SORT KEY,
    OtherAttr INT   NOT NULL
);
```

```SQL
-- Existing Item with HK as primary key: {'HK': 1, 'RK': 1, 'otherAttr': 5}
-- Outcome is:
--  SemanticError

UPSERT INTO Customers <<
{'HK': 1, 'RK': 1, 'otherAttr': 4, 'myAttr1': 1, 'myAttr2': 2}
>>
```

### `REPLACE` - DO REPLACE

For the following examples we assume the existence of composite key `partition_key_sort_key` comprising `HK` and `RK` attributes i.e `HK_RK`.

#### Example 5.14

Same table schema as in the Example 5.10. The following example leads to replacement of the item specified in the statement:

```SQL
-- Existing Item is:
--  {'HK': 1, 'RK': 1, 'myAttr': 12 }
-- Outcome is:
--  {'HK': 1, 'RK': 1, 'thirdAttr': 'world'}

REPLACE INTO Customers
<<{'HK': 1, 'RK': 1, 'thirdAttr': 'world'}>>
```

#### Example 5.15

The following example returns `SemanticError` due to missing primary key(s) from the statement:

```SQL
-- Existing Item is:
--  {'HK': 1, 'RK': 1, 'myAttr': 12 }
-- Outcome is:
--  SemanticError

REPLACE INTO Customers
<<{'HK': 1, 'thirdAttr': 'world'}>>

REPLACE INTO Customers
<<{'RK': 1, 'thirdAttr': 'world'}>>

REPLACE INTO Customers
<<{'thirdAttr': 'world'}>>
```

#### Example 5.16

For table with closed schema, adding new attributes that are not specified in the schema or missing not nullable ones leads to `SemanticError`.

```SQL
-- The following `CREATE TABLE` syntax is arbitrary since the PartiQL DDL is yet to be defined:
CREATE TABLE Customers /* SCHEMA CLOSED */
(
    HK      INT     NOT NULL PARTITION KEY,
    RK      INT     NOT NULL SORT KEY,
    OtherAttr INT   NOT NULL
);
```

```SQL
-- Existing Item is:
--  {'HK': 1, 'RK': 1, 'OtherAttr': 12 }
-- Outcome is:
--  SemanticError

REPLACE INTO Customers
<<{'HK': 1, 'RK': 1, 'OtherAttr': 13, 'thirdAttr': 'world'}>>

REPLACE INTO Customers
<<{'HK': 1, 'RK': 1, 'thirdAttr': 'world'}>>
```

## References

* [PartiQL Insert RFC](https://github.com/partiql/partiql-docs/blob/main/RFCs/0011-partiql-insert.md)
* [Database Systems: The Complete Book, second Edition](https://www.goodreads.com/book/show/6608398-database-systems), Hector Garcia-Molina Jeffrey D. Ullman Jennifer Widom Department of Computer Science Stanford University.
* [Designing Data-Intensive Applications](https://www.goodreads.com/book/show/23463279-designing-data-intensive-applications), by Martin Kleppmann
* [Oracle (SQL for Oracle NoSQL Datbase)](https://docs.oracle.com/en/database/other-databases/nosql-database/18.3/sqlfornosql/adding-table-rows-using-insert-and-upsert-statements.html)
* [BNF-EBNF Grammars](https://matt.might.net/articles/grammars-bnf-ebnf/)
* [SQL 1992](https://www.contrib.andrew.cmu.edu/~shadow/sql/sql1992.txt)

## Appendices

#### 1. Grammar Definition Syntax:
```
   < >   Angle brackets delimit character strings that are the names
         of syntactic elements, the non-terminal symbols of the SQL
         language.

   ::=   The definition operator. This is used in a production rule to
         separate the element defined by the rule from its definition.
         The element being defined appears to the left of the opera-
         tor and the formula that defines the element appears to the
         right.

   [ ]   Square brackets indicate optional elements in a formula. The
         portion of the formula within the brackets may be explicitly
         specified or may be omitted.

   { }   Braces group elements in a formula. The portion of the for-
         mula within the braces shall be explicitly specified.

   |     The alternative operator. The vertical bar indicates that
         the portion of the formula following the bar is an alterna-
         tive to the portion preceding the bar. If the vertical bar
         appears at a position where it is not enclosed in braces
         or square brackets, it specifies a complete alternative for
         the element defined by the production rule. If the vertical
         bar appears in a portion of a formula enclosed in braces or
         square brackets, it specifies alternatives for the contents
         of the innermost pair of such braces or brackets.

    ...  The ellipsis indicates that the element to which it applies
         in a formula may be repeated any number of times. If the el-
         lipsis appears immediately after a closing brace "}", then it
         applies to the portion of the formula enclosed between that
         closing brace and the corresponding opening brace "{". If
         an ellipsis appears after any other element, then it applies
         only to that element.
```

# Drawbacks
[drawbacks]: #drawbacks

Why should we not do this? Comparing to the specification of `UPSERT` and `REPLACE` semantics in `INSERT`, the following features are missing:

* Doesn’t support specifying explicit constraints.
* Doesn’t support using conditions - the `WHERE` clause. 
* The item to be inserted/updated/replaced is the same as specified in the statement. However, `INSERT INTO ON CONFLICT DO UPDATE` or `DO REPLACE` can have different values for non-key attributes.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

* Why is this design/proposal the best in the space of possible designs? PartiQL already has `INSERT INTO ON CONFLICT DO UPDATE/DO REPLACE` for the insert-or-update, insert-or-replace functionalities. Here, the syntax is succinct and can provide more ergonomic choices for implementations that require to adopt the corresponding semantics as separate statements.
* What is the impact of not doing this? No service level or production impact as this is an addition to the PartiQL specification.

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal. A few examples of what this can include are:

* For specification proposals: Does this feature exist in any ISO SQL standard or other SQL dialects? The `UPSERT` and `REPLACE` syntax has already been used in Apache Impala [SQL](https://docs.cloudera.com/runtime/7.2.9/impala-sql-reference/topics/impala-upsert.html) and [Oracle (SQL for Oracle NoSQL)](https://docs.oracle.com/en/database/other-databases/nosql-database/18.3/sqlfornosql/adding-table-rows-using-insert-and-upsert-statements.html)

# Future possibilities
[future-possibilities]: #future-possibilities

In the future, we can extend the grammar, as explained in section 3.2 (Query), to any expression returning applicable PartiQL values; this will be more aligned with PartiQL/SQL++ generalization.
