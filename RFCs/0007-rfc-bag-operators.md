- Feature Name: Bag Operators: UNION, INTERSECT, EXCEPT
- Start Date: 2022-06-07

# Summary
[summary]: #summary

This RFC defines the PartiQL specification for the operators: UNION, INTERSECT, and EXCEPT. The behavior of each operator conforms to the SQL standard specification. Additionally, this RFC defines three new operators: OUTER UNION, OUTER INTERSECT, and OUTER EXCEPT which act on arbitrary values by coercing values to bags. For all operators, member equality is consistent with the *GROUP BY* clause.

# Motivation
[motivation]: #motivation

The operators UNION, INTERSECT, and EXCEPT have been defined since SQL-92 specifications, yet these are missing from the PartiQL specification. These operators are the typical SQL `<non-join query expression>`s to combine queries for compatible relations. Two relations *R1* and *R2* are compatible for set operators if they have the same number of columns and the *i*-th column of *R1* is comparable to the *i*-th column of *R2*. Comparability in PartiQL conforms to the SQL specification and is defined in Appendix D. SQL compatibility for set operators is discussed in the following section, and a detailed look can be found in Appendix A.

The additional *OUTER* variants of each set operator are an extension to the SQL standard and are combined without compatibility concerns. These variants allow for combining any value by coercing the arguments to a bag. A scalar value is coerced into a singleton bag; a list is coerced to a bag by discarding ordering; NULL and MISSING are coerced into the empty bag.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## Definition

The bag operators are
```
- [OUTER] UNION     [ALL|DISTINCT]
- [OUTER] INTERSECT [ALL|DISTINCT]
- [OUTER] EXCEPT    [ALL|DISTINCT]
```
> 

Each bag operator has the form *q S q'* where *q* and *q'* are of type *\<query\>* and *S* is the operator. Additionally, the operator may be suffixed with *ALL* which indicates the output may have duplicate elements. In its absence, *DISTINCT* is implicit and duplicates are eliminated from the final result.

Let *T1* and *T2* be two compatible relations, and let *TR* be the result of a set operator. The standard SQL bag operators are defined as:

```
T1 UNION ALL T2     = SQL_UNION(T1, T2)
T1 INTERSECT ALL T2 = SQL_INTERSECT(T1, T2)
T1 EXCEPT ALL T2    = SQL_EXCEPT(T1, T2)

T1 UNION DISTINCT T2     = DISTINCT(SQL_UNION(T1, T2))
T1 INTERSECT DISTINCT T2 = DISTINCT(SQL_INTERSECT(T1, T2))
T1 EXCEPT DISTINCT T2    = DISTINCT(SQL_EXCEPT(T1, T2))
```

The SQL operators are defined in terms of: let *R* be some row in *T1* or *T2* or both. Let *m* be the number of duplicates of *R* in *T1* and let *n* be the number of duplicates of *R* in *T2* with *m* >= 0 and *n* >= 0.

For *SQL_UNION*, the number of duplicates of *R* in *TR* will be *(m + n)*.

For *SQL_INTERSECT*, the number of duplicates of *R* in *TR* will be the minimum of *m* and *n*.

For *SQL_EXCEPT*, the number of duplicates of *R* in *TR* will be the maximum of *(m - n)* and *0*.

Let *V1* and *V2* be arbitrary values, and let *F* be a function which coerces a value to a bag. The *OUTER* operators are defined as
```
V1 OUTER UNION ALL V2     = SQL_UNION(F(V1), F(V2))
V1 OUTER INTERSECT ALL V2 = SQL_INTERSECT(F(V1), F(V2))
V1 OUTER EXCEPT ALL V2    = SQL_EXCEPT(F(V1), F(V2))

V1 OUTER UNION DISTINCT V2     = DISTINCT(SQL_UNION(F(V1), F(V2)))
V1 OUTER INTERSECT DISTINCT V2 = DISTINCT(SQL_INTERSECT(F(V1), F(V2)))
V1 OUTER EXCEPT DISTINCT V2    = DISTINCT(SQL_EXCEPT(F(V1), F(V2)))
```

