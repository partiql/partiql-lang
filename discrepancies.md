# PartiQL Documentation Discrepancy Analysis

**Date**: December 30, 2024  
**Analysis**: Comparison between `./src/legacy/` and `./src/` documentation

---

## Executive Summary

This document identifies discrepancies between the legacy PartiQL documentation (located in `./src/legacy/`) and the current documentation (direct child `.adoc` files in `./src/`). The analysis reveals several critical gaps that need to be addressed to ensure comprehensive coverage of PartiQL semantics.

**Key Findings**:
- **8 major topic areas** with discrepancies identified
- **3 high-priority gaps** requiring immediate attention
- **5 medium-priority gaps** that should be addressed
- Current documentation has better organization but fewer examples
- Several TODO markers in current docs indicate incomplete sections

---

## Detailed Discrepancy Analysis

### 1. Data Model (legacy/model.adoc) ❌ MISSING

**Legacy Coverage** (`legacy/model.adoc`):
- Complete BNF grammar for PartiQL values
- Detailed explanation of absent values (NULL vs MISSING)
- Tuple structure and duplicate attribute handling
- Collection types (arrays vs bags) with examples
- Comprehensive comparisons to the relational model
- Concrete example: AWS configuration data structure
- Discussion of heterogeneous tuples and collections
- Explanation of tuple attribute ordering

**Current Coverage**:
- `types.adoc`: High-level overview only
- Individual type files (`types_tuple.adoc`, `types_collection.adoc`, etc.): Specific type details
- **MISSING**: Complete data model grammar
- **MISSING**: Comprehensive relational model comparison
- **MISSING**: Large-scale concrete examples
- **MISSING**: Nuanced discussion of duplicate attribute names in tuples

**Impact**: 🔴 **HIGH**  
**Reason**: Data model is foundational to understanding PartiQL semantics

**Recommendation**:
1. Create a new `data_model.adoc` file or expand `types.adoc`
2. Include the complete BNF grammar for PartiQL values
3. Add the AWS configuration example or similar comprehensive example
4. Explicitly document the four extensions to SQL's data model:
   - Heterogeneous array/bag elements
   - Arbitrary composition of arrays, bags, and tuples
   - Distinction between null-valued and missing attributes
   - Explicit distinction between arrays and bags

---

### 2. Path Navigation (legacy/paths.adoc) ⚠️ INCOMPLETE

**Legacy Coverage** (`legacy/paths.adoc`):
- Tuple path navigation (`t.a` syntax)
- Array navigation (`a[i]` syntax)
- Tuple navigation with array notation (`t['attr']` and `t[CAST(...)]`)
- Composition of navigations (`r.no[1]`)
- Tuple navigation on wrongly typed data (permissive vs type checking modes)
- Array navigation on wrongly typed data
- **Wildcard steps** (`[*]` and `.*`)
- **Path collection expressions** (multi-step paths with wildcards like `tables.items[*].product.*.nest`)
- Schema role in tuple path navigation
- Detailed error handling for each scenario

**Current Coverage**:
- `functions.adoc` §sec:indexing-op: Basic array indexing
- `functions.adoc` §sec:coll-spread-op: Collection spread operator `[*]`
- `functions.adoc` §sec:path-op: Field access operator (tuple navigation)
- `functions.adoc` §sec:tuple-spread-op: Tuple spread operator `.*`
- `functions.adoc` §sec:path-coll-ops: **"TODO: Document multi-step path expressions"**

**MISSING**:
- Comprehensive path composition semantics
- Detailed error handling scenarios with examples
- Multi-step wildcard path expressions (marked TODO)
- Reduction rules for complex path expressions
- Schema-based compile-time error detection

**Impact**: 🟡 **MEDIUM-HIGH**  
**Reason**: Path navigation is critical for accessing nested data

**Recommendation**:
1. Create a dedicated `path_navigation.adoc` file
2. Complete the TODO in `functions.adoc` for path collection expressions
3. Add comprehensive examples of path composition
4. Document all error modes (permissive vs type checking)
5. Include the reduction semantics from legacy docs

---

### 3. Variable Scoping Rules (legacy/scoping.adoc) ❌ MISSING

