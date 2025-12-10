# PartiQL Specification - Table of Contents

## 1. Introduction
   1.1. Document Purpose and Audience  
   1.2. PartiQL Core vs. Syntactic Sugar  
   1.3. Gradual Typing  
   &nbsp;&nbsp;&nbsp;&nbsp;1.3.1. Schema Validation  
   &nbsp;&nbsp;&nbsp;&nbsp;1.3.2. Type Inference  
   1.4. Execution Modes
   &nbsp;&nbsp;&nbsp;&nbsp;1.4.1. Permissive Mode  
   &nbsp;&nbsp;&nbsp;&nbsp;1.4.2. Type Checking Mode  

## 2. Data Types
   3.1. Numeric Types  
   3.2. Character Types (Text/String Types)  
   3.3. Timestamp Types  
   3.4. Boolean Types  
   3.5. Binary Types (LOB Types)  
   3.6. Tuple/Struct Types
   &nbsp;&nbsp;&nbsp;&nbsp;3.6.1. Ordered vs. Unordered Tuples  
   &nbsp;&nbsp;&nbsp;&nbsp;3.6.2. Attribute Names and Values  
   &nbsp;&nbsp;&nbsp;&nbsp;3.6.3. Duplicate Attribute Names  
   3.7. Collection Types  
   &nbsp;&nbsp;&nbsp;&nbsp;3.7.1. Arrays  
   &nbsp;&nbsp;&nbsp;&nbsp;3.7.2. Bags
   3.8. Absent Values (NULL and MISSING)
   3.8. Type System and Ion Integration  

## 3. Type Conversion and Coercion
   4.1. Explicit Type Conversion (CAST)  
   4.2. Implicit Type Coercion  
   4.3. Literal Coercion for SQL Compatibility  

## 4. Database Environments
   5.1. Binding Tuples and Environments  
   5.2. Database Environment vs. Variables Environment  
   5.3. Qualified Names  
   5.4. Scoping Rules  
   &nbsp;&nbsp;&nbsp;&nbsp;5.4.1. Variable Scoping  
   &nbsp;&nbsp;&nbsp;&nbsp;5.4.2. Resolving Naming Conflicts  

## 5. Functions and Operators
   6.1. Logical Operators  
   6.2. Mathematical Functions and Operators  
   6.3. String Functions and Operators  
   6.4. Date/Time Functions and Operators  
   6.5. Comparison Operators  
   &nbsp;&nbsp;&nbsp;&nbsp;6.5.1. Equality and Deep Equality  
   &nbsp;&nbsp;&nbsp;&nbsp;6.5.2. Order-By Less-Than Function  
   6.6. Aggregate Functions  
   &nbsp;&nbsp;&nbsp;&nbsp;6.6.1. SQL Aggregate Functions  
   &nbsp;&nbsp;&nbsp;&nbsp;6.6.2. PartiQL Collection Functions (COLL_*)  
   6.7. Type Checking Functions (IS operators)  
   6.8. Path Expressions  
   &nbsp;&nbsp;&nbsp;&nbsp;6.8.1. Tuple Path Navigation  
   &nbsp;&nbsp;&nbsp;&nbsp;6.8.2. Array Navigation  
   &nbsp;&nbsp;&nbsp;&nbsp;6.8.3. Wildcard Steps  
   &nbsp;&nbsp;&nbsp;&nbsp;6.8.4. Path Collection Expressions  
   &nbsp;&nbsp;&nbsp;&nbsp;6.8.5. Handling MISSING and Type Errors  
   6.9. Operator Precedence  
   6.10. Handling Functions with Wrong Input Types  

## 6. PartiQL Query Syntax
   7.1. Query Structure (SFW Queries)  
   7.2. FROM Clause  
   &nbsp;&nbsp;&nbsp;&nbsp;7.2.1. Ranging Over Collections  
   &nbsp;&nbsp;&nbsp;&nbsp;7.2.2. UNPIVOT  
   &nbsp;&nbsp;&nbsp;&nbsp;7.2.3. JOIN Operations (CROSS JOIN, LEFT JOIN, FULL JOIN)  
   &nbsp;&nbsp;&nbsp;&nbsp;7.2.4. AT Clause for Position Variables  
   &nbsp;&nbsp;&nbsp;&nbsp;7.2.5. LATERAL keyword  
   7.3. LET Clause  
   7.4. WHERE Clause  
   7.5. GROUP BY Clause  
   &nbsp;&nbsp;&nbsp;&nbsp;7.5.1. Core GROUP BY with GROUP AS  
   &nbsp;&nbsp;&nbsp;&nbsp;7.5.2. SQL Compatibility Features  
   &nbsp;&nbsp;&nbsp;&nbsp;7.5.3. Aggregate Functions in GROUP BY  
   &nbsp;&nbsp;&nbsp;&nbsp;7.5.4. HAVING Clause  
   7.6. SELECT Clause  
   &nbsp;&nbsp;&nbsp;&nbsp;7.6.1. SELECT VALUE (Core)  
   &nbsp;&nbsp;&nbsp;&nbsp;7.6.2. SQL SELECT (Syntactic Sugar)  
   &nbsp;&nbsp;&nbsp;&nbsp;7.6.3. SELECT * Semantics  
   &nbsp;&nbsp;&nbsp;&nbsp;7.6.4. Tuple/Array/Bag Constructors  
   7.7. PIVOT Clause  
   7.8. ORDER BY Clause  
   &nbsp;&nbsp;&nbsp;&nbsp;7.8.1. ORDER BY Semantics  
   &nbsp;&nbsp;&nbsp;&nbsp;7.8.2. ASC/DESC and NULLS FIRST/LAST  
   &nbsp;&nbsp;&nbsp;&nbsp;7.8.3. PRESERVE Directive  
   7.9. LIMIT and OFFSET Clauses  
   7.10. Set Operations  
   &nbsp;&nbsp;&nbsp;&nbsp;7.10.1. UNION (ALL)  
   &nbsp;&nbsp;&nbsp;&nbsp;7.10.2. INTERSECT (ALL)  
   &nbsp;&nbsp;&nbsp;&nbsp;7.10.3. EXCEPT (ALL)  
   7.11. WITH Clause (Common Table Expressions)  

## 7. Subqueries and Subquery Coercion
   8.1. Scalar Subqueries  
   8.2. Collection Subqueries  
   8.3. Subquery Coercion Rules  

## Appendices
   A. Grammar Reference (Complete EBNF)  
   B. SQL Compatibility Guide  
   C. Examples and Use Cases  
   D. Implementation Considerations