The coercion function *F* is defined for all PartiQL values (Appendix C) by:
```
F(absent_value) -> << >>               
F(scalar_value) -> << scalar_value >> # singleton bag
F(tuple_value)  -> << tuple_value >>  # singleton bag, see future extensions
F(array_value)  -> bag_value          # discard ordering
F(bag_value)    -> bag_value          # identity
```

### SQL Compatibility

In the presence of schema, the column descriptors for a table are known. These column descriptors are an ordered list of metadata objects which describe a column's: type, name, nullability, and more (see Appendix A). The column descriptors play an important role for set operators. Two relations *T1* and *T2* are said to be compatible if they have the same number of columns (\*) and the data type for the i-th column of *T1* and the data type for the i-th column of *T2* are comparable. Comparability and the resultant data type follow the same rules as the comparability rules and data types of results of aggregations defined in SQL-99 section 9.3.  

Each set operator produces a new relation *TR* which has its own column descriptors. The exact details of the these column descriptors is discussed in Appendix A, as well as SQL-99 section 7.12. The specification states that, for SQL conformance:

**Column Naming Rule**
- If the i-th columns of *T1* and *T2* share the name *C*, then the i-th column of *TR* will be named *C*.
- Otherwise, the name of the i-th column of *TR* is implementation-dependent and must be unique amongst all other column names.

> Note: Postgres, SQLite, and MySQL all use the name of the left-most relation's column.

**Nullability Rule**
- The i-th column of *TR* is not nullable only if both the i-th columns of *T1* and *T2* are known not nullable. Else, the i-th column of *TR* is possibly nullable.
- If the set operator is *EXCEPT*, then the i-ith column of *TR* is nullable only if the i-th column of *T1* is nullable.

These rules tell us, in the presence of schema, the set operators are valid when the argument relations are set operator compatible; and the resultant column descriptors inherit their names and nullability from the column descriptors of the arguments.

- \* If the statement contains a *CORRESPONDING* clause, then the column descriptors of *T1* and *T2* will be compared by the declared order of column names in the *CORRESPONDING* list.

## Examples

### Guide

1. Union of Compatible Relations
2. Union of Compatible Relations; Mismatch Column Names
  1. Without *CORRESPONDING* Clause
  2. With *CORRESPONDING* Clause
  3. Using *OUTER UNION*
3. Union of Heterogeneous Relations
4. Intersection of Compatible Relations
5. Except of Compatible Relations
6. Value Coercion
7. NULL and MISSING Coercion

### Environment

```sql
-- Each bag of employees is *almost* the same. Let's see how we can compose them.

!add_to_global_env { 
  'hr': { 
    'employees': <<
      { 'name': 'Bob Smith',   'title': 'Consultant' }, 
      { 'name': 'Susan Smith', 'title': 'Manager' },
      { 'name': 'Jane Smith',  'title': 'DEI' }
    >>
  },
  'accounting': { 
    'employees': <<
      { 'level': 3, 'name': 'Quincy Jones',   'title': 'Director' }, 
      { 'level': 2, 'name': 'James Earl Jones', 'title': 'Manager' },
      { 'level': 1, 'name': 'Indiana Jones',  'title': 'Intern' }
    >>
  },
  'engineering': { 
    'employees': <<
      { 'title': 'Manager', 'name': 'Paul McCharmly' }, 
      { 'title': 'SDE', 'name': 'George Parasol' },
      { 'title': 'Principal SDE', 'name': 'Ringo Stone' },
      { 'title': 'Intern', 'name': 'Eric Lennon' }
    >>
  }
} 
```

### Column Descriptors

The static types of *hr.employees*, *accounting.employees*, and *engineering.employees* relations are represented by the column descriptors below.