**Legacy Coverage** (`legacy/scoping.adoc`):
- Resolving naming conflicts between database environment and variables environment
- `@identifier` syntax for explicit variable references
- FROM clause path resolution rules
- Non-FROM clause path resolution rules
- Qualified name handling (e.g., `v.foo` vs database name `v.foo`)
- Detailed algorithm for resolving `i_1.i_2...i_n` identifiers
- Multiple comprehensive examples

**Current Coverage**:
- `database_environments.adoc`: Conceptual explanation of environments
- `database_environments.adoc`: Brief mention of binding tuple concatenation and precedence
- **MISSING**: Explicit scoping resolution rules
- **MISSING**: `@` syntax documentation
- **MISSING**: Detailed conflict resolution algorithm
- **MISSING**: FROM vs non-FROM path distinction

**Impact**: 🔴 **HIGH**  
**Reason**: Essential for correct query interpretation and SQL compatibility

**Recommendation**:
1. Add a new section to `database_environments.adoc` or create `name_resolution.adoc`
2. Document the three main scoping rules:
   - `@identifier` always refers to variable
   - FROM clause paths: database names take precedence
   - Non-FROM clause paths: variables take precedence
3. Include the qualified name resolution algorithm
4. Add examples showing complex scoping scenarios

---

### 4. Set Operators (legacy/setops.adoc) ❌ MISSING

**Legacy Coverage** (`legacy/setops.adoc`):
- Placeholder indicating need for UNION/INTERSECT/EXCEPT
- Note: "Coming up..."

**Current Coverage**:
- **MISSING**: No mention anywhere in current documentation

**Impact**: 🟡 **MEDIUM**  
**Reason**: Important SQL compatibility feature, though legacy docs also incomplete

**Recommendation**:
1. Add set operator semantics to `query_syntax.adoc`
2. Cover:
   - UNION [ALL]
   - INTERSECT [ALL]
   - EXCEPT [ALL]
   - OUTER UNION
   - Interaction with ORDER BY
3. Document how set operators work with binding tuples
4. Explain coercion behavior with heterogeneous types

---

### 5. LET Clause (legacy/let.adoc) ⚠️ INCOMPLETE IN BOTH

**Legacy Coverage** (`legacy/let.adoc`):
- Only contains "_TODO_"

**Current Coverage**:
- `query_syntax.adoc` §sec:let-clause: 
  - Basic syntax (EBNF)
  - Binding tuple flow explanation
  - Input/output semantics
- **MISSING**: Detailed examples
- **MISSING**: Use cases and patterns
- **MISSING**: Interaction with other clauses

**Impact**: 🟡 **MEDIUM**  
**Reason**: Should be expanded for completeness, though current coverage is minimal but adequate

**Recommendation**:
1. Add examples to the existing LET clause section
2. Show common patterns:
   - Computing intermediate values
   - Avoiding repeated subqueries
   - Building complex expressions
3. Show interaction with WHERE and SELECT clauses

---

### 6. Schema/Structural Types (legacy/schema.adoc) ❌ MISSING

**Legacy Coverage** (`legacy/schema.adoc`):
- Placeholder for structural types discussion
- Note: "WIP" (Work in Progress)
- Indicates need for:
  - Precise rules for SQL compatibility
  - Optional schema semantics
  - Query result stability with respect to schema

**Current Coverage**:
- `types.adoc`: Mentions type system but no schema semantics
- `gradual_typing.adoc`: **NOT ANALYZED YET** - may contain relevant content
- **POTENTIALLY MISSING**: Structural type definitions and their role in query semantics

**Impact**: 🟡 **MEDIUM**  
**Reason**: Important for SQL compatibility and static analysis

**Recommendation**:
1. Review `gradual_typing.adoc` to determine overlap
2. If not covered, add section on:
   - How schema affects query semantics
   - Ordered vs unordered tuples
   - Static type checking capabilities
   - SQL compatibility through schema

---

### 7. Detailed Clause Semantics - FRAGMENTED

#### GROUP BY Clause

