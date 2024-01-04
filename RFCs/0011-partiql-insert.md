- Start Date: 2022-06-17
- PartiQL Issue [partiql/partiql-docs/#4](https://github.com/partiql/partiql-docs/issues/4)
- RFC PR: [partiql/partiql-docs/#11](https://github.com/partiql/partiql-docs/pull/11)

# Summary

This RFC specifies PartiQL's syntax and semantics for insert-or-update—also known as UPSERT—and insert-or-replace as part of PartiQL data manipulation language (DML). UPSERT is an informal term for a statement that denotes inserting or updating a set of data to a target database’s table based on existing availability condition of that data in the same table.

# Motivation

The PartiQL Specification as it stands today does not specify the syntax and semantics of data manipulation language (DML). The `INSERT` is one of the statements of the DML that requires specification; the motivation is to provide guidance through the specification for the DML insert-or-update and insert-or-replace implementations based on PartiQL.

# Guide-level explanation

The keywords “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this format specification are to be interpreted as described in [Key Words RFC2119](https://datatracker.ietf.org/doc/html/rfc2119).

## Terminology

* **Arbiter constraint**: A constraint that is used by an algorithm in a database system for inferring if data-conflict exists during data insertion.
* **Arbiter index**: A unique index that is used by an algorithm in a database system for inferring if data-conflict exists on a unique index during data insertion.
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

* Returning output from INSERT statement—the specification of returning output is considered as an extension to this RFC which is planned to be defined in the future.
* Constraints and unique index syntax and semantics—this is considered as a DDL feature as part of `CREATE INDEX` which requires definition in a separate RFC.
* Conflict detection based on index expression and index predicate—index expression and index predicate grammar and semantics are considered as a DDL feature as part of `CREATE INDEX` which require definition in a separate RFC.
* Type coercion implementations—the type coercion when inserting data from a source schema to a target schema with an expression (E.g. a sub-select statement) is considered as a database system implementation concern.
* Security privileges for execution of DML statements—the data access and manipulation security specification is considered as a database system specific feature, hence is considered out of scope for this RFC.

## 2. Proposed grammar and semantics

Figure 1. illustrates the BNF grammar corresponding to PartiQL’s `INSERT INTO ... ON CONFLICT` statements. See Appendix (1) for the Grammar conventions.

Note:

* All the definitions that are missing from the grammar refer to the definitions that are already in SQL-92 standard.

```EBNF
<insert statement> ::= INSERT INTO <table name> [ AS <alias> ] 
    [  ( <attr name> [, <attr name> ]... ) ]
        <values>
    [ ON CONFLICT [ <conflict target> ] <conflict action> ]

<values> ::= DEFAULT VALUES | <values clause>
      | <bag value> | <sub-select>

<values clause> ::= VALUES <value> [, <value>]...

<value> ::= ( { <value expr> | DEFAULT } [, { <value expr> | DEFAULT } ]...)

<conflict target> ::=
    ( <index target> [, <index target>]... )
    | ( { <primary key> | <composite primary key> } )
    | ON CONSTRAINT <constraint name>

<index target> ::= <index attr name>

<conflict action> ::= DO NOTHING 
    | DO UPDATE <do update> [ WHERE <condition> ]
    | DO REPLACE <do replace> [ WHERE <condition> ]

<do update> ::= EXCLUDED
    | SET <attr values> [, <attr values>]...

<do replace> ::= EXCLUDED
    | SET <attr values> [, <attr values>]...
    | VALUE <tuple value>

<attr values> ::=  {
            <attr name> = { <value expr> | DEFAULT }
            | ( <attr name> [, <attr name>]... ) = <value>
            | ( <attr name> [, <attr name>]... ) = ( <sub-select> )
        }

<primary key> ::= <attr name>

<composite primary key> ::= <attr name>, <attr name> [, <attr name> ]...

<value expr> ::= <partiql value>
     | <sql value expr>

<partiql value> ::= NULL
    | <ion value>
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

(* See sfw_query in PartiQL spec. figure 3: BNF Grammar for PartiQL Queries:
https://partiql.org/assets/PartiQL-Specification.pdf *)
<sfw query>

<index attr name> ::= <attr name>
<attr name> ::= <identifier>
<constraint name> ::= <identifier>

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

Implementation based on this specification MUST lead to atomic insertions of PartiQL values into a Schema (E.g. a database table). Atomic insertion means that either all or none of the insertions specified by the `INSERT` statement are performed—E.g. when there is an error in insertion of one or more data items.

### 3. 1 Attribute names

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

When attribute names are provided with `VALUES` or bag of lists, they MAY be listed in any order. Each attribute—required or optional—that is omitted in the provided attributes, 
MAY be filled with a default value, either its declared default value or an implementation of `NULL`—in case attribute is _nullable_. If the expression for any attribute is not of the correct data type, automatic type conversion i.e. type coercion MAY be attempted.

Section 3.3 provides more clarification with examples.

### 3.2 Query

`SELECT` query expression can be used for inserting the resulting values from the query to the target table. The resulting tuples (E.g. rows) MUST have the same attribute name and type as the target schema’s required attributes.

Optional attributes in target schema that have the same name in the resulting tuples from the source schema, SHOULD have the same type. If the implementing database provides type coercion rule for the types involved in source and destination, the `SELECT` MAY lead to a successful outcome, otherwise it's a `SemanticError`. See examples for more clarification.

#### 3.3 Examples

In the following examples, `Films` table has a closed schema. In the table `len` attribute is implicitly defined as nullable.

_In the following examples `CREATE TABLE` including `SCHEMA OPEN` and `SCHEMA CLOSED` syntax is arbitrary since the PartiQL DDL is yet to be defined—see the "Terminology" section for open and closed schema definitions._

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

```SQL
INSERT INTO Films
VALUES ('UA502', 'Bananas', 105, '1971-07-13', 'Comedy', '82 minutes');
```

In this example, the `len` attribute is omitted in the target attributes and the implementing database allows using default value or `NULL` for the omitted attributes in the target attributes.
In addition, implementing database sets the value of `len` to `NULL`— this is because based on the DDL for `Films`, `len` is nullable.

```SQL
INSERT INTO Films (code, title, did, date_prod, kind)
VALUES ('T_601', 'Yojimbo', 106, '1961-06-16', 'Drama');
```

The following examples uses the `DEFAULT` clause for the `date` and `len` attributes rather than specifying a value. 

For the second query, the `len` attribute is omitted in the target attributes and the implementing database allows using default value or `NULL` for the omitted attributes. Therefore, implementing database sets the value of `len` to `NULL`— this is because based on the DDL for `Films` `len` is nullable.

For the third query, the `kind` attribute is omitted in the target attributes and the order of attributes has changed. In this case, the implementing database allows using a different order and default value or `NULL` for the omitted attributes. Therefore, implementing database sets the value of `kind` to its default value 'Comedy':

```SQL
INSERT INTO Films
VALUES ('UA503', 'Bananas', 105, DEFAULT, 'Comedy', DEFAULT);

INSERT INTO Films (code, title, did, date_prod, kind)
VALUES ('T_603', 'Yojimbo', 106, DEFAULT, 'Drama');

INSERT INTO films (title, code, did, date_prod, len)
VALUES ('MyTitle', 'MyCode', 108, '1961-06-16', '180 minutes');

SELECT * FROM Films;
<<
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
}
>>
```

The following example statement inserts a row or item consisting entirely of default values:

```SQL
INSERT INTO Films DEFAULT VALUES;
```

The following example statement inserts multiple rows using the multi-row VALUES syntax:

```SQL
INSERT INTO Films (code, title, did, date_prod, kind)
VALUES ('B6717', 'Tampopo', 110, '1985-02-10', 'Comedy'),
       ('HG120', 'The Dinner Game', 140, DEFAULT, 'Comedy');
```

In the following statement, we insert values to `Music` table, `Music` table has a closed schema with `Artist` and `SongTitle` as required attributes:

```SQL
-- The following `CREATE TABLE` syntax is arbitrary since the PartiQL DDL is yet to be defined:
CREATE TABLE Music /* SCHEMA CLOSED */
(
   Artist     VARCHAR(20) NOT NULL,
   SongTitle  VARCHAR(30) NOT NULL,
   PRIMARY KEY (Artist, SongTitle)
);

INSERT INTO Music
<<
{'Artist' : 'Acme Band', 'SongTitle' : 'PartiQL Rocks'},
{'Artist' : 'Emca Band', 'SongTitle' : 'PartiQL Rocks'}
>>;
```

In the following statement we insert multiple person items as a bag value to `Person` table, `Person` table has an open schema with `LastName` and `DOB` as required attributes and `FirtName` as optional attribute.

```SQL
-- The following `CREATE TABLE` syntax is arbitrary since the PartiQL DDL is yet to be defined:
CREATE TABLE Person /* SCHEMA OPEN */
(
   LastName    VARCHAR(50) NOT NULL,
   FirstName   VARCHAR(20),    
   DOB        DATE NOT NULL,
   PRIMARY KEY (LastName)
);

INSERT INTO Person
<<
{'FirstName' : 'Raul',
'LastName' : 'Lewis',
'DOB' : 1963-08-19T,
'GovId' : 'LEWISR261LL',
'GovIdType' : 'Driver License',
},{'LastName' : 'Logan',
'DOB' : 1967-07-03T,
'Address' : '43 Stockert Hollow Road, Everett, WA, 98203'
},{'LastName' : 'Pena',
'DOB' : 1974-02-10T,
'GovId' : '744 849 301',
'GovIdType' : 'SSN',
'Address' : '4058 Melrose Street, Spokane Valley, WA, 99206'
}
>>;

SELECT * FROM Person;
<<
{'LastName' : 'Lewis',
'FirstName' : 'Raul',
'DOB' : 1963-08-19T,
'GovId' : 'LEWISR261LL',
'GovIdType' : 'Driver License',
},
{'LastName' : 'Logan',
'FirstName' : NULL,
'DOB' : 1967-07-03T,
'Address' : '43 Stockert Hollow Road, Everett, WA, 98203'
},
{'LastName' : 'Pena',
'FirstName' : NULL,
'DOB' : 1974-02-10T,
'GovId' : '744 849 301',
'GovIdType' : 'SSN',
'Address' : '4058 Melrose Street, Spokane Valley, WA, 99206'
}
>>;
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
INSERT INTO Foo
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

The following example statement inserts items from `RockAlbums` table into `Music` table. Both tables have the same layout for required and optional attributes and `Music` table has open schema, therefore the first `INSERT` statement can insert items using a `SELECT *` query. Furthermore, the second `INSERT` leads to a `SemanticError` because `RockAlbums` has closed schema.

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
INSERT INTO Music
SELECT *
FROM RockAlbums
WHERE RockGenre IN ('Alternative', 'SpaceRock');

-- The following leads to a `SemanticError` because `RockAlbums` has closed schema.
INSERT INTO RockAlbums
SELECT *
FROM Music;
```

The following statement shows that the values can be specified with a sub-select. The INSERT goes through because
the values that are not `SELECT` default to `NULL`.

```SQL
INSERT INTO Music (Artist, SongTitle, AlbumTitle)
SELECT Artist, SongTitle, AlbumTitle
FROM RockAlbums
WHERE RockGenre IN ('Alternative', 'SpaceRock');
```

The following example statement attempts to insert items from `RockAlbums` table to `Music` table. Both tables have the same layout for required attributes but have different types for `Year` optional attribute, therefore—if the implementing database system does not implement a type coercion rule from `DATE` to `INT`—the `INSERT` statement using a `SELECT` query leads to a `SemanticError`.

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

INSERT INTO Music
SELECT * FROM RockAlbums;
```

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

INSERT INTO Foo (id, title)
<<
[2, 'some-name'],
>>;

SELECT * FROM Foo;
<<
{ 'id': 2, 'is_deleted': false, 'title': 'some-name', 'bar': 'baz' }
>>;

INSERT INTO Foo
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
INSERT INTO Foo (id, title)
<<
{ 'id': 1 },
{ 'id': 2, 'title': 'some-name' },
{ 'id': 3, 'is_deleted': true, 'title': NULL, 'bar': '10'}
>>;

-- `SemanticError` because not all <bag value> items (row values) can get mapped to the target attributes set.
INSERT INTO Foo (id, title)
<<
[2, 'some-name'],
1,
'some-other-name'
>>;

-- `SemanticError` because INSERT has more target attributes than row values specified by some of the `<bag value>` elements (E.g. `[1]`).
INSERT INTO Foo (id, title)
<<
[1],
[1, 'some_name']
>>;

-- `SemanticError` because usage of `DEFAULT` outside `VALUES(...)` is unsupported.
INSERT INTO Foo (id, title)
<<
[1, DEFAULT],
[2, 'some-name']
>>;

-- `SemanticError` because usage of `DEFAULT` outside `VALUES(...)` is unsupported.
INSERT INTO Foo
<<
{'id': 1, 'is_deleted': DEFAULT},
{'id': 2, 'is_deleted': true}
>>;
```

### ON CONFLICT

`ON CONFLICT` can be used to specify an alternative action to raising a constraint violation error (E.g. unique constraint violation). In case target schema does not specify at least one constraint and ON CONFLICT is used, a `SemanticError` SHOULD be thrown by the implementing database system.

For each individual PartiQL value proposed for insertion, either the insertion proceeds, or, if an arbiter constraint or index is violated, the alternative `<conflict action>` is taken. The order of insertions is intentionally undefined in this specification and the specification leaves the definition and ordering logic to the implementing database.

`ON CONFLICT DO NOTHING` avoids inserting a PartiQL value as its alternative action.
`ON CONFLICT DO UPDATE` updates the conflicting PartiQL value with its alternative value. `ON CONFLICT DO REPLACE` replaces the conflicting PartiQL value with its alternative value.

`ON CONFLICT` implementation based on this specification MUST guarantee atomic outcomes—in case of an error the existing data MUST remain intact.

See `ON CONFLICT` clause in the “Parameters section” for more details and examples.

## 4. Parameters

### 4.1 Insert Parameters

This section covers parameters that may be used when only inserting new PartiQL values. Parameters exclusively used with the `ON CONFLICT` clause are described separately.

`<table name>` is the name of an existing table.

`<alias>` is a substitute name for `<table name>`. When an alias is provided, it SHOULD hide the actual name of the table. This is particularly for the instance when `ON CONFLICT` targets a database table named `EXCLUDED`. In such cases since `EXCLUDED` is also the name of the special table representing rows proposed for insertion, using the alias, disambiguates the name conflict—see the “Excluded keyword” section for more details.

`<attr name>` is the name of an attribute in the table named by `<table name>`. When referencing an attribute with `ON CONFLICT`, the table name or table alias (if defined) MUST get omitted from the left-hand side of the specification of a target attribute. For example, `INSERT INTO table_name ... ON CONFLICT DO UPDATE SET table_name.attr = 1` is invalid.

`DEFAULT VALUES` denotes that all required attributes MUST be filled with their default values or `NULL` in case of attributes being nullable. This is as if `DEFAULT` were explicitly specified for each attribute. If the target database system has no concept or implementation of default values, this should return a `SemanticError` exception.

`<value expr>` is a value expression (E.g. `1` or `2+2`) that can get assigned to the corresponding attribute.

`DEFAULT` denotes the corresponding attribute gets filled with its default value. For a generated attribute (E.g. auto-increments), specifying this is permitted which SHOULD result in computing the attribute value from its data-generation expression.

### 4.2 ON CONFLICT Clause

`<conflict target>` is an optional parameter that specifies which conflicts `ON CONFLICT` takes the alternative action on. This SHOULD be implemented by choosing primary key, arbiter index(es), or specifying constraint using `ON CONSTRAINT` clause. When omitted, there MUST be an inference logic on the implementing database system to unambiguously determine if a constraint is being violated by the values specified in the `INSERT` statement. If the implementing database can’t guarantee this inference a `SemanticError` MUST be returned in case `<conflict target>` is omitted.
When provided, `<conflict target>` MUST either be used to infer primary key, unique indexes, or specify a constraint. For unique indexes it should be used to perform unique index inference using the provided index attribute names. All `<table name>` unique indexes that, without regard to order, contain exactly the `conflict target`–specified attributes should be inferred (chosen) as arbiter indexes. If an attempt at inference is unsuccessful, a `SemanticError` MUST be raised.

For both `ON CONFLICT DO UPDATE` and `ON CONFLICT DO REPLACE`, when conflict occurs on a primary key or a constraint and the proposed value for `UPDATE` or `REPLACE` is a `<tuple value>` — attributes comprising the primary key, or the constraint MUST exist in the proposed `<tuple value>`, otherwise a `SemanticError` MUST be raised. See “ON ** CONFLICT DO UPDATE Examples” section for more clarification.

Furthermore, for both `ON CONFLICT DO UPDATE` and `ON CONFLICT DO REPLACE`, the order of insertions for conflict resolution is intentionally undefined in this specification and the specification leaves the definition and the ordering logic to the implementing database.

#### Excluded keyword

In addition to the existing rows or items, `ON CONFLICT` and `WHERE` clauses have read-only access to rows/items proposed for insertion using the special `EXCLUDED` keyword.

*Note: The `EXCLUDED` is chosen to reduce the learning overhead as it’s a common term across PostgresSQL and SQLLite.*

`<conflict action>` specifies an alternative `ON CONFLICT` action. It can be either `DO NOTHING`, `DO UPDATE`, or `DO REPLACE`. The `ON CONFLICT` and `WHERE` clauses have read-only access to the existing rows or items using the table's name (or an alias). In case of existence of SQL triggers or equal event-driven procedures, the effects of those per-row/per-item triggers/operations MUST be reflected in `EXCLUDED` values, since those effects may have contributed to the row being excluded from insertion.

`<primary key>` explicitly specifies the `<table name>`'s primary key.

`<composite primary key>` explicitly specifies `<table name>`'s composite primary key—E.g. a combination for `partition_key_sort_key` for an implementation of a NoSQL database system.

`<index attr name>` denotes the name of a `<table name>` attribute used to infer arbiter indexes.

`<constraint name>` explicitly specifies an arbiter constraint by name, rather than inferring a constraint from a primary key or index.

`<condition>` is an expression that MUST return a boolean value `TRUE` or `FALSE`. Only rows or items for which this expression returns `TRUE` MUST be updated. Note that `<condition>` MUST be evaluated last; after a conflict has been identified as a candidate to update or replace.

Finally, `INSERT` with an `ON CONFLICT DO UPDATE` or `ON CONFLICT DO REPLACE` clause MUST be a “deterministic” statement. This means that the command MUST not be allowed to affect any single existing row or item more than once; a `SemanticError` (E.g. cardinality violation error) MUST be raised when this situation arises. Rows resulting from insertion after conflict, should not duplicate each other in terms of attributes constrained by an arbiter index or constraint. See “ON CONFLICT DO REPLACE Examples” section for more clarification.

#### ON CONFLICT DO UPDATE Examples
_In the following examples `CREATE TABLE` including `SCHEMA OPEN` and `SCHEMA CLOSED` syntax is arbitrary since the PartiQL DDL is yet to be defined—see the "Terminology" section for open and closed schema definitions._

##### Example 4.2.1

Inserts or updates new distributors in a relational database. Assumes a unique index has been defined that constrains values appearing in the `did` column. Note that the special `EXCLUDED` table is used to reference values originally proposed for insertion.

```SQL
-- The following `CREATE TABLE` syntax is arbitrary since the PartiQL DDL is yet to be defined:
CREATE TABLE Distributors /* SCHEMA CLOSED */
(
    did       INT     NOT NULL PRIMARY KEY,
    dname     VARCHAR(50),
);
```

```SQL
INSERT INTO Distributors
VALUES (5, 'Gizmo Transglobal'),
       (6, 'Associated Computing, Inc') ON CONFLICT (did) DO
UPDATE SET dname = EXCLUDED.dname;
```

```SQL
INSERT INTO Distributors AS e
VALUES
    (5, 'Gizmo Transglobal'),
    (6, 'Associated Computing, Inc')
ON CONFLICT (did) DO UPDATE SET dname = e.dname;
```

The following example leads to a `SemanticError` because the usage of `<table name>` or `<alias>` is disallowed on the left-hand side of `e.dname = e.dname` expression.
```SQL
INSERT INTO Distributors AS e
VALUES
    (5, 'Gizmo Transglobal'),
    (6, 'Associated Computing, Inc')
ON CONFLICT (did) DO UPDATE SET e.dname = e.dname;
```

The following example leads to a `SemanticError` because the usage of `<table name>` alongside `<alias>` is disallowed—see "4.1 Insert Parameters" `<alias>` for more details:
```SQL
INSERT INTO Distributors AS e
VALUES
    (5, 'Gizmo Transglobal'),
    (6, 'Associated Computing, Inc')
ON CONFLICT (did) DO UPDATE SET dname = Distributors.dname;
```

The following succeeds because `<table name>` is used on the right-hand side of `ON CONFLICT` target and `<alias>` is not specified for the table:
```SQL
INSERT INTO Distributors
VALUES
    (5, 'Gizmo Transglobal'),
    (6, 'Associated Computing, Inc')
ON CONFLICT (did) DO UPDATE SET dname = Distributors.dname;
```

##### Example 4.2.2

Inserts or updates an item into a NoSQL database. Assumes a unique constraint e.g. a primary key has been violated for an existing item. In this case `DO UPDATE SET` updates the item with an additional attribute.

```SQL
-- The following `CREATE TABLE` syntax is arbitrary since the PartiQL DDL is yet to be defined:
CREATE TABLE Customers /* SCHEMA OPEN */
(
    HK      INT     NOT NULL PARTITION KEY,
    RK      INT     NOT NULL SORT KEY,
);
```

```SQL
-- Existing Item with HK as primary key: {'HK': 1, 'RK': 1, 'myOtherAttr': 5}
-- Item after the update:  {'HK': 1, 'RK': 1, 'myOtherAttr': 5, 'myAttr': 1}

INSERT INTO Customers <<
{'HK': 1, 'RK': 1}
>>
ON CONFLICT DO
UPDATE SET myAttr = 1;
```

##### Example 4.2.3

Inserts or updates an item into NoSQL database. Assumes a unique constraint like primary key has been violated for an existing item. In this case `DO UPDATE SET` merges the existing item with the value specified in `INSERT`.

```SQL
-- Existing Item with HK as primary key:
--  {'HK': 1, 'RK': 1, 'myAttr': 10}
-- Item after the update:
--  {'HK': 1, 'RK': 1, 'myAttr': 12, 'anotherAttr': 'hello'}

INSERT into Customers
<<
{'HK': 1, 'RK': 1, 'myAttr': 12, 'anotherAttr': 'hello'}
>>
ON CONFLICT DO
UPDATE EXCLUDED
```

##### Example 4.2.4

Inserts or updates two items into a NoSQL database. Assumes a unique constraint like primary key has been violated for an existing item.

In this case the `UPSERT` succeeds; this is because, although `EXCLUDED` references an attribute which is non-existent in `EXCLUDED`, but the implementing database allows the existence of PartiQL value `MISSING` in the schema—the same
query can lead to a `SemanticError` if the implementing database disallows existence of PartiQL value `MISSING` in the schema.

```SQL
-- Existing Item with HK as primary key: 
--  {'HK': 1, 'RK': 1, 'myAttr': 10}
-- Items after the update: 
--  {'HK': 1, 'RK': 1, 'myAttr': MISSING, 'newAttr': 'World'}
--  {'HK': 4, 'RK': 1, 'someAttr': 'Foo'}

INSERT into Customers
<<
{'HK': 4, 'RK': 1, 'someAttr': 'Foo'},
{'HK': 1, 'RK': 1, 'myAttr': 12, 'anotherAttr': 'hello'}
>>
ON CONFLICT
    DO
UPDATE SET myAttr = EXCLUDED.someAttr, newAttr = 'World';
```

In the following example, the insertion leads to a `SemanticError`, because irrespective of implementing database allowing or disallowing existence of PartiQL value `MISSING` in the schema, because the schema is closed, no new attribute can be added to the schema:

```SQL
-- The following `CREATE TABLE` syntax is arbitrary since the PartiQL DDL is yet to be defined:
CREATE TABLE Orders /* SCHEMA CLOSED */
(
    OrderId          INT     NOT NULL PARTITION KEY,
    OrderVolume      INT     NOT NULL SORT KEY,
);

-- Existing Item with HK as primary key: 
--  {'OrderId': 1, 'OrderVolume': 1200, 'myAttr': 10}
-- Item after the update: 
--  {'OrderId': 1, 'OrderVolume': 1200, 'myAttr': 10}

INSERT into Customers
<<
{'OrderId': 4, 'OrderVolume': 2300, 'someAttr': 'Foo'},
{'OrderId': 1, 'OrderVolume': 1400, 'myAttr': 12, 'anotherAttr': 'hello'}
>>
ON CONFLICT
    DO
UPDATE SET myAttr = EXCLUDED.someAttr, newAttr = 'World';
```

*Note: In the above example before and after items are the same because an error is raised.*

##### Example 4.2.5

Inserts or updates an item into a NoSQL database. Assumes a unique constraint like primary key has been violated for an existing item. In this case `DO UPDATE SET` updates the item with an additional attribute and keeps the previous attributes.

```SQL
-- Existing Item with HK and RK form a composite primary key:
--  {'HK': 1, 'RK': 1, 'myAttr': 10}
-- Item after the update: 
--  {'HK': 1, 'RK': 1, 'myAttr': 12, 'newAttr' = 'World'}

INSERT into Customers
<<{'HK': 1, 'RK': 1, 'myAttr': 12, 'anotherAttr': 'hello'}>>
ON CONFLICT
    DO
UPDATE SET myAttr = EXCLUDED.myAttr, newAttr = 'World';
```

##### Example 4.2.6

Inserts or updates an item into a NoSQL database. Assumes a unique constraint like primary key has been violated for an existing item. In this case `DO UPDATE SET` does not update the existing item because the condition in `WHERE` clause is not met:

```SQL
-- Existing Item with HK as primary key:
--   {'HK': 1, 'RK': 1, 'myAttr': 10}
-- Item after the execution:
--   {'HK': 1, 'RK': 1, 'myAttr': 10}

INSERT into Customers AS CX 
<<{'HK': 1, 'RK': 1, 'myAttr': 12, 'anotherAttr': 'hello'}>>
ON CONFLICT
    DO
UPDATE SET myAttr = CX.myAttr, newAttr = 'World'
WHERE CX.myAttr > 10;
```

#### ON CONFLICT DO REPLACE Examples

For the following examples we assume the existence of composite key `partition_key_sort_key` comprising `HK` and `RK` attributes i.e `HK_RK`.

##### Example 4.2.7

The following example leads to a `SemanticError` exception because `sort_key` is absent from the value specified by the `REPLACE`.

```SQL
-- Existing Item is: 
--  {'HK': 1, 'RK': 1, 'myAttr': 12}
-- Outcome is a `SemanticError` with the existing item being intact.

INSERT into Customers
<<{'HK': 1, 'RK': 1, 'myAttr': 12, 'anotherAttr': 'hello'}>>
ON CONFLICT
    DO REPLACE VALUE {'HK': 1, 'thirdAttr': 'world'};
```

##### Example 4.2.8

The following example leads to a `SemanticError` exception because partition key is absent from the value specified by replace.

```SQL
-- Existing Item is {'HK': 1, 'RK': 1, 'myAttr': 12}
-- Outcome is a `SemanticError` with the existing item being intact.

INSERT into Customers
<<{'HK': 1, 'RK': 1, 'myAttr': 12, 'anotherAttr': 'hello'}>>
ON CONFLICT
    DO REPLACE VALUE {'RK': 1, 'thirdAttr': 'world'};
```

##### Example 4.2.9

The following example leads to a `SemanticError` exception because both partition key and sort_key are absent in the update statement:

```SQL
-- Existing Item is { 'HK': 1, 'RK': 1, 'myAttr': 12 }
-- Outcome is a SemanticError
INSERT into Customers
<<{'HK': 1, 'RK': 1, 'myAttr': 12, 'anotherAttr': 'hello'}>>
ON CONFLICT
    DO REPLACE VALUE {'thirdAttr': 'world'};
```

##### Example 4.2.10

The following example leads to replacement of the item specified by replace with the existing one:

```SQL
-- Existing Item is:
--  {'HK': 1, 'RK': 1, 'myAttr': 12 }
-- Outcome is:
--  {'HK': 1, 'RK': 1, 'thirdAttr': 'world'}

INSERT into Customers
<<{'HK': 1, 'RK': 1, 'myAttr': 12, 'anotherAttr': 'hello'}>>
ON CONFLICT
    DO REPLACE VALUE {'HK': 1, 'RK': 1, 'thirdAttr': 'world'};
```

##### Example 4.2.11

The following example SHOULD* lead to replacing the existing item with the item specified by `REPLACE` as a new item:

```SQL
-- Existing items are:
--  {'HK': 1, 'RK': 1, 'myAttr': 12 }, 
--  {'HK': 1, 'RK': 2, 'myAttr': 12 }
-- Outcome:
--  {'HK': 1, 'RK': 3, 'thirdAttr': 'World'}, 
--  {'HK': 1, 'RK': 2, 'myAttr': 12 }

INSERT into Customers
<<{'HK': 1, 'RK': 1, 'myAttr': 12, 'anotherAttr': 'hello' }>>
ON CONFLICT
    DO REPLACE VALUE {'HK': 1, 'RK': 3, 'thirdAttr': 'world'};
```

** SHOULD, because for a distributed database system, implementing this logic may lead to data inconsistency. This is because there is no guarantee that the INSERT procedure that generates the conflict executes on the same database host where the conflict is going to get resolved.

##### Example 4.2.12

The following example MUST lead to a `SemanticError` if there is an existing item that conflicts with the item specified by replace:

```SQL
-- Existing items is:
--  {'HK': 1, 'RK': 1, 'myAttr': 12 },
--  {'HK': 1, 'RK': 2, 'myAttr': 12 }
-- Outcome is SemanticError

INSERT into Customers
<<{ 'HK': 1, 'RK': 1, 'myAttr': 12, 'anotherAttr': 'hello' }>>
ON CONFLICT
    DO REPLACE VALUE {'HK': 1, 'RK': 2, 'thirdAttr': 'world'};
```

##### Example 4.2.13

The following example replaces the existing item with item provided by `INSERT` statement:

```SQL
-- Existing items is:
--  {'HK': 1, 'RK': 1, 'myAttr': 12 },
-- Outcome is:
--  { 'HK': 2, 'RK': 1, 'myAttr': 12, 'anotherAttr': 'hello' }

INSERT into Customers
<<{ 'HK': 2, 'RK': 1, 'myAttr': 12, 'anotherAttr': 'hello' }>>
ON CONFLICT
    DO REPLACE EXCLUDED;
```

##### Example 4.2.14

The following example conditionally replaces the existing item with the item provided by `INSERT` statement. Note that as an outcome the statement does not replace the existing item because the condition is not met.

```SQL
-- Existing items is:
--  {'HK': 1, 'RK': 1, 'myAttr': 5 },
-- Outcome is:
--  {'HK': 1, 'RK': 1, 'myAttr': 5 },

INSERT into Customers
<<{ 'HK': 2, 'RK': 1, 'myAttr': 13, 'anotherAttr': 15 }>>
ON CONFLICT
    DO REPLACE EXCLUDED WHERE EXCLUDED.anotherAttr = 12;
```

## 5. Other Examples
Insert or update new distributors as appropriate. Assumes a unique index has been defined that constrains values appearing in the `did` attribute. Note that the special `excluded` table is used to reference values originally proposed for insertion:

```SQL
INSERT INTO Distributors (did, dname)
VALUES (5, 'Gizmo Transglobal'),
       (6, 'Associated Computing, Inc') ON CONFLICT (did) 
       DO UPDATE SET dname = EXCLUDED.dname;
```

Insert or update new distributors values as bag value. Assumes the database engine implementation can infer the primary key for `<conflict target>`. Note that the special `EXCLUDED` table is used to reference values originally proposed for insertion:

```SQL
INSERT INTO Distributors
<<
{'id': 5, dname: 'Gizmo Transglobal'},
{'id': 6, dname: 'Associated Computing, Inc'}
>>
ON CONFLICT
    DO
UPDATE SET dname = EXCLUDED.dname;
```

Insert distributors values as bag value. In case of a conflict on unique constraint merges the existing item with its corresponding tuple in `INSERT`. Assumes the database engine implementation can infer the unique index on table `distributors`. Note that the special `excluded` table is used to reference values originally proposed for insertion:

```SQL
INSERT INTO distributors
<<
{'id': 5, dname: 'Gizmo Transglobal'},
{'id': 6, dname: 'Associated Computing, Inc'}
>>
ON CONFLICT
    DO
UPDATE EXCLUDED;
```

## References

* [Database Systems: The Complete Book, second Edition](https://www.goodreads.com/book/show/6608398-database-systems), Hector Garcia-Molina Jeffrey D. Ullman Jennifer Widom *Department of Computer Science Stanford University.*
* [Designing Data-Intensive Applications](https://www.goodreads.com/book/show/23463279-designing-data-intensive-applications), by Martin Kleppmann
* [Postgres INSERT ref](https://www.postgresql.org/docs/current/sql-insert.html)
* [Postgres constraints](https://www.tutorialspoint.com/postgresql/postgresql_constraints.htm)
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

#### 2. PostgresQL query examples

1. ON CONFLICT using ON CONSTRAINT:
    1. https://www.db-fiddle.com/f/pdzztVpZqAreALu5Wquwrw/0
    2. https://www.db-fiddle.com/f/pdzztVpZqAreALu5Wquwrw/1 (with same primary key)
2. UPSERT with auto-increment postgres:
    1. https://www.db-fiddle.com/f/gWBLQ4NXhhxu7KTg9M6JwB/0
    2. https://www.db-fiddle.com/f/gWBLQ4NXhhxu7KTg9M6JwB/2 (with existing key in UPDATE)
    3. https://www.db-fiddle.com/f/gWBLQ4NXhhxu7KTg9M6JwB/3 (replacing an existing item)
3. UPSERT with EXCLUDED in postgres:
    1. https://www.db-fiddle.com/f/nP56Wo66Qj1Bswsx3a8GxU/0
4. UPSERT with multiple values in postgres:
    1. https://www.db-fiddle.com/f/nP56Wo66Qj1Bswsx3a8GxU/1
    2. https://www.db-fiddle.com/f/nP56Wo66Qj1Bswsx3a8GxU/2
5. ON CONFLICT Error for a table with no constraints:
    1. https://www.db-fiddle.com/f/gWBLQ4NXhhxu7KTg9M6JwB/6
6. ON CONFLICT w/ multi-column:
    1. https://www.db-fiddle.com/f/nP56Wo66Qj1Bswsx3a8GxU/3
7. ON CONFLICT w/ multi-column and sub-select:
    1. https://www.db-fiddle.com/f/nP56Wo66Qj1Bswsx3a8GxU/4
8. Postgres index_expression example:
    1. https://www.db-fiddle.com/f/gWBLQ4NXhhxu7KTg9M6JwB/7
9. PostgresSQL multi-column update discussion:
    1. https://www.postgresql.org/message-id/87muw0tfml.fsf%40news-spur.riddles.org.uk
10. PostgresSQL `INSERT INTO` with default values:
    1. https://www.db-fiddle.com/f/kHZSgG9F5EjWPpu99ezLRo/2
    2. https://www.db-fiddle.com/f/rDRSMrUQKJ6D6vEgte6Q58/2
    3. https://www.db-fiddle.com/f/ePbzN3JT2GSgZbvKDULjBe/1
    4. https://www.db-fiddle.com/f/dJ4Go1TW3NfDCGa57Vhc6a/0 (with a different field order)
    5. https://www.db-fiddle.com/f/ePbzN3JT2GSgZbvKDULjBe/0 (an erroneous query)

# Drawbacks

Why should we *not* do this?
One reason for not taking this approach is the statement complexity for `UPSERT` and `REPLACE` scenarios. Considering this, having the `INSERT` statement incorporating both `UPSERT` and `REPLACE` leaves the room open for rewrites from other statements designated for `UPSERT` and `REPLACE` separately.

# Rationale and alternatives

* Why is this design/proposal the best in the space of possible designs? The `INSERT INTO ... ON CONFLICT` statement is chosen based on the assessment of both relational and non-relational use-cases and the need for being able to upsert in the absence of a source table. We believe `INSERT INTO ... ON CONFLICT` as adopted by PostgresSQL and SQLite allows for incorporating attribute level and row-level atomic operations with the option of re-using the values used for insertion (by using EXCLUDED keyword). On the other hand, [Merge](https://en.wikipedia.org/wiki/Merge_%28SQL%29#Other_non-standard_implementations) as defined by SQL standard require a table as a source which might not be required in all cases. In addition, arguably, among other alternatives there isn't necessarily a "best" option available. Finally, this is not a one-way door decision because so long as the semantics remain the same, other statements can be added to the language with possible rewrites to this reference syntax. For example, we can have a syntax solely for UPSERT or another one for REPLACE.
* Which other designs/proposals have been considered, and what is the rationale for not choosing them? SQL [Merge](https://en.wikipedia.org/wiki/Merge_%28SQL%29#Other_non-standard_implementations) statement has been considered but considering that it requires a source table it has been discarded.
* What is the impact of not doing this? No service level or production impact as this is an addition to the PartiQL specification.

# Prior art

Discuss prior art, both the good and the bad, in relation to this proposal. A few examples of what this can include are:

* For specification proposals: Does this feature exist in any ISO SQL standard or other SQL dialects? As mentioned in prior section, the `INSERT INTO ... ON CONFLICT` has already been used in PostgresSQL and SQLite. Generally the `UPSERT` functionality which is formally known as [Merge](https://en.wikipedia.org/wiki/Merge_%28SQL%29#Other_non-standard_implementations) already exists in SQL Standard and has been implemented in various databases systems—both relational as NoSQL systems.

# Unresolved questions

# Future possibilities

In the future, we may be able to accommodate `DO DELETE` conflict action which specifies the delete operation if a conflict is raised. In addition, we see addition of specification for other syntactic variation of the semantics specified by this RFC using designated syntax for `UPSERT` and `REPLACE` operations separately. This is to provide more ergonomic choices for implementations that require to adopt the corresponding semantics as separate statements.

In addition, we can extend the grammar, as explained in section 3.2 (Query), to any expression returning applicable PartiQL values; this will be more aligned with PartiQL/SQL++ generalization.