```
              hr.employees

| Ordinal | Name  | Type   | Nullable |
|---------|-------|--------|----------|
| 0       | NAME  | String | NO       |
| 1       | TITLE | String | NO       |

        accounting.employees

| Ordinal | Name  | Type   | Nullable |
|---------|-------|--------|----------|
| 0       | LEVEL | Int    | NO       |
| 1       | NAME  | String | NO       |
| 2       | TITLE | String | NO       |

        engineering.employees

| Ordinal | Name  | Type   | Nullable |
|---------|-------|--------|----------|
| 0       | TITLE | String | NO       |
| 1       | NAME  | String | NO       |
```

### Examples

#### Example 1 — Union of Compatible Relations

```sql
SELECT name, title FROM hr.employees
UNION ALL
SELECT name, title FROM engineering.employees

<<
  {
    'name': 'Bob Smith',
    'title': 'Consultant'
  },
  {
    'name': 'Susan Smith',
    'title': 'Manager'
  },
  {
    'name': 'Jane Smith',
    'title': 'DEI'
  },
  {
    'name': 'Paul McCharmly',
    'title': 'Manager'
  },
  {
    'name': 'George Parasol',
    'title': 'SDE'
  },
  {
    'name': 'Ringo Stone',
    'title': 'Principal SDE'
  },
  {
    'name': 'Eric Lennon',
    'title': 'Intern'
  }
>>
-- OK!
```

#### Example 2 — Union of Compatible Relations; Mismatch Column Names

Here is **Example 1** but the **names** of the column descriptors do not match. Notice the lack of selection list here. These two relations are union-compatible because the i-th column of *hr.employees* is comparable to the i-th column of *engineering.employees*, but notice the column names. This behavior conforms to SQL specification whereby, if the column names do not match, then the column name is implementation-dependent i.e. it is not derived from any particular value. The column names are taken from the left-most relation.

**Example 2.1** — Without *CORRESPONDING* Clause

```sql
SELECT * FROM hr.employees
UNION ALL
SELECT * FROM engineering.employees

<<
  {
    'name': 'Bob Smith',
    'title': 'Consultant'
  },
  {
    'name': 'Susan Smith',
    'title': 'Manager'
  },
  {
    'name': 'Jane Smith',
    'title': 'DEI'
  },
  {
    'name': 'Manager',          -- !! column name from left-most relation
    'title': 'Paul McCharmly'
  },
  {
    'name': 'SDE',              -- !!
    'title': 'George Parasol'
  },
  {
    'name': 'Principal SDE',    -- !!
    'title': 'Ringo Stone'
  },
  {
    'name': 'Intern',           -- !!
    'title': 'Eric Lennon'
  }
>>
-- OK!

-- Result Column Descriptors
-- | Ordinal | Name  | Type   | Nullable |
-- |---------|-------|--------|----------|
-- | 0       | name  | String | NO       |
-- | 1       | title | String | NO       |
```

Notice that we have some issues with the meaning of our keys/columns. For the first three structs/tuples, the *0*-th column contains a name and *1*-th column contains a title; yet it is the opposite for the last four structs/tuples. We can fix this by using the SQL *CORRESPONDING* clause.

**Example 2.2** — With *CORRESPONDING* Clause

```sql
SELECT * FROM hr.employees
UNION ALL CORRESPONDING BY ( name, title )
SELECT * FROM engineering.employees

-- Equivalent

TABLE hr.employees
UNION ALL CORRESPONDING BY ( name, title )
TABLE engineering.employees

-- Equivalent, *Example 1*

SELECT name, title FROM hr.employees
UNION ALL
SELECT name, title FROM engineering.employees
```

**Example 2.3** — Using *OUTER UNION*

Here is Example 2.1 but using *OUTER UNION*. In this case, we are not concerned about relation compatibility because the *OUTER UNION* is simply the union of the bags of tuples.

