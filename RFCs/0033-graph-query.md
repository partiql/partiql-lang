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


# Motivation
[motivation]: #motivation

See RFC-0025 "Graph Data Model".


# Guide-level explanation
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

With the result of a graph matching expression being a bag of structs, the expression 
can occur anywhere in a PartiQL query, as any other expression.  In particular, 
it can occur as a data source in a `FROM` clause of a query, which is the primary 
intended usage.

TODO? An example query. (Within FROM or independent of it?)

Computation of a pattern match involves navigation around the graph, but
graph navigation capabilities are supported only within the scope of this computation.
The result table that gets out does not carry any implicit info about the graph's structure, 
it only carries the "payload" information from the matched graph elements (see RFC-0025).
In a sense,  what is inside a MATCH expression's pattern is a separate query language, 
distinct from PartiQL, and this language also has its own distinct evaluation mechanisms.


## Relation to GPML and SQL/PGQ 

<!-- Better in RFC-0025 ? -->
GPML is the name given to the graph pattern matching language described in a 
publicly available paper [^gpml-paper] that previews such a language developed for 
SQL/PGQ and GQL, the upcoming standards from ISO and IEC. 

- GPML
  - TODO: what it is
  - GPML is for a more concrete data model (RFC-0025 is more general: allows any PartiQL value 
    as a payload at a graph element, not just something that mimics GPML's key/value properties).
  -  `MATCH pat` -- the graph is implicit, vs `(graph MATCH pat)` -- the input graph is explicit.
- SQL/PGQ
  - TODO: what it is
  - This RFC describes for PartiQL what SQL/PGQ is expected to describe for SQL: 
    describe GPML (by reference or by inclusion) and specify how it is integrated 
    into the host language.
  - There is no detailed public information available at the moment about SQL/PGQ. 
    Consequently, the proposal in this RFC may need adjustments (more likely in syntax,
    but possibly in semantics as well) after SQL/PGQ becomes available.


## Internal semantics of graph pattern match

TODO: given a PartiQL binging environment, a graph _graph_, a pattern _pattern_, 
what does it mean to evaluate the graph pattern match expression _graph_ `MATCH` _pattern_ 
in the environment?

What's inside a MATCH is, essentially, a separate query language, distinct from PartiQL/SQL.

- For most of the details, defer to [^gpml-paper]
- Essential for this RFC: the "info model" of the result. 
  -- We **probably** can describe the result as a bag of binding tuples ... 
  -- ... but a pattern variable in such a tuple is bound to a graph element 
     that is "aware" of its location within the graph, 
     not just carries the "data payload" at that location. 
- This "locational info" is essential for supporting graph pattern features such as
  - Sameness when the variable is repeated in the pattern.
  - Explicit equality/inequality between variables.
  - Postfilter predicates: e IS DIRECTED, s IS SOURCE OF e, and d IS DESTINATION OF e [p12; sec 4.7].
  - SAME(e1, e2, ...),  ALL_DIFFERENT(e1, e2, ...) [sec 4.7]

TODO: Also, a pattern can have "regular" PartiQL expressions embedded into it.  
Need to explain how to construct a binding environment in which such an expression is 
evaluated, as part of pattern matching process. 


## External semantics of graph pattern match

The "external" result of a MATCH expression is obtained by stripping away 
graph locational info from the "internal" result and materializing it as a bag of structs. 

TODO: What happens to path variables?  
- They do not have "natural" data payload.
- Probably the only way a path variable can transpire in the final table is as 
  a grouping mechanism for (data payloads of) other 
  variables occurring inside the path. 
- [^gpml-paper] indicates that a path can be aggregated over, but the only example
  is where the aggregate (path length) is used in a comparison in a MATCH WHERE clause.
  Conceivably, they aggregate could be wanted in the output as well, 
  but [^gpml-paper] does not discuss a mechanism for that. 


## Grammatical details

### Idealized grammar

Ideally, graph pattern matching would be added to PartiQL grammar as a new production 
to the non-terminal `<expr_query>` (which defines constructs like scalar, struct, 
and collection literals; function calls; parenthesized SELECT-FROM-WHERE, etc.):

```EBNF
<expr_query> ::= 
   // ...
   |  <expr_query> 'MATCH' <graph_pattern>
   
<graph_pattern> ::= 
   <selector>? <path_pattern> [',' <path_pattern>]...
```
where `<graph_pattern>`, as well as the non-terminals on which it depends, are defined as in GPML. 

After this, the match expression would become available, as an `<expr_query>`, 
to be used as a source in the `FROM` clause of SELECT-FROM-WHERE queries, which is defined as 

````EBNF
<from_clause> ::=  
  FROM <from_item> [ ',' <from_item> ]...
  
<from_item> ::= 
    <expr_query> [ 'AS' <id> [ AT <id> ]? ]?
  // ... joins, etc
````
and, as a grammar that would not need any change. 

Unfortunately, such straightforward modifications would introduce ambiguities, making it impossible
to parse for this grammar with one-token lookahead: the new rule for `<expr_query>` 
is left-recursive, and there could be uses of `,` for which it is impossible to decide whether 
they separate instances of `<path_pattern`s or `<from_item>`.


### More practical grammar

To address these parsing difficulties, there are a couple tweaks to the above grammar. 

First, a match expression, in general, must be parenthesised. That is, `<expr_query>` 
is extended as 

```EBNF
<expr_query> ::=
  // ...
   |  '(' <expr_query> 'MATCH' <graph_pattern> ')'
   
<graph_pattern> ::= 
   <selector>? <path_pattern> [',' <path_pattern>]...
```

Second, the parenthesation restriction is relaxed in a `FROM` clause: when 
a match expression is used as a source of data _and_ its pattern consists of a 
single path pattern, then the parentheses are not necessary: 

```EBNF
<from_clause> ::=  
  FROM <from_item> [ ',' <from_item> ]...
  
<from_item> ::= 
    <expr_query> [ 'AS' <id> [ AT <id> ]? ]?
  | <expr_query> 'MATCH' <selector>? <path_pattern> [ 'AS' <id> [ AT <id> ]? ]?
  // ... joins, etc
````

### On MATCH-WHERE clause

In a few places, [^gpml-paper] mentions `WHERE` clause -- sometimes referred to as 
a "postfilter" -- that is not a part of any node, edge, or path fragment of a pattern, 
but rather applies to the entirety of the pattern in a graph match expression. 
From the limited discussion in the paper and the lack of examples on the matter, 
it is not clear whether such `WHERE` is meant to be the `WHERE` of the "host" 
select-from-where query (in SQL or PartiQL) or it is a new kind of `WHERE` 
that is coupled with `MATCH`. 

In the case of the latter, the syntax of graph match expression woulf have to accommodate 
this possibility:
```EBNF
<expr_query> ::=
  // ...
   |  '(' <expr_query> 'MATCH' <graph_pattern> [ 'WHERE' <expr_query> ]?  ')'
```

Currently, this is not being proposed for the reasons of economy, 
until and unless it becomes clear that the "host" `WHERE` 
cannot support necessary use cases.


## Implementation-dependent aspects

- Implementation-dependent choice (or config modes) for:
  DIFFERENT NODES / DIFFERENT ELEMENTS ; TRAIL / ACYCLIC
  - [Or have this in the "Internal semantics" section?] 


# Drawbacks
[drawbacks]: #drawbacks

~~Why should we *not* do this?~~

- It is a very sizable extension of the data model and the language.
  The size is on the order of magnitude of what is there in PartiQL already. 
  - Specification and reference implementation effort (PartiQL team).
  - Implementation effort in a host system (customer teams).
    - That is, not everyone might want to implement this. 
      This might make more acute the need for defining PartiQL subsets/profiles
      and supporting them in the reference implementations. 



# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- ~~Why is this design/proposal the best in the space of possible designs?~~
- ~~Which other designs/proposals have been considered, and what is the rationale for not choosing them?~~
- ~~What is the impact of not doing this?~~

- MATCH expression (here) vs MATCH source in a FROM clause.

- GRAPH_TABLE, seen in some public slideware about GQL. 


# Prior art
[prior-art]: #prior-art

~~Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:~~

- ~~For specification proposals: Does this feature exist in any ISO SQL standard or other SQL dialects?~~
- ~~For API changes: Do similar APIs exist in libraries such as Calcite? What are some details of the specific implementation?~~
- ~~Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.~~

~~This section is intended to encourage you, as an author, to think about the lessons from other SQL dialects; provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other dialects and implementations.~~

~~Note that while precedent set by other dialects and libraries is some motivation, it does not on its own motivate an RFC.~~

- See Section 3 in [^gpml-paper]

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- GPML-WHERE vs SQL-WHERE.
  - aka Should we have (graph MATCH pattern WHERE ...) as the expression form? 
  - The IS_DIRECTED, SAME (etc.) predicates would often make sense only in a WHERE clause,
    but to work there, the variables involved must be bound to graph elements, not merely to their content,
    so we seem to need a variant of WHERE clause that is part of MATCH, distinct from the PartiQL/SQL MATCH!

- Syntactic delineation of the graph match expression. 
  - The current syntax requires parentheses around the expression, almost everywhere.  
    This is unusual - there is no other prior construct like that.
  - GRAPH_TABLE(...) might be the approach in SQL/PGQ.

- "Locational" information: should it seep into the rest of PartiQL semantics? 
  - Might be already necessary in order to evaluate PartiQL expressions embedded within patterns. 
  - Could be useful for the semantics of updates (not only in graphs, 
    but in "regular" PartiQL data).

- ~~What parts of the design do you expect to resolve through the RFC process before this gets merged?~~
- ~~What parts of the design do you expect to resolve through the implementation of this feature before stabilization?~~
- ~~What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?~~

# Future possibilities
[future-possibilities]: #future-possibilities

# References
[references]: #references

[^gpml-paper]: [Graph Pattern Matching in GQL and SQL/PGQ](https://arxiv.org/pdf/2112.06217.pdf)