### Summary

The RFC adds the specification for PartiQL Identifier. 

Identifier in this RFC should be viewed as a lexical element,
and later RFCs might extend the semantics based on the specific context.  

This RFC first purposes the semantics of identifiers in PartiQL (normalization and comparison),
then use the purposed semantics to clarify various operations stated in the PartiQL specification.

Once approved, this RFC should be considered as complementary to the existing PartiQL Specification.

### Motivation
The PartiQL Specification does not explicitly state the semantics for identifier, causing ambiguities when interpreting the operational semantics of PartiQL. 

The goal of this RFC is to add the specification of identifier in PartiQL, in a way such that: 
  - Follow-up specification works should be allowed to extend the semantics of identifier without being breaking or overly-verbose. 
  - The PartiQL identifier should be able to accommodate the semantics of SQL and schemaless data operation. 

# Guide-level explanation

The keywords “MUST”, “MUST NOT”, “REQUIRED”, “SHALL”, “SHALL NOT”, “SHOULD”, “SHOULD NOT”, “RECOMMENDED”, “MAY”, and “OPTIONAL” in this format specification are to be interpreted as described in [Key Words RFC2119](https://datatracker.ietf.org/doc/html/rfc2119).

## Terminology

* **Identifier**: A token that forms a name. An identifier in this RFC is treated as a lexical element, and might be extended based on the context.
* **Regular Identifier**: Identifiers that comply with the rules for the format of identifiers.
* **Delimited Identifier**: Identifiers that are enclosed in double quotation marks.
* **Qualified Identifier**: Dot-separated sequence of identifiers.
* **Name**: Context-dependent usage of identifier. 
* **Attribute Name**: An identifier that designates a struct field. 
* **Function Name**: An identifier that designates a function. 
* **Bind Name**: An identifier that designates a value/type in binding tuple. 
* **Tuple Value**: A tuple value is a collection of fields.
* **Struct Value**: Struct is synonymous with tuple in the value context. 
* **Field**: A key value Pair, where the key is a String, and value is any valid PartiQL Value. 

# Guide-level explanation

## 1. Out of Scope
- The identifier in the RFC refers to names of objects within the binding/type environment. The DBMS instance may store instance-level objects(i.e., Roles that are shared across catalogs), the identifiers of such objects are out of the scope for this RFC. 
- This RFC focus on the base semantics of identifier. When used in different contexts, an identifier may carry extended semantics rules. This RFC does not list all possible use cases of identifier.  

## 2. Character Set
- This RFC assumes a Character Set called PARTIQL_IDENTIFIER which includes all characters that the PartiQL-implementation supports for use in regular identifier and delimited identifier. 
- All implementations of PartiQL MUST support the characters listed in [Minimum character support requirement for all implementation](#minimum-character-support-requirement-for-all-implementation)
- An implementation MAY choose to extend the supported character, in which case the collation for the character set is implementation defined.
> Note: Collation defined for character-set was used during character comparison.
> 
> As indicated in a later section, PartiQL support comparison between regular identifier and delimited identifier.
> Hence regular identifier and delimited identifier must use the same character set.
> 
> For example: 
> 
> Å   -- U+212B ANGSTROM SIGN
> 
> Å   -- U+00C5 LATIN CAPITAL LETTER A WITH RING ABOVE
> 
> A ◌̊ -- U+0041 LATIN CAPITAL LETTER A, U+030A COMBINING RING ABOVE Text render &#65;&#778; (`&#65;&#778;`)
> 
> All the above might be considered equivalent based on the collection rules

## 3. Grammar and Lexer Rules
```

<qualified identifier> : 
    [ <at symbol> ] <identifier> [ ( <dot> <identifier> ) ...]
<identifier> ::= 
    ( <regular identifier> | <delimited identifier> )
   
<regular identifier> ::=
    <identifier body>

<identifier body> ::=
    <identifier start> [ <identifier part>... ]

<identifier part> ::=
    <identifier start>
  | <identifier extend>

<identifier start> ::=
    !! See Syntax Rules Below.  
    
<identifier extend> ::=
    !! See Syntax Rules Below.  
    
<delimited identifier> ::=
    <double quote> <delimited identifier body> <double quote>
    
<delimited identifier body> ::=
    <delimited identifier part>...
<delimited identifier part> ::=
    <nondoublequote character>
   | <doublequote symbol>        

<doublequote symbol> ::=
    ""!! two consecutive double quote characters
```

### Syntax Rule: 

1. Identifier Start: An identifier may start with any character in the Unicode General Category classes "Lu", "Ll", "Lt", "Lm", "Lo", "Nl", U+005F (Low Line)
    - Lu: Uppercase letter; Ll: Lowercase letter; Lt: TitleCase letter; Lm: Modifier Letter; Lo: Other Letter; Nl: Letter Number;

2. Identifier Extend: An identifier extend is any character in the Unicode General Category classes "Mn", "Mc", "Nd", "Pc", or "Cf" or U+00B7 (Middle Dot)
    - Mn: Nonspacing mark; Mc: spacing mark; Nd: Decimal Number; Pc: Connector Punctuation; Cf: Format

3. An (qualified) identifier can be optionally prefixed with an <at symbol> U+0040 (at)
   - If an identifier is prefixed with @, 
     - such identifier refers to the environment variable named _identifier_. 
     - If there is no such environment variable, the identifier refers to the database name _identifier_.

4. Nondoublequote Character: Any character that is supported by a PartiQL implementation other than a <double quote>.

#### Example Identifier
```
-- Legal Identifier; Required for all implementation
Foo            -- Regular identifier.
"Foo"          -- Delimited identifier.
Foo.Bar        -- Qualified Identifier, all parts are regular identifier.
"Foo"."Bar"    -- Qualified Identifier, all parts are delimited identifer.
Foo."Bar"      -- Qualified Identifier, mixed parts of regular and delimited identifier.
@Foo           -- Identifier starts with an @ symbol.
@Foo.Bar       -- Qualified Identifier starts with an @ symbol.
@"Foo"         -- Delimited Identifier starts with an @ symbol.
_1             -- Regular Identifier starts with an _ symbol.
标识符          -- Legal Identifier if character set is supported by given implementation.

-- Illegal Identifier: 
Foo.@Bar       -- the @symbol should only appear at the beginning of a (qualified) identifer.
₤1             -- Illegal identfier start, the ₤ symbol is catagorized as Sc.
F⃠⃠           -- Illegal identifier extend, the ⃠ symbol is catagorized as Me.

-- Delimited Identifier can escape the syntax rule
"₤1"           -- Legal Delimited Identifier if character set is supported by given implementation.
"F⃠⃠"         -- Legal Delimited Identifier if character set is supported by given implementation.
```

## 4. Identifier Semantic
This section defines the operational semantics of identifier.
[Section 5](#5-extensionclarification-on-partiql-specification) then uses the operational semantics defined to extend/clarify the existing PartiQL Spec.

In this RFC, identifiers are treated as Lexical elements.
That is: In subsequent RFCs, additional grammar rules might be defined to use the Lexical elements:

For example, a subsequent RFC on PartiQL Function may choose to define:

```
<function name> : <identifier>
```

In such cases:

- All function names MUST comply with the rules defined for identifier.
- There might be additional requirement when identifier is used as function name, in which case, the `<function name>` should be used to clarify the context.

### 4.1 Identifier Modeling and representation
For this RFC, an identifier can be described using:

1. IDENTIFIER_BODY: a case-sensitive textual representation of the identifier.

2. IS_REGULAR: a boolean flag indicating if the identifier is a regular identifier.

Visually, this RFC represents regular identifier by text only (i.e., `identifier`),
delimited identifier by text with double quotes (i.e., `"identifier"`), 
string by text with single quotes (i.e., `'string'`)

### 4.2 Identifier Normalization
In the interests of SQL specification compatibility,
common DBMS implementation compatibility(Postgres, etc.), and DocumentDB compatibility(N1QL, etc.), 
an implementation may choose to turn on case normalization mode,
and in such cases, an implementation defined normalization algorithm (IDN) is used to normalize the identifier body.

The IDN function takes ONLY the regular identifier
and returns a textual representation of the regular identifier. 

> Let RI1 and RI2 be two regular identifiers:
>
> IDN(RI1) = IDN(RI2) must return true if: 
> 
>       RI1.IDENTIFIER_BODY.isEqualTo(RI2.IDENTIFIER_BODY, ignoreCase = false) == TRUE
>
>
> IDN(RI1) = IDN(RI2) must return false if: 
> 
>       RI1.IDENTIFIER_BODY.isEqualTo(ID2.IDENTIFIER_BODY, ignoreCase = True) == FALSE

See [Appendix](#example-idn) for reference normalization algorithms.

Identifier normalization is for purposes such as determination of identifier equivalence, representation in the binding/type environment, etc.

This RFC first defines the normalization rules on identifier, 
then generalizes the rule for qualified identifier in [section 4.3.2](#432-eqi-function-on-qualified-identifier). 

PartiQL internally MUST normalize identifiers before preservation or resolution. 

Formally, this RFC defines two functions: `cni` (case normal identifier), and `cnf` (case normal form). 

The `cnf` function takes in an identifier body and returns a normalized identifier body.
```
                          { 
                            ID.IDENTIFIER_BODY if ID is a delimited Identifier      
cni(ID.IDENTIFIER_BODY) =   ID.IDENTIFIER_BODY if ID is a regular identifier and normalization mode is set to OFF
                            IDN(ID.IDNETIFIER_BODY) if ID is a regular identifier and normalization mode is set to ON
                          }                            
```


The `cni` function takes in an identifier and returns a normalized identifier.
```
          { 
            ID if ID is a delimited Identifier      
cni(ID) =   ID if ID is a regular identifier and normalization mode is set to OFF
            delimited_identifier(cnf(ID)) if ID is a regular identifier and normalization mode is set to ON
          }                            
```

#### 4.2.1 Example of CNF function

|       | ON(UPPERCASE) | ON(LOWERCASE) | ON(EXACTCASE) | OFF   |
|-------|---------------|---------------|---------------|-------|
| BAR   | 'BAR'         | 'bar'         | 'BAR'         | 'BAR' |
| bAr   | 'BAR'         | 'bar'         | 'bAr'         | 'bAr' |
| bar   | 'BAR'         | 'bar'         | 'bar'         | 'bar' |
| "bAr" | 'bAr'         | 'bAr'         | 'bAr'         | 'bAr' |


#### 4.2.2 Example of CNI function

|       | ON(UPPERCASE) | ON(LOWERCASE) | ON(EXACTCASE) | OFF   |
|-------|---------------|---------------|---------------|-------|
| BAR   | "BAR"         | "bar"         | "BAR"         | BAR   |
| bAr   | "BAR"         | "bar"         | "bAr"         | bAr   |
| bar   | "BAR"         | "bar"         | "bar"         | bar   |
| "bAr" | "bAr"         | "bAr"         | "bAr"         | "bAr" |

### 4.2.3 Qualified Identifier Normalization
Let ID to be an qualified identifier i<sub>1</sub>. ... . i<sub>n</sub>.

The `cni` function is defined over qualified identifier as follows: 

Let j<sub>k</sub> to be the corresponding k<sup>th</sup> path in the normalized identifier,

j<sub>k</sub> = cni(i<sub>k</sub>)

Note: the function `cnf` can not be directly invoke on qualified identifier, as cnf returns a textual representation of identifier body.

There is no identifier body defined directly on qualified identifier.

### 4.3 Identifier Equivalence - Identifier to Identifier Comparison
This RFC first defines the comparison rules between identifiers,
then generalizes the rule for qualified identifier in [section 5.5.1](#551-qualified-identifier-equivalence).

PartiQL defines a function called `eqi` (equivalent identifier).

Let the signature of `eqi` be defined as:
```
eqi(ID1: IDENTIFIER, ID2: IDENTIFIER) : BOOLEAN
```

The function `eqi` returns true if the two identifiers are considered equivalent by PartiQL.

The function `eqi` is defined as following:

Let ID1 and ID2 be the two arguments for `eqi` function,

- If ID1 and ID2 are both delimited Identifiers, then ID1 and ID2 are considered equivalent if and only if:

       ID1.IDENTIFIER_BODY.isEqualTo(ID2.IDENTIFIER_BODY, ignoreCase = false) returns TRUE.

> Note the operation
considered ID1.IDENTIFIER_BODY and ID2.IDENTIFIER_BODY as string literal
that uses collation of character set PARTIQL_IDENTIFIER for comparison operation.


- If ID1 or ID2 is a regular Identifier,
  then ID1 and ID2 are considered equivalent if and only if:

       ID1.IDENTIFIER_BODY.isEqualTo(ID2.IDENTIFIER_BODY, ignoreCase = True) returns TRUE.

This is:
two regular identifiers are equivalent
if their IDENTIFIER_BODY is considered equal
after replacing every lowercase letter with the equivalent of upper case letter.  

#### 4.3.1 Example eqi function

|              | ON(UPPERCASE)               | ON(LOWERCASE)               | ON(EXACTCASE)                | OFF                       |
|--------------|-----------------------------|-----------------------------|------------------------------|---------------------------|
| (BAR, bAr)   | eqi("BAR", "BAR")<br/>TRUE  | eqi("bar", "bar")<br/>TRUE  | eqi("bAr", "BAR")<br/> FALSE | eqi(BAR, bAr)<br/>TRUE    |
| (bAr, bAr)   | eqi("BAR", "BAR")<br/>TRUE  | eqi("bar", "bar")<br/>TRUE  | eqi("bAr", "bAr")<br/>TRUE   | eqi(bAr, bAr)<br/>TRUE    |
| (bar, bAr)   | eqi("BAR", "BAR")<br/>TRUE  | eqi("bar", "bar")<br/>TRUE  | eqi("bAr", "bar")<br/>FALSE  | eqi(bar, bAr)<br/>TRUE    | 
| (bAr, "BAR") | eqi("BAR", "BAR")<br/>FALSE | eqi("bar", "BAR")<br/>FALSE | eqi("bAr", "BAR")<br/>FALSE  | eqi(bAr, "BAR")<br/>TRUE  | 
| (bAr, "bAr") | eqi("BAR", "bAr")<br/>FALSE | eqi("bar", "bAr")<br/>FALSE | eqi("bAr", "bAr")<br/>TRUE   | eqi(bAr, "bAr")<br/>TRUE  | 
| (bAr, "bar") | eqi("BAR", "bar")<br/>FALSE | eqi("bar", "bar")<br/>TRUE  | eqi("bAr", "bar")<br/>FALSE  | eqi(bAr, "bar")<br/>TRUE  | 

#### 4.3.2 eqi function on qualified identifier
Let ID1 to be an qualified identifier i<sub>1</sub>. ... . i<sub>n</sub>.
Let ID2 to be an qualified identifier j<sub>1</sub>. ... . j<sub>m</sub>.

ID1 and ID2 is considered equal if and only if:
1. n == m
2. eqi(i<sub>k</sub>, j<sub>k</sub>) return true for every k where k in [1, n].


### 4.4 Matching — Identifier to String comparison
PartiQL defines a function called `match`. 

```
                    {
                      ID.IDENTIFIER_BODY.isEqualTo(STRING, ignoreCase = false) if ID is a delimited identifier
match(ID, STRING) =   
                      ID.IDENTIFIER_BODY.isEqualTo(STRING, ignoreCase = true)  if ID is a regular identifier
                    } 
```
This is, 
let `RI` be any Regular identifier and `S` be any String: 

Let `S1` to be `RI.IDENTIFIER_BODY` with every lowercase letter with the equivalent of upper case letter.
Let `S2` to be `S` with every lowercase letter with the equivalent of upper case letter.

`RI` matches `S` if `S1` and `S2` are considered equal according to string comparison. 


## 5. Extension/clarification on PartiQL Specification


### 5.1 Preservation to Binding tuple
Let ID be an identifier that is to be preserved in the binding tuple;
in such context, the identifier is called `<bind name>`. 

let `v` be a PartiQL Value associated with ID; let `t` be the PartiQL Type associated with ID.

The binding tuple to be added/concatenated is `<cni(ID), v>`, which binds `<bind name>` `cni(ID)` with PartiQL Value `v`.

The binding tuple to be added/concatenated the type environment is `<cni(ID), t>`, which binds identifier `<bind name>` `cni(ID)` with PartiQL type `t`.

#### 5.1.1 Uniqueness of Binding Tuple
> In either case, an environment is a binding tuple < x1: v1, . . . , xn: vn > ,
> where each xi is a bind name that is unique and binds to the PartiQL value vi.

The uniqueness of binding tuple is then defined as:

`eqi(xi, xj)` must return false for any two elements `xi`, `xj`, i != j, in the binding tuple.

Consider the FROM Clause:
```
FROM [{'a' : 1}] AS x AT y
```

The binding tuple comes out of the FROM Clause is:
```
B_out_from = <<  < cni(x): {'a' : 1}, cni(y) : 1>  >>
```

Next, modify as alias and at alias to be: 
```
FROM [{'a' : 1}] AS x AT x
```

The binding tuple comes out of the FROM Clause is:
```
B_out_from = <<  < cni(x): {'a' : 1}, cni(x) : 1>  >>
```
Note that `cni(x)` must be equal to `cni(x)`,
hence the binding tuple produced violates the binding tuple uniqueness constraint,
leading to query failure.

Compare the above with: 
```
FROM [{'a' : 1}] AS x AT X

B_out_from = <<  <cni(x): {'a' : 1}, cni(X) : 1>  >>
```
Note that what `eqi(cni(x), cni(X))`
returns is dependent on normalization mode and implementation defined normalization algorithm. 

For example, If the IDN is to preserve the original case:

```
B_out_from = <<  <cni(x): {'a' : 1}, cni(X) : 1>  >>
           = <<  <"x": {'a' : 1}, "X" : 1>  >>
```
and the query should succeed.

If the IDN is folding to upper-case: 
```
B_out_from = <<  <cni(x): {'a' : 1}, cni(X) : 1>  >>
           = <<  <"X": {'a' : 1}, "X" : 1>  >>
```
and query complication should fail. 

Similar: 
```
FROM [{'a' : 1}] AS x AT "x"
```

The binding tuple comes out of the FROM Clause is:
```
B_out_from = <<  < cni(x): {'a' : 1}, cni("x") : 1>  >>
           = <<  < cni(x): {'a' : 1}, "x" : 1 >> 
```
The result of `eqi(cni(x), "x")` returns is mode-dependent.


#### 5.1.2 Concatenation of Binding Tuple
Most operations in PartiQL Specification (Join, subquery, etc.)
produce a new binding environment by concatenating the variable environment binding tuples( `b||b'`).

The same `eqi(x1,xj)` function is used
to assert equivalent bind name see [Section 3.4](https://partiql.org/assets/PartiQL-Specification.pdf#subsection.3.4) holds.

Consider the FROM Clause contains a simple join
```
FROM <<{'a' : 1, 'b': 2} >> as t1, <<{'a' : 1, 'c' : 3}>> as t2
```

The binding tuple come out of FROM is trivial:

```
B_out_from = <<  < cni(t1): {'a' : 1, 'b': 2} > || < cni(t2): {'a' : 1, 'c' : 3} >  >>
           = <<  < cni(t1) : {'a' : 1, 'b': 2}, cni(t2): {'a' : 1, 'c' : 3} >  >>
```

Modifying the above FROM Clause slightly:

```
FROM <<{'a' : 1, 'b': 2} >> as t, <<{'a' : 1, 'c' : 3}>> as t
```

based on the PartiQL Specification:
>  the “l CROSS JOIN r” outputs all binding tuples b = b^l || b^r

```
B_out_from = <<  < cni(t): {'a' : 1, 'b' : 2}> || < cni(t): {'a' : 1, 'c' : 3}>  >>
           = <<  < cni(t): {'a' : 1, 'c' : 3}>  >>
```

Now: Consider
```
FROM <<{'a' : 1, 'b': 2} >> as t, <<{'a' : 1, 'c' : 3}>> as "t"

B_out_from = <<  <cni(t): {'a' : 1, 'b' : 2}> || < cni("t"): {'a' : 1, 'c' : 3}>  >>
           = <<  <cni(t): {'a' : 1, 'b' : 2}> > || < "t" : {'a' : 1, 'c' : 3}>  >>

** Note that the return is dependent on normalization mode
```

### 5.2 Preservation to Tuple Value
> The SQL syntax:
>
> SELECT e1 AS a1, . . ., en AS an
>
> is syntactic sugar for:
>
> SELECT VALUE {'a_1' :e_1, . . ., 'a_n': e_n}
>
> whereas if the attribute name a_i is written as an identifier (e.g., a or "a").
> it is replaced by a single-quoted form a_i' (e.g., 'a').

We expand the specification by 
suggesting the `<attribute name>` is used in place of AS alias, 
and the struct field name is defined `cnf(a1)`. 

The PartiQL generic query then becomes:
```
SELECT VALUE {cnf(a1): cni(e1)}
```
In the case where SQL coercion is not concerned, the equivalence relationship between

    SELECT e1 AS a1, . . ., en AS an

and

    SELECT VALUE {cnf(a1): cni(e1)}

holds.


## 5.3 Look-up
The spec describes two types of look-up operation,
look up from binding tuple and look up from Tuple value (path navigation). 

The rules of which look-up operation should be invoked is defined in the PartiQL Specification
and will not be restated in the following section. 

Instead, this section assumes one type of look-up operation is initialized, and defines the semantics of equivalence.

### 5.3.1 Binding tuple look up
Reducing the problem to be comparing an `<identifier>` ID with `<bind name>` BN: 

The comparison operation is defined as: 
- If `eqi(cni(ID), BN)` return false, then the comparison operation must failed. 
- Otherwise, if BN is a regular identifier, then the comparison operation is effectively
  cni(ID).IDENTIFIER_BODY = BN.IDENTIFIER_BODY, where the equal function is a case sensitive string comparison. 

For example: Consider the query: 

```
SELECT "T" FROM <<{'a' : 1, 'b': 2} >> as t
```

in normalization -- OFF mode: 

```
B_out_from = <<  < t: {'a' : 1, 'b': 2} >
```

Even though eqi("T", t) returns true, the second rule prevents the identifier "T" to be resolved. 

### 5.3.2 Tuple Field look up
Reducing the problem to be comparing an `<identifier>` ID with `<field key>` A (note that A is a string)

The comparison operation should succeed if: 
`match(cni(ID), A)` return true. 

Consider a tuple path navigation `t.a`, where `t` is a tuple, the expression is equivalent to `t.cni(a)`.  

For example: 
```
{'a': 1, 'b':2}.a

<=> 

{'a': 1, 'b':2}.cni(a)
```

> Note: The result of the operation is mode-dependent. 

## 5.4 Function resolution
This section assumes function name should behave case-insensitively regardless of mode. 

This is: Suppose we have a build-in function called `UPPER`. 
`Upper('a')` should work regardless of the normalization mode and implementation defined normalization algorithm.  

> Note the spec is not explicit on how function resolution works. The definition of such is out of the scope for this RFC.
> 
> This example purposes one possible model to explain the behavior, 
but is not hinting on the model being used outside of this example. 

Suppose the build-in function UPPER is stored in a "prelude environment": with name being regular identifier `UPPER`: 

The given resolution of function name `Upper` depends on the result of `eqi(cni(Upper), UPPER)`,
which always returns true.

Formally: 

Reducing the problem to be comparing an `<identifier>` ID with `<function name>` FN. 

The function resolution should succeed if: eqi(cni(ID), FN) returns true.

# Drawbacks
The purposed semantics is backward incompatible with the PartiQL-Lang-Kotlin implementation
(which treats `AS` alias differently than identifier).

Consider:
```
SELECT * FROM [{'a' : 1}] AS x AT X
```

In the current PLK implementation, the above query will succeed.


# Rationale and alternatives

* Why is this design/proposal the best in the space of possible designs?

This RFC purposed a generalization framework for identifier,
while leaving the door open for extension based on the context the identifier is used in. 

One may think identifier as a base class, while the context-dependent usage of identifier
(function name, attribute name, etc) as an extension.

Such framework allows for unification of identifier semantics and is easy to extend in the future,
without the need to define semantics based on each usage context. 

* Which other designs/proposals have been considered, and what is the rationale for not choosing them?

One alternative is to make the `eqi` function independent of `cni` function
(and hence the implementation-defined normalization algorithm). 

Doing so, a PartiQL query, with all identifiers being quoted or unquoted,
will be portable<sup>1</sup> between different normalization mode.

> portable: A PartiQL query that can be compiled in one mode should also be compiled in another mode. 

However, considering the compatibility with databases with case-sensitive identifier, this approach is discarded. 

* What is the impact of not doing this? No service level or production impact as this is an addition to the PartiQL specification.

# Prior art

The SQL specification is quite explicit on the definition and semantics on identifier.
Unfortunately,
the implementations of SQL (Postgres, Redshift, MySQL, etc.) have different semantics and rules on this subject.
For example, Postgres internally lowercase all usage of regular identifier. 

Moreover,
other SQL++ implementations (Asterix, N1QL, etc.) have made the decision that their identifier is case-sensitive.

# Unresolved questions

# Future possibilities

In the subsequent RFCs,
where additional usages of identifiers are clarified/specified, the semantics might be extended based on the context. 


## Appendix

### Minimum character support requirement for all implementation
```
<simple Latin upper case letter> ::=
A | B | C | D | E | F | G | H | I | J | K | L | M | N | O
| P | Q | R | S | T | U | V | W | X | Y | Z

<simple Latin lower case letter> ::=
a | b | c | d | e | f | g | h | i | j | k | l | m | n | o
| p | q | r | s | t | u | v | w | x | y | z

<digit> ::=
0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9

<special character> ::= 
     <space>
   | <double quote>
   | <percent>
   | <ampersand>
   | <quote>
   | <left paren>
   | <right paren>
   | <asterisk>
   | <plus sign>
   | <comma>
   | <minus sign>
   | <period>
   | <solidus>
   | <colon>
   | <semicolon>
   | <less than operator>
   | <equals operator>
   | <greater than operator>
   | <question mark>
   | <left bracket>
   | <right bracket>
   | <circumflex>
   | <underscore>
   | <vertical bar>
   | <left brace>
   | <right brace>
```


### Example IDN

1. UPPERCASE MODE:
Let **RI** be any regular identifier. Let n be the number of character in `RI.IDENTIFIER_BODY`.

For i ranging from 1 to n, the i-th character M<sub>i</sub> of IB is transliterated into the corresponding character or characters of case normal form as follows:

    - If M<sub>i</sub> is a lower case character or a title case character for which an equivalent upper case sequence U is defined by Unicode, then let j be the number of character in U; the next j characters of CNF are U.
    
    - Otherwise, the next character of CNF is M<sub>i</sub>

2. LOWERCASE

Let **RI** be any regular identifier. Let n be the number of character in `RI.IDENTIFIER_BODY`.

For i ranging from 1 to n, the i-th character M<sub>i</sub> of IB is transliterated into the corresponding character or characters of case normal form as follows:

    - If M<sub>i</sub> is a upper case character or a title case character for which an equivalent upper case sequence U is defined by Unicode, then let j be the number of character in U; the next j characters of CNF are U.
    - Otherwise, the next character of CNF is M<sub>i</sub>

3. EXACTCASE

Let **RI** be any regular identifier. Let n be the number of character in `RI.IDENTIFIER_BODY`.

For i ranging from 1 to n, the i-th character M<sub>i</sub> of IB is transliterated into the corresponding character or characters of case normal form as follows:

    - the next character of CNF is M<sub>i</sub>