```sql
SELECT * FROM hr.employees
OUTER UNION ALL
SELECT * FROM engineering.employees

-- Note the tuple keys
<<
  {
    'name': 'Bob Smith',
    'title': 'Consultant'
  },
  {
    'name': 'Susan Smith',
    'title': 'Manager'
  },
  {
    'name': 'Jane Smith',
    'title': 'DEI'
  },
  {
    'title': 'Manager',
    'name': 'Paul McCharmly'
  },
  {
    'title': 'SDE',
    'name': 'George Parasol'
  },
  {
    'title': 'Principal SDE',
    'name': 'Ringo Stone'
  },
  {
    'title': 'Intern',
    'name': 'Eric Lennon'
  }
>>
-- OK!
```

#### Example 3 — Union of Heterogeneous Relations

Here we attempt to union the job titles in HR and Engineering, but there is a mistake. The known static type of *T1* is not compatible with the known static type of *T2*. The column descriptors of each relation are shown below. These relations are not compatible for the standard SQL union.

```sql
SELECT * FROM hr.employees               -- T1
UNION
SELECT title FROM engineering.employees  -- T2
-- ERROR! T1 and T2 are not union-compatible

--       Column Descriptors of T1                          Column Descriptors of T2 

-- | Ordinal | Name  | Type   | Nullable |             | Ordinal | Name  | Type   | Nullable |       
-- |---------|-------|--------|----------|             |---------|-------|--------|----------|
-- | 1       | NAME  | String | NO       |  > UNION <  | 0       | TITLE | String | NO       |
-- | 2       | TITLE | String | NO       |

-- Equivalent, but with OUTER UNION

SELECT * FROM hr.employees
OUTER UNION
SELECT title FROM engineering.employees

<<
  {
    'name': 'Bob Smith',
    'title': 'Consultant'
  },
  {
    'name': 'Susan Smith',
    'title': 'Manager'
  },
  {
    'name': 'Jane Smith',
    'title': 'DEI'
  },
  {
    'title': 'Manager'
  },
  {
    'title': 'SDE'
  },
  {
    'title': 'Principal SDE'
  },
  {
    'title': 'Intern'
  }
>>
-- OK!
```

#### Example 4 — Intersection of Compatible Relations

```sql
SELECT title FROM hr.employees INTERSECT SELECT title FROM engineering.employees

<<
  {
    'title': 'Manager'
  }
>>
-- OK!
```

#### Example 5 — Except of Compatible Relations

```sql
SELECT title FROM engineering.employees EXCEPT (
  SELECT title FROM hr.employees
  UNION
  SELECT title FROM accounting.employees
)

<<
  {
    'title': 'SDE'
  },
  {
    'title': 'Principal SDE'
  }
>>
-- OK!
```

#### Example 6 — Value Coercion

```sql

-- Coercion of single value
SELECT * FROM << 1 >> OUTER UNION 'A'
SELECT * FROM << 1 >> OUTER UNION << 'A' >> -- equivalent
<<
  {
    '_1': 1
  },
  'A'
>>
-- OK!

SELECT * FROM << 1 >>
UNION
SELECT * FROM << 'A' >>
-- ERROR! Not union-compatible

SELECT * FROM << 1 >>
OUTER UNION
SELECT * FROM << 'A' >>
<<
  {
    '_1': 1
  },
  {
    '_1': 'A'
  }
>>
-- OK!
```

#### Example 7 — NULL and MISSING Coercion

```sql
-- result is the same as `TABLE engineering.employees`
SELECT * FROM engineering.employees OUTER EXCEPT << >> 
TABLE engineering.employees OUTER UNION NULL              -- equivalent
TABLE engineering.employees OUTER UNION MISSING           -- equivalent

TABLE engineering.employees UNION << MISSING >>
-- ERROR!

TABLE engineering.employess OUTER UNION << MISSING >>
<<
  {
    'title': 'Manager',
    'name': 'Paul McCharmly'
  },
  {
    'title': 'SDE',
    'name': 'George Parasol'
  },
  {
    'title': 'Principal SDE',
    'name': 'Ringo Stone'
  },
  {
    'title': 'Intern',
    'name': 'Eric Lennon'
  }
  MISSING
>>
-- OK!

-- result is the empty bag
SELECT * FROM engineering.employees OUTER INTERSECT << >>
TABLE engineering.employees OUTER INTERSECT NULL          -- equivalent
TABLE engineering.employees OUTER INTERSECT MISSING       -- equivalent
```


