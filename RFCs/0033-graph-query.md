- RFC-0033: Graph Query
- Start Date: 2022-10-25
- RFC PR: https://github.com/partiql/partiql-docs/pull/33

# Summary
[summary]: #summary

This RFC proposes how graph instances conforming to the data model defined in 
[DRAFT] RFC-0025 "Graph Data Model" will be queried in PartiQL. 
The core of the proposal is to largely follow GPML, the Graph Pattern Matching Language, 
which is under standardization as part of ISO 9075-16: SQL/PGQ. 
Consequently, this RFC focuses on how GPML is adapted for and incorporated into PartiQL, 
while, for shared details, it defers to publicly available GPML information 
(currently, [^gpml-paper]). 

Familiarity with [^gpml-paper] is required for understanding the _full impact_ 
of this RFC on PartiQL, 
however this RFC is meant to be self-contained (in conjunction with RFC-0025).


# Motivation
[motivation]: #motivation

See RFC-0025 "Graph Data Model".


# Guide-Level Explanation
[guide-level-explanation]: #guide-level-explanation

## Overview

The construct for querying a graph that is proposed in this RFC is the *graph matching expression* 
of the form 

>    `(` _graph_ `MATCH` _pattern_ `)`

where 

- _graph_ is an expression that evaluates to a graph, in the data model proposed in RFC-0025;
- _pattern_ is a graph matching pattern, mostly following the GPML syntax and semantics; 
- `MATCH` is a keyword, while the surrounding parentheses `(` ... `)` are required, in most cases.

A graph pattern _pattern_ describes a graph fragment of interest to be found in the input 
graph _graph_.  Variables within the pattern are used to designate elements of the fragment 
that are needed in the result.  
The result of this expression is a bag of structs (a.k.a. a table), 
with one struct (a.k.a. row) per each fragment found in _graph_ that matched _pattern_, 
where fields 
in the struct correspond to the variables in _pattern_.  

With the result of a graph matching expression being a bag of structs, this expression 
can occur anywhere in a PartiQL query, as any other expression.  In particular, 
it can occur as a data source in a `FROM` clause of a query, which is the primary 
intended usage.

<!-- TODO? An example query. (Within FROM or independent of it?) -->

Computation of a pattern match involves navigation around the graph, but
graph navigation capabilities are supported only within the scope of this computation.
The result table that gets out does not carry any implicit information about the graph's structure; 
it only carries the "payload" information from the matched graph elements (see RFC-0025).
In a sense,  what is inside a MATCH expression's pattern is a separate query language, 
distinct from PartiQL, and this language also has its own distinct evaluation mechanisms.


## Relation to GPML and SQL/PGQ 

GPML is the name given to the graph pattern matching language described in a 
publicly available paper [^gpml-paper] that previews such a language developed for 
SQL/PGQ and GQL, the upcoming standards from ISO and IEC. 
SQL/PGQ is ISO 9075-16 -- a new section in the SQL standard ISO 9075 that adds to SQL ability to 
query graphs and, presumably, those will be graphs defined as "views" over relational tables, 
according to conventions to be described in ISO 9075-16 as well. 
GQL will be a language for "native" graph databases, with full CRUD capabilities for graphs.
Graph queries by means of graph patterns will be a capability 
shared between SQL/PGQ and GQL, and that is what is referred to as GPML.  

This RFC intends to describe for PartiQL parts of what SQL/PGQ is meant 
to describe for SQL: a graph pattern language and how it is integrated into 
the host language (PartiQL or SQL).  
In general, PartiQL strives for SQL compatibility.
While [^gpml-paper] provides good amount of detail about the pattern language, 
the details of its integration into SQL are less clear.
Consequently, the proposal in this RFC may need adjustments (more likely in syntax,
but possibly in semantics as well) after SQL/PGQ becomes available.

Graph pattern language for PartiQL described in this RFC largely follows GPML 
as descibed in [^gpml-paper], with two differences:

- Since the graph data model in RFC-0025 is more general than in GPML
  (it allows any PartiQL value as a payload at a graph element, not just something 
  that mimics GPML's key/value properties), this has to be addressed for the relevant 
  aspects of the pattern language.

- In GPML, at least as described in [^gpml-paper], graph queries have form
  `MATCH` _pattern_, where the graph being queried is implicit, 
  while here graph match expression has form
  `(` _graph_ `MATCH` _pattern_ `)`, where the graph is explicit. 


## Grammatical Details

### Idealized Grammar

Ideally, graph pattern matching would be added to PartiQL grammar as a new production 
to the non-terminal `<expr_query>` (which defines constructs like scalar, struct, 
and collection literals; function calls; parenthesized SELECT-FROM-WHERE, etc.):

```EBNF
<expr_query> ::= 
   // ...
   |  <expr_query> 'MATCH' <graph_pattern>
   
<graph_pattern> ::= 
   <selector>? <path_pattern> [',' <path_pattern>]*
```
where `<graph_pattern>`, as well as the non-terminals on which it depends, are defined as in GPML. 

After this, the match expression would become available, as an `<expr_query>`, 
to be used as a source in the `FROM` clause of SELECT-FROM-WHERE queries, which is defined as 

````EBNF
<from_clause> ::=  
      'FROM' <from_item> [ ',' <from_item> ]*
  
<from_item> ::= 
      <expr_query> [ 'AS' <id> [ AT <id> ]? ]?
    // ... joins, etc
````
and, as a grammar, that would not need any change. 

Unfortunately, such straightforward modifications would introduce ambiguities, making it impossible
to parse for this grammar with one-token lookahead: the new rule for `<expr_query>` 
is left-recursive, and there could be uses of `,` for which it is impossible to decide whether 
they separate instances of `<path_pattern` or instances of `<from_item>`.


### More Practical Grammar

To address these parsing difficulties, there are a couple tweaks to the above grammar. 

First, a match expression, in general, must be parenthesised. That is, `<expr_query>` 
is extended as 

```EBNF
<expr_query> ::=
    // ...
    | '(' <expr_query> 'MATCH' <graph_pattern> ')'
   
<graph_pattern> ::= 
    <selector>? <path_pattern> [ ',' <path_pattern> ]*
```

Second, the parenthesation restriction is relaxed in a `FROM` clause: when 
a match expression is used as a source of data _and_ its pattern consists of a 
single path pattern, then the parentheses are not necessary: 

```EBNF
<from_clause> ::=  
      'FROM' <from_item> [ ',' <from_item> ]*
  
<from_item> ::= 
      <expr_query> [ 'AS' <id> [ AT <id> ]? ]?
    | <expr_query> 'MATCH' <selector>? <path_pattern> [ 'AS' <id> [ AT <id> ]? ]?
    // ... joins, etc
````

### Graph Patterns

The grammar in this section is a reverse-engineered reconstruction based on the examples 
in [^gpml-paper]. It is followed in the current experimental implementation of 
graph query parsing in partiql-lang-kotlin, starting in 0.8.0.  
This grammar is subject to change, taking into account any details that might appear 
in SQL/PGQ but were not covered in [^gpml-paper].
It also does not cover several features mentioned in the paper in passing only
(label patterns, predicates for node/edge relationships).

To clarify how graph pattern and the "host" PartiQL grammars are combined, 
PartiQL non-terminals are indicated with double angle brackets, 
specifically `<<integer>>`, `<<identifier>>`, and `<<where_clause>>`.

```EBNF
<graph_pattern> ::= 
    <selector>? <path_pattern> [ ',' <path_pattern> ]*

<path_pattern> ::=
    <restrictor>?  [ <path_variable> '=' ]? <path_part>+ 

<path_variable> ::=  <<identifier>>

<path_part> ::=
      <node_pattern>
    | <edge_pattern>
    | <group_pattern>

<selector> ::=
      [ 'ANY' | 'ALL' ] 'SHORTEST'
    | 'ANY' <<integer>>?
    | 'SHORTEST' <<integer>> 'GROUP'?

<restrictor> ::= 
    'TRAIL'  |  'ACYCLIC'  |  'SIMPLE'

<node_pattern>  ::=  '(' <node_labeled> <<where_clause>>? ')'
      
<node_labeled> ::= 
      <node_variable> ':' <label_descr>
    | <node_variable> 
    |                 ':' <label_descr>
      
<node_variable> ::=  <<identifier>>

<label_descr> ::=              //Simplified. GPML provides %, !, |, &, grouping. 
      <label_name>
      
<label_name> ::=  <<identifier>>

<edge_pattern> ::=  <edge_spec> <quantifier>?

<group_pattern> ::=
      '(' <path_pattern> <<where_clause>>? ')' <quantifier>?
    | '[' <path_pattern> <<where_clause>>? ']' <quantifier>?

<quantifier> ::=
      '+'  |  '*'
    | '{' <<integer>> ',' <<integer>>? '}'
    
<edge_spec> ::= 
      '<-'  |  '<-' '[' <edge_filtered> ']' '-'  
    | '~'   |  '~'  '[' <edge_filtered> ']' '~' 
    | '->'  |  '-'  '[' <edge_filtered> ']' '->' 
    | '<~'  |  '<~' '[' <edge_filtered> ']' '~' 
    | '~>'  |  '~'  '[' <edge_filtered> ']' '~>' 
    | '<->' |  '<-' '[' <edge_filtered> ']' '->' 
    | '-'   |  '-'  '[' <edge_filtered> ']' '-' 
    
<edge_filtered> ::=  <edge_labeled> <<where_clause>>?
    
<edge_labeled> ::= 
      <edge_variable> ':' <label_descr>
    | <edge_variable> 
    |                 ':' <label_descr>
    
<edge_variable> ::=  <<identifier>>
```

It is worthy of a note that `<<where_clause>>`, referenced in the above grammar 
in node, edge, and group patterns, is, syntactically, the `WHERE` clause of host PartiQL,
where it is defined as 
```EBNF
<where_clause> ::=  'WHERE' <expr_query>
```
where the expression `<expr_query>` is meant to evaluate to a boolean. 
This expession, however, can contain, as references, variables introduced as 
`<node_variable>`, `<edge_variable>`, or `<path_variable>` 
elsewhere in the pattern, as well as graph-specific predicates and functions 
applied to these references.   


## Evaluation Semantics

### Background: Evaluation in GPML

The details of graph pattern matching semantics are available in [^gpml-paper] (Section 6).
For the purposes of this RFC, it is sufficient to describe the structure of a
pattern-matching result.

Suppose we have a GPML graph pattern of the form
>  _P<sub>1</sub>_ *,* _..._ *,* _P<sub>n</sub>_

where _P<sub>i</sub>_ are path patterns.
(We abstract away from the possibility that a `<graph_pattern>` can start with a selector,
since that does not affect the structure of the result.)
The path patterns can contain variables marking node, edge, or path subpatterns.
The variables that occur in the scope of a quantifier
(such as `+`, `*`, `{2,7}`, or `{2,}`) are _group variables_,
while the remaining ones are _singleton variables_.

The intent of pattern matching is to find all (possibly overlapping) fragments
of the graph, each of which is described by the pattern
_P<sub>1</sub>_ *,* _..._ *,* _P<sub>n</sub>_, and associate
the pattern's variables with elements of the graph from the fragment.
In a given match result (fragment), a singleton variable gets associated to
one graph element, while a group variable can get associated to none or several
elements -- one for each repetition of the containing quantified subpattern.  
A singleton variable `x` for a node or an edge can occur in more than one
path pattern _P<sub>i</sub>_, in which case it is meant to be bound,
in a given result, to the same node or edge across all path patterns.
For the sanity of this semantics, it follows that this is not
allowed for group variables.


Inferring from [^gpml-paper] (Section 6), the result of evaluating a GPML pattern
on a graph can be characterized as
a bag of matching graph fragments, each being an ordered tuple of the form
>  _p<sub>1</sub>_ *,* ...*,* _p<sub>n</sub>_

with each _p<sub>i</sub>_ being an annotated path from the graph corresponding
to the path pattern _P<sub>i</sub>_.
In turn, an annotated path _p<sub>i</sub>_ is a path from the graph
(a sequence of alternating nodes and edges)
that is annotated with variables from the corresponding path pattern _P<sub>i</sub>_.
Among these annotations,
- A singleton variable can be used multiple times, possibly in different
  paths _p<sub>i</sub>_ and _p<sub>j</sub>_, but always annotating the same node or edge,
  in a given matching fragment.
- A group variable annotates zero or more elements within one of the
  _p<sub>i</sub>_ paths in the match.
- A path variable annotates a contiguous subpath of one of the
  _p<sub>i</sub>_ paths (which could be the entirety of _p<sub>i</sub>_).

It is important to note that the variables are annotated to entities
in the graph (nodes, edges, paths) that are "aware" of their identity
and location within the graph.  This capability is 
essential within the pattern matching process, 
to determine sameness of nodes and edges when they are probed for 
being annotated with the same (singleton) pattern variable.


### Evaluation of Graph Pattern Matching in PartiQL

In the context of this RFC, the notion of a PartiQL value has been extended,
according to RFC-0025, to include graph values, while the notion of an expression
is being extended in this RFC by adding graph matching expressions.
Furthermore, a graph matching expression can contain PartiQL expressions,
notably within the filtering `WHERE` clauses.
<!-- They, in turn, can contain graph-specific predicates like `IS DIRECTED` and
`ALL_DIFFERENT`.  -->

A PartiQL expression is evaluated in an environment that maps variables to PartiQL values,
with the result of an evaluation being a PartiQL value.  
The above-outlined evaluation semantics of graph pattern matching in GPML is
structured differently (for one, it produces a bag of tuples of annotated paths)
and operates over values of a kind that are not part
of PartiQL data model, even after the RFC-0025 extension (such as nodes and edges
that have awareness of their identity and location within a graph).
In a sense, graph matching is a distinct query language of its own, embedded
within PartiQL as a `MATCH` expression form.

Consequently, our goal here is to describe how the two semantics are combined,
specifically addressing:
- Role of PartiQL environments while evaluating pattern matching.
- Evaluation of PartiQL expressions within graph pattern `WHERE` clauses.
- And, crucially, how the result of pattern match computation is
  represented as a value within the PartiQL data model.


For evaluating a  graph matching expression
>    `(` _graph_ `MATCH` _pattern_ `)`

this RFC  proposes the following.

- As any PartiQL expression, the graph matching expression
  is evaluated within a regular PartiQL environment (𝝆<sub>0</sub>, 𝝆) that maps variables
  (defined outside this expression) to values.

- The _graph_ subexpression is evaluated within (𝝆<sub>0</sub>, 𝝆) and must result
  in a graph value (in the sense of RFC-0025).

- Given the graph value, graph matching computation for _pattern_ proceeds according
  to the GPML semantics, with the following modifications:

  - Variables defined in _pattern_ shadow variables bound in (𝝆<sub>0</sub>, 𝝆).

  - Whenever a `WHERE e` clause within the pattern needs to be applied,
    the clause's expression `e` is evaluated within (𝝆<sub>0</sub>, 𝝆)
    though regular PartiQL evaluation, except that any pattern-defined
    variable `x` is not looked up in (𝝆<sub>0</sub>, 𝝆).
    Instead, its value, as a graph element _v_, is provided by the GPML
    pattern matching procedure.  
    To make the evaluation proceed further, 
    according to regular PartiQL evaluation rules, _v_ is 
    replaced with its payload: 
    - If _v_ is a node or an edge, **payload(**_v_**)** is used.
    - If _v_ is a path, it is an error, according to this RFC.
<!-- To support graphical predicates, it would be something like this, 
     instead of the above paragraph:
    The subsequent processing of the graph-element value _v_ depends on the
    context of `x` within `e`:
    - If `x` is used within a graph-specific construct (such as a predicate
      like `IS DIRECTED` or a path-aggregating function), the graph-element
      value _v_ is passed to this construct directly.
    - Otherwise, `x` occurs in a regular PartiQL context, in which case it is 
      replaced with its payload:
      - If _v_ is a node or an edge, **payload(**_v_**)** is used.
      - If _v_ is a path, it is an error, according to this RFC.
-->

- Upon completion of the GPML pattern matching computation, the resulting
  bag of matching graph fragments is converted into a bag of structs
  by converting each fragment into a struct as follows.  
  Given a fragment of the form
  >  _p<sub>1</sub>_ *,* ...*,* _p<sub>n</sub>_

  where _p<sub>i</sub>_ are variable-annotated paths,
  the resulting struct gets key/value attribute pairs where keys are variable names
  while the overall attribute for variable `x` is determined is follows.

  - If the `x` is a singleton variable that annotates a node or an edge _v_,
    the value of its `x`'s attribute is **payload(**_v_**)**.
  - If `x` is a group variable, the value of its attribute is
    the list of payloads at the elements (nodes or edges) annotated with `x`,
    taken in the order of their appearance in the fragment's path.

  - If `x` is a path variable, it does not produce an attribute in the struct,
    according to this RFC.

This semantics naturally supports the important property lookup use case 
from GPML[^gpml-paper] on property graphs. That is, evaluating an expression 
of the form _v_**.a** where _v_ is a node or an edge and **a** is a property name: 
- In GPML[^gpml-paper], **.a** is a property lookup operation 
  specifically defined for graph nodes and edges. 
- In PartiQL, **.a** remains the operation of a key lookup in a struct, 
  which would succeed under the above semantics 
  as long as _v_ has a struct as its payload.


## Implementation-Dependent Aspects

Certain aspects of graph pattern matching behavior can be implementation-dependent. 
This section identifies such aspects, as well as possibilities that an implementation 
may choose to adopt for a given aspect. The chosen possibility must be clearly documented. 
An implementation may also support several of the designated possibilities. 
In this case, it must provide a configuration mechanism for a user to choose 
the desired one.

### Global Restrictors for Graph Patterns

A path subpattern in a graph pattern can be marked with a `TRAIL` or `ACYCLIC`
restrictor (also `SIMPLE`) that restricts the result bag of otherwise-matching paths
to those that additionally do not repeat an edge (for `TRAIL`) 
or do not repeat a node (for `ACYCLIC`). 

It is conceivable to have similar restrictors that apply to the whole graph pattern, 
but it appears this is not the design in GPML and SQL/PGQ.  
Instead, PartiQL defines this as an implementation-dependent behavior. 
That is, dependent on an application's choice, the bag of annotated graph fragments
computed as matches for any graph pattern is allowed to contain only certain fragments:
 - **repeats ok**: any matching fragment is allowed;
 - **no repeat nodes** a fragment is allowed only if no node occurs twice; 
 - **no repeat edges** a fragment is allowed only if no edge occurs twice;
 - **no repeat elements** a fragment is allowed only neither an edge nor a node occurs twice. 

Note that these modes, when enabled, can render relevant path restrictors 
(`TRAIL`, `ACYCLIC`, `SIMPLE`) unnecessary.


# Prior Art
[prior-art]: #prior-art

Section 3 in [^gpml-paper] contains an informative overview of modern query languages 
with graph matching.


# Unresolved Questions
[unresolved-questions]: #unresolved-questions

### MATCH-Associated WHERE

The pattern grammar proposed in this RFC supports a variant of `WHERE` clause within 
node, edge, and path (grouping) patterns. 
In a few places, [^gpml-paper] also mentions `WHERE` clause that is not a part of any
node, edge, or path fragment of a pattern,
but rather applies to the entirety of the pattern in a graph `MATCH` expression.
This `WHERE` is sometimes referred to as a "postfilter" of a pattern.
From the limited discussion in the paper and the lack of examples on the matter,
it is not fully clear whether such `WHERE` is meant to be the `WHERE` of the "host"
select-from-where query (in SQL or PartiQL) or it is a new kind of `WHERE`
that is coupled with `MATCH`.

In the case of the latter, the syntax of graph match expression would have to accommodate
this possibility with a grammar production like
```EBNF
<expr_query> ::=
  // ...
   |  '(' <expr_query> 'MATCH' <graph_pattern> [ 'WHERE' <expr_query> ]?  ')'
```

Currently, this is not being proposed for the reasons of economy of design --
until and unless it becomes certain that the "host" `WHERE`
cannot support necessary use cases.


### Graphical Predicates and First-Class Nodes/Edges 

GPML[^gpml-paper] supports several _graphical predicates_, such as
_e_ `IS DIRECTED`, _n_ `IS SOURCE OF` _e_, _n_ `IS DESTINATION OF` _e_,
`SAME(`_v1_`,` _v2_`,` ...`)`,  `ALL_DIFFERENT(`_v1_`,` _v2_`,` ...`)`,
which can be used in `WHERE` filters associated with nodes, edges, and paths (groupings), 
as well as, presumably, in the post-filter `WHERE` associated with the overall pattern
(see the previous section). 

Evaluation of these predicates, naturally, requires that nodes _n_ and edges _e_ 
to which they are applied are entities that are "aware" of their location within
a graph _G_ to which they belong.  
In particular, the values _n_ and _e_ cannot be merely the payloads 
taken from nodes and edges of _G_. 

Since the semantics proposed in this RFC equates graph elements with their payloads
for all purposes except the internal mechanics of pattern matching computation,
this RFC does not specify support for the graphical predicates.  

Properly supporting them would require modeling values of nodes and edges 
as something like _in-graph references_ of the form (_id_, _G_) or similar
and, if we are to support property lookups _v_**.a** as naturally as possible so far, 
introducing rules on when an in-graph reference is treated as itself vs. 
when it is reduced to its payload.

Then, an important question would be whether these in-graph references were to become
first-class values in PartiQL or be limited to the scope of `MATCH` expressions.
- If they were to become first-class, then the post-filter `WHERE` could likely 
  coincide with the "host" PartiQL `WHERE`.  However, this would significantly 
  complicate the overall data model: in addition to graph values, it would 
  be expanded with in-graph reference values, of variants specific to nodes, edges,
  and, possibly, paths.
  Anticipating these consequences makes it worthwhile to investigate 
  whether such reference could be an instance of some more general construct, 
  perhaps relevant to update operations in non-graph parts of the data model as well.
- If they were to be limited in scope, this would give further justification
  to adding the post-filter `WHERE` as an option to `MATCH` expressions. 
  The non-graph PartiQL data model would remain simple, 
  while the overall language should still be able to support all interesting use cases.
  It seems likely this approach is being chosen for SQL/PGQ.


### Expressing Path Bindings in MATCH Result

While GPML patterns support path variables, which bind to graph paths within 
graph fragments that constitute the result of a pattern match, 
this RFC does not specify how to represent these path bindings in the 
final result of a PartiQL `MATCH` expression. 
This is obvious from the explicit exclusions in two places in the evaluation semantics
where paths are encountered.

While representing a singleton variable bound to a node or an edge 
as the latter's payload
fairly naturally translates to representing a group variable 
as a sequence of payloads for the nodes or edges that the variable annotates, 
settling on a natural way to extend this approach to path variables
is not straightforward. 

In the first approximation, the MATCH-result residual
of a graph path bound to a path variable should probably be a list 
composed of payloads of certain nodes and edges on the path. 
However, clarifying details of this idea leads to questions such as
- Should this residual only contain payloads of elements of the path named by 
  node/edge variables or should it include anonymous elements as well?
- Should we address the double representation of node/edge variables
  (once as the top-level attributes in the result struct and then again
  within the path's residual)?
  - Is there a way to express the connection between the two representations?
  - Or should those variables that occur within a path be only present in the 
    path's residual and not on the top level?
    But then placement of node/edge variables in the result struct
    would become sensitive to the presence or absence of path variables, 
    which is a disadvantage relative to the always-on-top-level placement. 
- Do nested named path patterns lead to a hierarchical structure of the 
  result struct, or should these paths be "flattened out"? 

While it could be possible to cobble together a "reasonable" solution based on 
some choices from the above, the choices would be essentially arbitrary.
At the same time, this design challenge is likely to be addressed in SQL/PGQ, 
so it would be fitting for the PartiQL solution to be consistent. 


# Future Possibilities
[future-possibilities]: #future-possibilities

Further improvements of graph support in PartiQL can be
- Syntax for literal graph values.
- Create, update, and delete functionality within a graph.

# References
[references]: #references

[^gpml-paper]: Alin Deutsch et al., 
  [Graph Pattern Matching in GQL and SQL/PGQ]( https://doi.org/10.1145/3514221.3526057).
  SIGMOD'22: Proceedings of the 2022 International Conference on Management of Data,
  June 2022, pp. 2246–2258.
  [PDF](https://arxiv.org/pdf/2112.06217.pdf)