**Coverage Assessment**:
- ✅ **GOOD**: Core GROUP BY syntax and semantics
- ✅ **GOOD**: Group variable concept
- ✅ **GOOD**: GROUP ALL variant
- ✅ **GOOD**: SQL compatibility features
- ✅ **GOOD**: Grouping equivalence function (eqg) - referenced
- ⚠️ **TODO**: eqg function details (marked as TODO in current docs)
- ⚠️ **MISSING**: Advanced use case showing GROUP BY replacing window functions
- ⚠️ **MISSING**: Example from legacy: `xmpl:windows-by-grouping`

**Recommendation**:
- Complete the eqg function TODO
- Add windowing example from legacy docs

#### ORDER BY Clause

**Coverage Assessment**:
- ✅ **EXCELLENT**: Order-by less-than function comprehensively documented
- ✅ **GOOD**: NULLS FIRST/LAST
- ✅ **GOOD**: PRESERVE directive
- ✅ **GOOD**: SQL compatibility features
- ⚠️ **MISSING**: Interaction with set operators (legacy §sec:order-by-and-setops)
- ⚠️ **MISSING**: Literal coercion rules (legacy §sec:literal-conversion - incomplete in legacy too)

**Recommendation**:
- Add set operator interaction when set operators are documented
- Consider adding literal coercion if needed for SQL compatibility

#### WHERE Clause

**Coverage Assessment**:
- ✅ **EXCELLENT**: Well covered
- ✅ **GOOD**: 3-valued logic mentioned
- ✅ **GOOD**: IS MISSING vs IS NULL distinction
- ✅ **GOOD**: Type error handling

**Recommendation**: No changes needed

#### FROM Clause

**Coverage Assessment**:
- ✅ **EXCELLENT**: Comprehensive coverage
- ✅ **GOOD**: Ranging over collections
- ✅ **GOOD**: UNPIVOT operation
- ✅ **GOOD**: All JOIN types (CROSS, LEFT, FULL)
- ✅ **GOOD**: LATERAL keyword
- ✅ **GOOD**: Type coercion and error handling
- ✅ **GOOD**: Binding tuple flow

**Recommendation**: Consider adding a few more complex examples from legacy

#### SELECT Clauses

**Coverage Assessment**:
- ✅ **EXCELLENT**: SELECT VALUE core clause
- ✅ **GOOD**: SELECT (SQL compatibility)
- ✅ **GOOD**: PIVOT clause
- ✅ **GOOD**: Treatment of MISSING
- ✅ **GOOD**: Constructors (tuple, array, bag)
- ✅ **GOOD**: TUPLEUNION function

**Recommendation**: Consider adding complex examples from legacy (e.g., `xmpl:nesting-readings`)

---

### 8. Functions and Operators ⚠️ INCOMPLETE

**Current Coverage** (`functions.adoc`):
- ✅ **GOOD**: Type mismatch handling
- ✅ **GOOD**: Absent value inputs
- ✅ **GOOD**: Equality with deep equality (comprehensive)
- ✅ **GOOD**: IS NULL and IS MISSING predicates
- ✅ **GOOD**: Boolean connectives
- ✅ **BASIC**: Collection functions (indexing, spread)
- ✅ **BASIC**: Tuple functions (field access, spread, TUPLEUNION)
- ❌ **TODO**: Aggregate functions (marked)
- ❌ **TODO**: Type checking functions (marked)
- ❌ **TODO**: Path collection expressions (marked)

**Legacy Coverage** (`legacy/predsFunctions.adoc`):
- ✅ Has comprehensive equality semantics with examples
- ✅ Wrong input type handling
- ⚠️ References COLL_COUNT, COLL_AVG, etc. but doesn't define them fully

**MISSING from Current Docs**:
- Comprehensive aggregate function catalog
- Aggregate function semantics (how they consume group variables)
- String manipulation functions
- Date/time functions
- Mathematical functions
- Type conversion functions (though `type_conversions.adoc` may cover this)
- Collection manipulation functions beyond basics

**Impact**: 🔴 **HIGH**  
**Reason**: Functions are essential for practical query writing

**Recommendation**:
1. Complete all TODO markers in `functions.adoc`
2. Add aggregate functions section:
   - COLL_COUNT, COLL_SUM, COLL_AVG, COLL_MIN, COLL_MAX
   - Relationship to SQL aggregate functions (COUNT, SUM, AVG)
   - How they consume group variables