# Drawbacks
[drawbacks]: #drawbacks

We do not recognize drawbacks to implementing these features as these are clearly defined in the specification yet missing from the implementation.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The proposal is based upon the bag operators of SQL++ [1]. These operators define the extension of SQL bag operators for heterogeneous, semi-structured data. Implementing these operators in required for ISO SQL compliance so there are no alternatives.

[1] *The SQL++ Unifying Semi-structured Query Language*: Kian Win Ong, Yannis Papakonstantinou, Romain Vernoux.

# Prior art
[prior-art]: #prior-art

As previously covered, these operators are defined in both the ISO SQL spec as well as the SQL++ paper [1]. Few databases implement these, but their implementations for *UNION ALL* showcase the handling of three problems:

1. Collection Mismatch
2. Coercion Behavior
3. Equality

With respect to the above, PartiQL has chosen to produce heterogeneous bags in permissive mode and homogenous bags or error in conventional typing mode. PartiQL has chosen to coerce NULL and MISSING to the empty bag; but allows for a singleton bag of NULL or MISSING. Finally, equality in bag operators is consistent with equality used in the *GROUP BY* clause.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- Questions in PR: https://github.com/partiql/partiql-docs/pull/7

# Future possibilities
[future-possibilities]: #future-possibilities

We can consider introducing extended bag operators for Ion structs. These extended operators would allow for joining or combining two structs' key-value pairs. That is, these operators would treat each struct as bag of key-value tuples, and the semantics of the set operators apply. 


Let *S* be a struct with key-value pairs *(k_i, v_i)* for *i* in 0 to *n*. We map a key-value pair to a tuple with the following function.

```
(key, value) -> { "k": k_i, "v": v_i }
```

We can then map *S* to a bag by applying the key-value pair mapping function to all pairs.

```
S = {
  k_0: v_0,
  k_1: v_1,
  ...
  k_n: v_n
}

-- as bag

<<
  { "k": k_0, "v": v_0 },
  { "k": k_1, "v": v_1 },
  ...
  { "k": k_n, "v": v_n }
>> 
```

Without this extension, OUTER UNION would behave like so. Each operand is coerced into a singleton bag.

```sql
SELECT * FROM { 'A': 0, 'B': 1 }
OUTER UNION
SELECT * FROM { 'A': 2, 'B': 3, 'C': 4 }

<<
  {
    'A': 0,
    'B': 1
  },
  {
    'A': 2,
    'B': 3,
    'C': 4
  }
>>
-- OK!
```

Extending UNION to Ion structs might behave like so.

```sql
SELECT * FROM { 'A': 0, 'B': 1 }
STRUCT UNION
SELECT * FROM { 'A': 2, 'B': 3, 'C': 4 }

<<
  {
    'A': 0,
    'B': 1,
    'C': 4
  }
>>
-- OK!
```

Alternatively, we could combine the values by keys.

```sql
SELECT * FROM { 'A': 0, 'B': 1 }
STRUCT OUTER UNION
SELECT * FROM { 'A': 2, 'B': 3, 'C': 4 }

<<
  {
    'A': << 0, 2 >>,
    'B': << 1, 3 >>,
    'C': << 4 >>
  }
>>
-- OK!
```

This has been included for future extension because it is related to the set operators, but is not in the SQL specification nor will it be added to the PartiQL specification.

# Appendices

## Appendix A — SQL Set Operators

The goal of this appendix is to clearly define the behavior of column descriptors for SQL-99 conformance set operators.

http://web.cecs.pdx.edu/~len/sql1999.pdf

### Definitions

**Definition**: Query Expression (p265) — For the purposes of this document, *<with_clause>* clause is omitted.

```
<query_expression> ::= [<with_clause] <query_expression_body>

<query_expression_body> ::=
    <non-join_query_expression>
  | <joined_table>

<non-join-query_expression> ::=
    <non-join_query_term>
  | <query_expression_body> UNION  [ALL|DISTINCT] [corr_spec>] <query_term>
  | <query_expression_body> EXCEPT [ALL|DISTINCT] [corr_spec>] <query_term>

<query-term> ::=
    <non-join_query_term>
  | <joined_table>

<non-join_query_term> ::=
    <non-join_query_primary>
  | <query_term> INTERSECT [ALL|DISTINCT] [corr_spec] <query_primary>

<query_primary> ::=
    <simple_table>
  | <joined_table>

<non-join_query_primary> ::=
    <simple_table>
  | ( <non-join_query_expression> )

<simple_table> ::=
    <query_specification>      # SFW
  | <table_value_constructor>  # VALUES ( ... )
  | TABLE <table_name>

<corr_spec> ::= CORRESPONDING [ BY ( <column_name_list> )]
```

**Definition**: Set Operators (p269) — SQL-99 defines the set operators as
- UNION ALL
- UNION DISTINCT
- EXCEPT ALL
- EXCEPT DISTINCT
- INTERSECT ALL
- INTERSECT DISTINCT.

If the set quantifier (ALL, DISTINCT) is not specified, then DISTINCT is implicit. 

**Definition**: Column Descriptor (p41) — A column C is described by a *column descriptor* which includes
- The name of the column
- Whether the name of the column is an implementation-dependent name
- The data type descriptor of the declared type of C
- The default value (if any) of C
- The nullability characteristic of C
- The ordinal position of C within the table that contains it
- An indication of whether the column is a self-referencing column of a base table or not

### Rules for Column Descriptors

The following rules describe what the column descriptors are for various types of *<query_expression_body>*. It is important to understand the column descriptors of the operands because these descriptors are used as the descriptors for the resultant relation of a set operator. 

**Rule 1**

If a *<simple_table>* is a *<query_spec>*, then the column descriptor of the i-th column of the *<simple_table>* is the same as the column descriptor of the i-th column of the *<query_spec>*.

**Rule 2**

If a *<simple_table>* is an *<explicit_table>* (`SELECT * FROM T`), then the column descriptor of the i-th column of the *<simple_table>* is the same as the column descriptor of the i-th column of the table identified by the *<explicit_table>*

**Rule 3**

If a *<simple_table>* is a *<table_value_constructor>*, then the column descriptor of the i-th column of the *<simple_table>* is the same as the column descriptor of the i-th column of the *<table_value_constructor>*, except that the column name of the column descriptor is implementation-dependent and not equivalent to the column name of any column, other than itself, of any table referenced by a *<table_reference>* contained in the outermost SQL-statement.

**Rule 4**
If a *<non-join_query_primary>* is a *<simple_table>*, then its column descriptors are the same as the *<simple_table>*. If a *<query_primary>* is a *<non-join_query_primary>*, the column descriptors are the same as the column descriptors of the *<non-join_query_primary>*. 

### Rules for Set Operators

Let *T1* and *T2* be the first and second operands of a set operator, and let *TR* be the result. 

**Rule 1**
If *CORRESPONDING* is specified, then
1. The column name list cannot contain duplicates
2. At least one column name of T1 must be in T2
3. If no names are specified in the list, then the list is all the column names of T1.
4. Every column name in the list must be a column name of T1 and T2.

**Rule 2**
With a corresponding clause, T1 and T2 must have the same number of columns.

**Rule 3**
If the i-th column of T1 and T2 have the same name *C*, then the i-th column of TR has the name *C*.

**Rule 4**
If the i-th column of T1 and T2 does not have the same name, then the name of i-th column of TR is implementation-dependent and unique among all other column names.