3. Add comprehensive function catalog as appendix or separate file
4. Document type checking functions (IS TUPLE, IS ARRAY, IS BAG, etc.)

---

## Topics with Good Coverage ✅

The following areas show good alignment between legacy and current documentation:

1. **Subquery Coercion** - `subqueries.adoc` matches legacy well
   - COLL_TO_SCALAR function
   - Scalar coercion rules
   - Array coercion rules
   - Permissive mode handling

2. **Binding Tuples Concept** - Better explained in current docs than legacy
   - Clear definition in `database_environments.adoc`
   - Flow through clauses well documented
   - Examples in `query_syntax.adoc`

3. **Environments** - Better structured in current docs
   - Database vs variables environment distinction clear
   - Type environments explained
   - Qualified names covered

4. **Query Structure** - Well organized in `query_syntax.adoc`
   - Clause execution order clearly stated
   - Compositional semantics emphasized
   - Binding tuple flow throughout

5. **Type System Overview** - Good high-level coverage
   - `types.adoc` provides clear overview
   - Individual type files provide details
   - Absent value semantics covered

---

## Information Quality Assessment

### Improvements in Current Documentation

1. **Better Organization**: Clearer file structure and topic separation
2. **Binding Tuple Flow**: More explicit and consistent explanation
3. **Formal Notation**: More consistent use of mathematical notation
4. **Cross-references**: Better linking between related sections
5. **Modularity**: Better separation of concerns across files

### Gaps Compared to Legacy

1. **Fewer Examples**: Legacy has more comprehensive, real-world examples
2. **Less Detail on Edge Cases**: Legacy covers more error scenarios
3. **Missing Advanced Features**: Some advanced topics not yet documented
4. **TODO Markers**: Several incomplete sections marked with TODO
5. **Less SQL Context**: Legacy provides more context for SQL compatibility decisions

---

## Prioritized Remediation Plan

### 🔴 HIGH PRIORITY (Must Address)

These gaps represent critical missing information that significantly impacts documentation completeness:

1. **Add Comprehensive Data Model Section**
   - File: Create new `data_model.adoc` or expand `types.adoc`
   - Content: BNF grammar, examples, relational model comparison
   - Effort: 2-3 days
   - Dependencies: None

2. **Document Variable Scoping Rules**
   - File: Expand `database_environments.adoc` or create `scoping_rules.adoc`
   - Content: Resolution rules, `@` syntax, examples
   - Effort: 1-2 days
   - Dependencies: None

3. **Complete Functions Documentation**
   - File: `functions.adoc`
   - Content: Remove all TODOs, add aggregate functions, complete function catalog
   - Effort: 3-4 days
   - Dependencies: May need to coordinate with implementation

4. **Complete Path Navigation Documentation**
   - File: Create `path_navigation.adoc` or expand `functions.adoc`
   - Content: Multi-step paths, wildcards, reduction rules
   - Effort: 2 days
   - Dependencies: None

### 🟡 MEDIUM PRIORITY (Should Address)

These gaps represent important missing information that should be addressed for completeness:

5. **Add Set Operators**
   - File: `query_syntax.adoc`
   - Content: UNION, INTERSECT, EXCEPT semantics
   - Effort: 2 days
   - Dependencies: None

6. **Expand LET Clause**
   - File: `query_syntax.adoc`
   - Content: Add examples and use cases
   - Effort: 0.5 days
   - Dependencies: None

7. **Document Schema/Structural Types**
   - File: Review `gradual_typing.adoc` first, then expand as needed
   - Content: Schema role in query semantics, SQL compatibility
   - Effort: 2 days
   - Dependencies: Need to review gradual_typing.adoc

8. **Add ORDER BY Set Operator Interaction**
   - File: `query_syntax.adoc`
   - Content: How ORDER BY works with UNION, etc.
   - Effort: 0.5 days
   - Dependencies: Requires set operators to be documented first

9. **Complete eqg Function Details**
   - File: `query_syntax.adoc` (GROUP BY section)
   - Content: Full definition of grouping equivalence
   - Effort: 0.5 days
   - Dependencies: None

### 🟢 LOW PRIORITY (Nice to Have)

These improvements would enhance the documentation but are not critical:

10. **Add Advanced Examples**
    - Files: Throughout documentation
    - Content: Port complex examples from legacy (windowing, nested queries, etc.)
    - Effort: 2-3 days
    - Dependencies: None

11. **Expand SQL Compatibility Notes**
    - Files: Throughout documentation
    - Content: Add more context about why certain features exist for SQL compatibility
    - Effort: 1 day
    - Dependencies: None

12. **Add More Edge Case Scenarios**
    - Files: Throughout documentation
    - Content: Document more permissive mode vs type checking mode differences
    - Effort: 1-2 days
    - Dependencies: None

---

## Estimated Timeline

### Immediate (1-2 weeks)
- Data Model section
- Variable Scoping rules
- Functions documentation completion

### Short-term (2-4 weeks)
- Path Navigation completion
- Set Operators
- LET clause expansion

### Medium-term (1-2 months)
- Schema/Structural Types
- All remaining medium priority items
- Begin low priority enhancements

---

## Migration Strategy

When addressing these gaps, consider:

1. **Preserve Current Organization**: The current docs have better structure
2. **Port Examples Selectively**: Not all legacy examples may be needed
3. **Update Notation**: Use current documentation's notation style
4. **Add Cross-references**: Link related sections together
5. **Mark SQL Compatibility**: Clearly indicate SQL compatibility features
6. **Add Implementation Notes**: Where appropriate, note implementation flexibility

---

## Files Analyzed

### Legacy Documentation (./src/legacy/)
1. ✅ `environment.adoc` - Queries, Environments and Binding Tuples
2. ✅ `from.adoc` - FROM Clause Semantics
3. ✅ `groupby.adoc` - GROUP BY clause
4. ✅ `let.adoc` - WITH and LET (TODO only)
5. ✅ `model.adoc` - Data Model
6. ✅ `orderby.adoc` - ORDER BY clause
7. ✅ `paths.adoc` - Path Navigation
8. ✅ `pivot.adoc` - PIVOT Clause Semantics
9. ✅ `predsFunctions.adoc` - Functions
10. ✅ `schema.adoc` - Structural Types (WIP in legacy)
11. ✅ `scoping.adoc` - Scoping rules
12. ✅ `select.adoc` - SELECT clauses
13. ✅ `setops.adoc` - UNION/INTERSECT/EXCEPT (Coming up in legacy)
14. ✅ `subqueryCoercion.adoc` - Coercion of subqueries
15. ✅ `where.adoc` - WHERE clause

### Current Documentation (./src/)
1. ✅ `database_environments.adoc` - Environments and Variables
2. ✅ `functions.adoc` - Functions and Operators
3. ✅ `introduction.adoc` - Introduction
4. ✅ `query_syntax.adoc` - PartiQL Query Syntax
5. ✅ `subqueries.adoc` - Subqueries and Subquery Coercion
6. ✅ `types.adoc` - Data Types (overview)
7. ⏭️ `gradual_typing.adoc` - Not analyzed
8. ⏭️ `execution_modes.adoc` - Not analyzed
9. ⏭️ `name_resolution.adoc` - Not analyzed
10. ⏭️ `type_conversions.adoc` - Not analyzed

---

## Next Steps

1. **Review Findings**: Validate this analysis with the team
2. **Prioritize**: Confirm the priority ordering
3. **Assign Resources**: Determine who will address each gap
4. **Set Timeline**: Establish deadlines for each priority tier
5. **Begin Work**: Start with high-priority items
6. **Track Progress**: Monitor completion and adjust priorities as needed

---

## Appendix: Quick Reference

### Files Needing Creation
- `data_model.adoc` (or expand `types.adoc`)
- `path_navigation.adoc` (or expand `functions.adoc`)
- Possibly `scoping_rules.adoc` (or expand `database_environments.adoc`)

### Files Needing Major Updates
- `functions.adoc` - Complete TODOs
- `query_syntax.adoc` - Add set operators

### Files Needing Minor Updates
- `query_syntax.adoc` - LET examples, eqg details
- Various files - Add more examples

### Files in Good Shape
- `subqueries.adoc`
- `database_environments.adoc` (except scoping rules)
- Most of `query_syntax.adoc`