## Appendix B — PartiQL Set Operator Grammar

Aligning PartiQL parser implementation with the PartiQL specification. Right now, the parser will parse the arguments of each set operator as an expression, and the parser does not support the corresponding clause. Here is what we should consider updating our specification grammar to.

```antlr
query
  : sfw_query
  | expr_query
  ;

sfw_query
  : (WITH query AS variable)
    select from
    (WHERE expr_query)?
    (GROUP BY expr_query (AS variable)? (',' expr_query (AS variable)?)*)?
    (HAVING expr_query)?
    ((UNION|INTERSECT|EXCEPT) (ALL|DISTINCT)? corr_spec? query)?
    (ORDER BY sort_item (',' sort_item)*)?
    (LIMIT expr_query)?
    (OFFSET expr_query)?
  ;

expr_query
  : ( sfw_query )
  | path_expr
  | function '(' (expr_query (',' expr_query)*)? ')'
  | '{' (expr_query ':' expr_query (',' expr_query ':' expr_query)*)? '}'
  | '[' (expr_query (',' expr_query)*)? ']'
  | '<<' (expr_query (',' expr_query)*)? '>>'
  | sql_scalar_expr
  | literal
  ;

path_expr
  : variable
  | '(' expr_query ')'
  | path_expr '.' attr_name
  | path_expr '.' '*'
  | path_expr '[' expr_query ']'
  | path_expr '[' '*' ']'
  ;

corr_spec
  : CORRESPONDING ( BY '(' column_name (',' column_name)* ')' )?
  ;
```

## APPENDIX C — PartiQL Values

```antlr
value
  : absent_value
  | scalar_value
  | tuple_value
  | collection_value
  ;

absent_value
  : NULL
  | MISSING
  ;

scalar_value
  : '`' ion_literal '`'
  | literal
  ;

tuple_value
  : '{' (string_value ':' value (',' string_value ':' value )*)? '}'
  ;

collection_value
  : array_value
  | bag_value
  ;

array_value
  : '[' (value (',' value)*)? ']'
  ;

bag_value
  : '<<' (value (',' value')*)? '>>'
  ;
```

## Appendix D — Column Type Comparability

SQL defines two types as comparable if their values are comparable. This is a brief overview of comparability as defined by the SQL-99 specification. This is section is included for aiding the reader, not as a definitive reference.

> **comparable:** The characteristic of values that permits one value to be compared with another value. Also said of data types: Two data types are comparable if values of those data types are comparable. If one of the two data types is a user-defined type, then both shall be in the same subtype family. See Subclause 4.12, ‘‘Type conversions and mixing of data types’’.

### Subclause 4.12

- Values of the data types NUMERIC, DECIMAL, INTEGER, SMALLINT, FLOAT, REAL, and DOUBLE PRECISION are numbers and are all mutually comparable.
- CHARACTER, CHARACTER VARYING, and CHARACTER LARGE OBJECT are mutually comparable if and only if they are taken from the same character repertoire (ASCII, UNICODE).
- When binary string values are compared, they must have exactly the same length (in octets) to be considered equal. Binary string values can only be compared for equality.
- Values corresponding to the data types BIT and BIT VARYING are always mutually comparable and are mutually assignable
- Values corresponding to the data type boolean are always mutually comparable and are mutually assignable.
- Values corresponding to row types are mutually comparable if and only if both have the same degree and every field in one row type is mutually comparable to the field in the same ordinal position of the other row type.
- Two collections are comparable if and only if they are of the same collection type and their element types are comparable
- Items of type datetime are mutually comparable only if they have the same <primary datetime field>s.
- Year-month intervals are mutually comparable only with other year-month intervals
- Day-time intervals are mutually comparable only with other day-time intervals.
- Two values V1 and V2 of whose declared types are user-defined types T1 and T2 are comparable if and only if T1 and T2 are in the same subtype family and each have some comparison type CT1 and CT2, respectively
