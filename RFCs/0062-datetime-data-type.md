# PartiQL Datetime RFC

## Summary

This RFC purposes the formal syntax and semantics for PartiQL’s datetime data types.

## Motivation

Date time types is among the primitive types specified in SQL spec. However, the PartiQL Specification as it stands today does not specify the syntax and semantics of date time types. The Kotlin reference implementation has put together an initial attempt for date time types but its syntax/semantics is not completed or documented anywhere. The motivation is to provide guidance to customers and library implementors through a formalized semantics of Date Time date type.

## Guide-level explanation

### Terminology

* Datetime types: The data types DATE, TIME, and TIMESTAMP are collectively referred to as datetime types.
* Coordinated Universal Time (UTC): the primary [time standard](https://en.wikipedia.org/wiki/Time_standard) by which the world regulates clocks and time.
* Time zone: For the purpose of this RFC, we define three representation of Time Zone:
    * UTC offset: Defined by ISO 8601, represents the difference between UTC time and local time.
    * Time Zone Abbreviations: Represented by alphabetic abbreviations such as "EST", "WST", and "CST"
    * Unknown Time Zone: Defined by RFC 3339. If the time in UTC is known, but the offset to local time is unknown, this can be represented with an offset of "-00:00"

### Out of Scope

* Time Zone Abbreviations is out of the scope for this RFC.
    * Many commercial database support evaluation of Time Zone Abbreviation by keeping a copy of [**IANA**](https://en.wikipedia.org/wiki/Internet_Assigned_Numbers_Authority) time zone data in the system table. But rarely would one store such information in persistent format.
        * Oracle is a noticeable exception, as it stores Time Zone Abbreviation in the original input.
    * SQL spec defines Time zone to be UTC offset, PartiQL will follow SQL spec on this matter for now, and left time zone abbreviation out of the scope in this RFC.
* Datetime arithmetic is out of scope for this RFC.
    * A complete set of datetime arithmetic requires `INTERVAL` type (representation of a span of time) to properly explain. As `INTERVAL` type is not yet defined in PartiQL, and it is out of scope for this RFC, the Datetime arithmetic will be covered in a follow-up RFC in the future.

### Datetime Data Type

PartiQL defines 5 datetime data types:
- DATE
- TIME WITH TIME ZONE
- TIME WITHOUT TIME ZONE
- TIMESTAMP WITH TIME ZONE
- TIMESTAMP WITHOUT TIME ZONE.

The data type `TIME WITHOUT TIME ZONE` and `TIME WITH TIME ZONE` are collective referred to as **TIME** types. The data type `TIMESTAMP WITHOUT TIME ZONE` and `TIMESTAMP WITH TIME ZONE` are collectively referred to as **TIMESTAMP** type. The data types `DATE`, `TIME`, and `TIMESTAMP` are collectively referred to as **DATETIME** types. Values of `DATETIME` types are referred to as **datetimes**.

Table1 “Fields in datetime values”, specifies the fields that can make up a datetime value; a datetime value is made up of a subset of those fields. 


| Keyword	         | Meaning	                                  |
|------------------|-------------------------------------------|
| YEAR	            | Proleptic Year	                           |
| MONTH	           | Month of Year	                            |
| DAY	             | Day of Month	                             |
| HOUR	            | Hour of Day	                              |
| MINUTE	          | Minute of Hour	                           |
| SECOND	          | Second and possibly fraction of a second	 |
| TIMEZONE_HOUR	   | Hour value of UTC offset	                 |
| TIMEZONE_MINUTE	 | Minute value of UTC offset	               |

Table1: Fields in datetime values

The `YEAR`, `MONTH`, `DAY`, `HOUR`, `MINUTE`, and `SECOND` are called **primary datetime field**, and is listed by the order of significant. The primary datetime fields other than SECOND contain non-negative integer values, constrained by the natural rules for dates using the Gregorian calendar. SECOND, contains non-negative decimal values. The number before the decimal point represents whole second, constrained by natural rules using Gregorian calendar. The number after the decimal point represents fractional second. PartiQL implementation **SHALL** permit arbitrary precision for fraction second.

A datetime value, of data type `TIME WITHOUT TIME ZONE` or `TIMESTAMP WITHOUT TIME ZONE`, MAY represent a local time, whereas a datetime value of data type `TIME WITH TIME ZONE` or `TIMESTAMP WITH TIME ZONE` MUST represent UTC.

On occasion, UTC is adjusted by the omission of a second or the insertion of a “[leap second](https://www.nist.gov/pml/time-and-frequency-division/leap-seconds-faqs)” in order to maintain synchronization with  time. Whether a PartiQL-implementation supports leap seconds, and the consequences of such support for date and interval arithmetic, is implementation-defined.

For conformance of SQL spec, PartiQL permits implicit conversion between a datetime value with time zone and a datetime value without time zone. See [Conversion Between Datetime Types](https://quip-amazon.com/x1qAAhtS8m36#temp:C:ZMf21e817dd22c9470c9579d39d2) for detail.

### Operational Semantics

#### Data Type

```
<datetime Type> ::= 
    DATE
  | TIME [ <left paren> <precision> <right paren> ] 
      [ <with or without time zone> ]  
  | TIMESTAMP [ <left paren> <precision> <right paren> ]
      [ <with or without time zone> ]  
      
<with or without time zone> ::=
    WITH TIME ZONE
  | WITHOUT TIME ZONE         

<precision> ::= <unsigned integer> 
```

Rules:

1. `<precision>` defines the number of decimal digits following the decimal point in the SECOND `<primary datetime field>`.
2. If `<precision>` is not specified in TIME type, then 0 is implicit.
3. If `<precision>` is not specified in TIMESTAMP type, then 6 is implicit.
4. If `<with or without time zone>` is not specified, then WITHOUT TIME ZONE is implicit.
    1. SQL-1999 - Section 6.1 - Syntax Rules - 30/ 31

>Item 2,3,4 is to confirm with SQL spec ( see section 2,3,4). However, doing so means PartiQL currently does not have the syntactical ability to declare a type with arbitrary precision. See Unresolved question.

5. If `DATE` is specified, then the data type contains the `<primary datetime field>`s YEAR, MONTH, and DAY.
6. If `TIME` is specified, then the data type contains the `<primary datetime field>`s HOUR, MINUTE, and SECOND.
7. If `TIMESTAMP` is specified, then the data type contains the `<primary datetime field>`s YEAR, MONTH, DAY, HOUR, MINUTE, and SECOND.
8. Table 2, Valid Values for datetime fields, specifies the constraints on the values of the date time fields. The values of `TIMEZONE_HOUR` and `TIMEZONE_MINUTE` shall either both be non-negative or both be non-positive.

| Keyword	         | Valid Values	                                                                                                                                                       |
|------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| YEAR	            | 0001 to 9999	                                                                                                                                                       |
| MONTH	           | 01 to 12	                                                                                                                                                           |
| DAY	             | With in the range 1 to 31, but further constrained by the value of MONTH and YEAR fields, according to the Gregorian Calendar.	                                      |
| HOUR	            | 00 to 23	                                                                                                                                                           |
| MINUTE	          | 00 to 59	                                                                                                                                                           |
| SECOND	          | 00 to 60 (excluded) if not supporting leap second;  00 to 62(excluded) if supporting leap second. Whether or not Leap Second is supported is up to implementation.	 |
| TIMEZONE_HOUR	   | -23 to 23	                                                                                                                                                          |
| TIMEZONE_MINUTE	 | -59 to 59	                                                                                                                                                          |

Table 2 : Valid Value for datetime fields.


| Name	                                | Description	                          | Min Value	                 | Max Value	                       |
|--------------------------------------|---------------------------------------|----------------------------|----------------------------------|
| Date	                                | date (no time of day; no time zone)	  | 0001-01-01	                | 9999-12-31	                      |
| time [(p)] [WITHOUT TIME ZONE]	      | time of day ( no date, no time zone)	 | 00:00:00	                  | 23:59:59.999...	                 |
| time [(p)] WITH TIME ZONE	           | time of day (no date, has time zone)	 | 00:00:00-23:59	            | 23:59:59.999...+23:59	           |
| timestamp [(p)] [WITHOUT TIME ZONE]	 | both date and time (no time zone).	   | 0001-01-01 00:00:00	       | 9999-12-31 23:59:59.999	         |
| timestamp [(p)] WITH TIME ZONE	      | both date and time (has time zone).	  | 0001-01-01 00:00:00-23:59	 | 9999-12-31 23:59:59.999 + 23:59	 |

Table 3 DateTime Types

#### Datetime Literal

Grammar:

```
<datetime literal> ::= 
 <date literal>
 | <time literal>
 | <timestamp literal>
 
<date literal> ::= DATE <date string>

<time literal> ::= TIME <time string> 
 
<timestamp literal> ::= TIMESTAMP <timestamp string> 
 
<date string> ::= <quote> <unquoted date string> <quote>
 
<time string> ::= <quote> <unquoted time string> <quote>
 
<timestamp string> ::= <quote> <unquoted timestamp string> <quote>

<unquoted date string> ::= <date value>

<unquoted time string> ::= <time value> [ <time zone interval> ]

<unquoted timestamp string> ::= 
 <unquoted date string> <datetime delimiter> <unquoted time string>

<date value> ::= <years value> <minus sign> <months value> <minus sign> <days value>

<time value> ::= <hours value> <colon> <minutes value> <colon> <seconds value>

<time zone interval> ::= 
 <sign> <hours value> <colon> <minutes value>
 | <zulu time zone>

<zulu time zone> ::= Z | z

<datetime delimiter> ::= <space> | T | t

<years value> ::= <datetime value>

<months value> ::= <datetime value>

<days value> ::= <datetime value>

<hours value> ::= <datetime value>

<minutes value> ::= <datetime value>

<seconds value> ::= <seconds integer value> [ <period> [ <seconds fraction> ] ]

<seconds integer value> ::= <unsigned integer>

<seconds fraction> ::= <unsigned integer>

<datetime value> ::= <unsigned integer>
```

>PartiQL extends the SQL date time string format to support RFC 3339 format. That is 1) PartiQL **SHALL** support use of `T` as date time delimiter and use of `Z` as Zulu time zone. 2)  PartiQL **SHALL** support Unknown Time Zone (-00:00).

Rules:

1. The declared type of `<date literal>` is a DATE.
2. The declared type of `<time literal>` that does not specify `<time zone interval>` is of type `TIME(p) WITHOUT TIME ZONE` where `p` is the number of digits in second fraction.
3. The declared type of `<time literal>` that specifies `<time zone interval>` is of type `TIME(p) WITH TIME ZONE` where `p` is the number of digits in second fraction.
4. The declared type of `<timestamp literal>` that does not specify `<time zone interval>` is of type `TIMESTAMP(p) WITHOUT TIME ZONE` where `p` is the number of digits in second fraction.
5. The declared type of `<timestamp literal>` that specifies `<time zone interval>` is of type `TIMESTAMP(p) WITH TIME ZONE` where `p` is the number of digits in second fraction.

>Doing so limited our ability to declare a time/timestamp value with arbitrary precision. See Unresolved Question for detail.

6. If `<time zone interval>` is not specified, then the effective `<time zone interval>` of the datetime data type is the current time zone displacement for the session.
7. Within a `<datetime literal>`, the <years value> shall contain four digits. The `<seconds integer value>` and other date components, except `<seconds fraction>`, shall each contain two digits.
8. Within the definition of a `<datetime literal>`, the `<datetime values>` are constrained by the natural rules for dates and times according to the Gregorian calendar.
9. If `<time zone interval>` is specified, then the time and timestamp values in `<time literal>` and `<timestamp literal>` represent a datetime in the specified time zone.
10. If `<time zone interval>` is specified as `-00:00`(unknown time zone), then the `<time literal>` and `<timestamp literal>` represent a date time in zulu timezone.

>RFC 3339: If the time in UTC is known, but the offset to local time is unknown, this can be represented with an offset of "-00:00".  This differs semantically from an offset of "Z" or "+00:00", which imply that UTC is the preferred reference point for the specified time.

11. If `<date value>` is specified, then it is interpreted as a date in the Gregorian calendar. if `<time value>` is specified, then it is interpreted as a time of day. Let DV be the value of the `<datetime literal>`, disregarding `<time zone interval>`.
12. Case:
    1. If `<time zone interval>` is specified, then let TZI be the value of the interval denoted by `<time zone interval>`. The value of the `<datetime literal>` is DV-TZI, with time zone displacement TZI.
    2.  otherwise, the value of the `<datetime literal>` is DV.
13. If `<time zone interval>` is specified, then a `<time literal>` or `<timestamp literal>` is interpreted as local time with specified time zone displacement. However, it is effectively converted to UTC while retaining the original time zone displacement.
14. If `<time zone interval>` is not specified, then no assumption is made about time zone displacement. However, should a time zone displacement be required during subsequent processing, the current default time zone displacement of the PartiQL-session will be applied at that time.

Examples:

```
// Date type, 
// year = 2023, month = 06, day = 01
// It is a snapshot of a Gregorian calendar.
DATE '2023-06-01' 

// TIME WITHOUT TIME ZONE type, 
// hour = 00, minute = 00, second = 00
// It is a snapshot of a clock. 
TIME '00:00:00' 

// TIME(3) WITHOUT TIME ZONE type
// hour = 00, minute = 00, second = 00.000
TIME(3) '00:00:00' 

// TIME(3) WITHOUT TIME ZONE type
// hour = 00, minute = 00, second = 00.000
TIME(3) '00:00:00.00000' 

// TIME WITH TIME ZONE type
// hour = 00, minute = 00, second = 00
// timezone_hour = 00, timezone_minute = 00
// It is 00:00:00 at Zulu Time Zone. 
TIME '00:00:00Z'
TIME '00:00:00+00:00'

// TIME WITH TIME ZONE type
// hour = 00, minute = 00, second = 00
// timezone_hour = -00, timezone_minute = -00
// Semantically this represents a time recorded from "somewhere",
// in which the time at Zulu time zone is 00:00:00
TIME '00:00:00-00:00'

// TIME WITH TIME ZONE type
// hour = 00, minute = 00, second = 00
// timezone_hour = -07, timezone_minute = -00
TIME '00:00:00-07:00'

// TIMESTAMP WITHOUT TIME ZONE type
// year = 2023, month = 06, day = 01,
// hour = 00, minute = 00, second = 00
// This is a snapshot of a calender and a clock. 
TIMESTAMP '2023-06-01 00:00:00'

// TIMESTAMP WITH TIME ZONE type
// year = 2023, month = 06, day = 01,
// hour = 00, minute = 00, second = 00
// timezone_hour = 00, timezone_minute = 00
// This represents a date time instance in Zulu time zone. 
TIMESTAMP '2023-06-01 00:00:00Z'

// TIMESTAMP WITH TIME ZONE type
// year = 2023, month = 05, day = 31,
// hour = 17, minute = 00, second = 00
// timezone_hour = -07, timezone_minute = -00
// The same instance as above, but with different Time Zone. 
TIMESTAMP '2023-05-31 17:00:00-07:00'

// Same instance, use of T as delimiter
TIMESTAMP '2023-05-31T17:00:00-07:00'
```

#### Conversion Between Datetime Types

| 	                                 | to DATE	                  | to TIME WITHOUT TIME ZONE	  | to Time WITH TIME ZONE	                       | to TIMESTAMP WITHOUT TIME ZONE	                               | to TIMESTAMP WITH TIME ZONE	                                               |
|-----------------------------------|---------------------------|-----------------------------|-----------------------------------------------|---------------------------------------------------------------|----------------------------------------------------------------------------|
| from DATE	                        | trivial	                  | not supported	              | not supported	                                | copy year, month, and day; set hour, minute, and second to 0	 | SV => TSw/oTZ => TSw/TZ	                                                   |
| from TIME WITHOUT TIME ZONE	      | not supported	            | trivial	                    | TV.UTC = SV - STZD (modulo 24); TV.TZ = STZD	 | copy date field from CURRENT_DATE and time fields from SV	    | SV => TSw/oTZ => TSw/TZ	                                                   |
| from TIME WITH TIME ZONE	         | not supported	            | SV.UTC + SV.TZ(modulo 24)	  | trivial	                                      | SV => TSw/TZ => TSwo/TZ	                                      | Copy date fields from CURRENT_DATE and time and time zone fields from SV.	 |
| from TIMESTAMP WITHOUT TIME ZONE	 | Copy date fields from SV	 | Copy time fields from SV	   | SV => TSw/TZ => TIMEw/TZ	                     | trivial	                                                      | TV.UTC = SV - STZD; TV.TZ = STZD;	                                         |
| from TIMESTAMP WITH TIME ZONE	    | SV => TSw/oTZ => DATE	    | SV => TSw/oTZ => TIMEw/oTZ	 | Copy time and time zone fields from SV	       | SV.UTC + SV.TZ	                                               | trivial	                                                                   |

Table 4: Datetime data type conversion; Table from SQL Spec.

> SV is the source value, TV is the target value, UTC is the UTC component of SV or TV (if and only if the source or target has time zone), TZ is the time zone displacement of SV or TV (if and only if the source or target has time zone), STZD is the session default time zone displacement, => means to cast from the type preceding the arrow to the type following the arrow. TIMEw/TZ is TIME WITH TIME ZONE, TIMEw/oTZ is TIME WITHOUT TIME ZONE, TSw/TZ is TIMESTAMP WITH TIME ZONE, and TSw/oTZ is TIMESTAMP WITHOUT TIME ZONE.

Examples:

```
// Illegal operation
// DATE -> TIME WITHOUT TIME ZONE
cast(DATE '2023-06-01' AS TIME) // Exception

// DATE -> TIME WITH TIME ZONE
cast(DATE '2023-06-01' AS TIME WITH TIME ZONE) // Exception

// TIME WITHOUT TIME ZONE -> DATE
cast(TIME '00:00:00' AS DATE) // Exception

// TIME WITH TIME ZONE -> DATE
cast(TIME '00:00:00+00:00' as DATE) // Exception

// session independent operation
cast(DATE '2023-06-01' AS TIMESTAMP) // TIMESTAMP '2023-06-01 00:00:00'

cast(TIMESTAMP '2023-06-01 00:00:00' AS DATE) // DATE '2023-06-01'

cast(TIMESTAMP '2023-06-01 00:00:00' AS TIME) // TIME '00:00:00'

// Session Dependent
// assume current_date is 2023-06-01 
// current time is 00:00:00
// current tiem zone is -07:00

// TIME WITHOUT TIME ZONE -> TIME WITH TIME ZONE
cast(TIME '00:00:00' AS TIME WITH TIME ZONE)
// TIME '00:00:00-07:00'

// TIME WITH TIME ZONE -> TIME WITHOUT TIME ZONE
cast(TIME '00:00:00+00:00' AS TIME)
// TIME '17:00:00'

// TIMESTAMP WITHOUT TIME ZONE -> TIMESTAMP WITH TIME ZONE
cast(TIMESTAMP '2023-06-01 00:00:00' AS TIMESTAMP WITH TIME ZONE)
// TIMESTAMP '2023-06-01 00:00:00-07:00'

// TIMESTAMP WITH TIME ZONE -> TIMESTAMP WITHOUT TIME ZONE
cast(TIMESTAMP '2023-06-01 00:00:00+00:00' AS TIMESTAMP) 
// TIMESTAMP '2023-05-31 17:00:00'

// TIME WITHOUT TIME ZONE -> TIMESTAMP WITHOUT TIME ZONE
cast(TIME '00:00:00' AS TIMESTAMP) 
// TIMESTAMP '2023-06-01 00:00:00'

// TIME WITHOUT TIME ZONE -> TIMESTAMP WITH TIME ZONE  
cast(TIME '00:00:00' AS TIMESTAMP WITH TIME ZONE)
// TIMESTAMP '2023-06-01 00:00:00-07:00'

// TIME WITH TIME ZONE -> TIMESTAMP WITH TIME ZONE
cast(TIME '00:00:00+00:00' AS TIMESTAMP WITH TIME ZONE)
// TIMESTAMP '2023-06-01 00:00:00+00:00'

// TIME WITH TIME ZONE -> TIMESTAMP WITHOUT TIME ZONE
cast(TIME '00:00:00+00:00' AS TIMESTAMP)
// TIMESTAMP '2023-05-31 17:00:00'

// DATE -> TIMESTAMP WITH TIME ZONE
cast(DATE '2023-06-01' AS TIMESTAMP WITH TIME ZONE)
// TIMESTAMP '2023-05-31 17:00:00'
```

#### EXTRACT function

Define function `EXTRACT` as
`EXTRACT(<datetime field> FROM <datetime>) → <exact numeric value>.`

If the `<datetime field>` is not `SECOND`, the return type shall be an exact numeric value with implementation defined precision and scale 0.
If the `<datetime field>` is SECOND, the return type shall be an exact numeric value with implementation defined precision and scale. The result shall preserve all digit in the `SECOND` field.

If `<datetime>` does not contains the requested `<datetime field>`, the function shall throw an exception in type checking mode, and return `MISSING` in permissive mode.

Examples:

```
EXTRACT(DAY FROM DATE '2023-06-01') // 1
EXTRACT(MINUTE FROM DATE '2023-06-01') // Error in type checkign mode, MISSING in permissive mdoe
EXTRACT(SECOND FROM TIME '00:00:00.000') // 0.000
```

#### Comparison between two datetimes

Two datetimes are comparable only if they have the same `<primary datetime field>`s.


>The following section is for explanatory purpose only and will be replaced using Interval Type once it is defined.

To explain the semantic of datetime comparison, we define a function `DATE_DIFF` as
`DATE_DIFF(<primary datetime field>, <datetime1>, <datetime2>) → <exact numeric value>.`
The result of `DATE_DIFF` function is according to the natural rules associated with date and times, and should be interpreted as **a span of time**.
`<datetime1>` and `<datetime2>` are required to have the same `<primary datetime fields>`, and needs to include the `<primary datetime field>` requested.
For example:

```
DATE_DIFF(DAY, DATE '2023-06-02', DATE '2023-06-01') // 1; The result should be interpreted as time span of 1 days
DATE_DIFF(DAY, TIME '00:00:00', TIME '00:00:00') // invalid
DATE_DIFF(SECOND, TIME '00:00:00', TIMESTAMP '2023-06-02 00:00:00') // invalid
DATE_DIFF(
    DAY, 
    TIMESTAMP '2023-06-02 00:00:00+00:00', 
    TIMESTAMP '2023-06-01 00:00:00+00:00'
) // 1; The result should be interpreted as time span of 1 days
DATE_DIFF(
    DAY, 
    TIMESTAMP '2023-06-02 00:00:00', // timestamp without time zone
    TIMESTAMP '2023-06-01 00:00:00+00:00'
) // With implicit conversion. 
DATE_DIFF( 
    HOUR, 
    TIMESTAMP '2023-06-01 12:00:00+00:00', 
    TIMESTAMP '2023-06-01 00:00:00+00:00'
) // 12; The result should be interpreted as time span of 12 hours
DATE_DIFF(
    HOUR, 
    TIMESTAMP '2023-06-02 00:00:00-07:00', // timestamp without time zone
    TIMESTAMP '2023-05-31 17:00:00-07:00'
) // the difference is 1 day 7 hour. so the result is 31. The result should be interpreted as a time span of 31 hours. 
```


The comparison of two datetimes is determined according to the result of `DATE_DIFF` function. Let X and Y be the two values to be compared and let H be the least significant `<primary datetime field>` of X and Y, including fractional seconds precision if the data type is time or timestamp.

1. X is equal to Y if and only if DATE_DIFF(H, X, Y) = 0
2. X is less than Y if and only if DATE_DIFF(H, X, Y) < 0

Example:

```
DATE '2023-06-01' = DATE '2023-06-01' // TRUE
// H -> DAY FIELD
// DATE_DIFF(DAY, DATE '2023-06-01', DATE '2023-06-01') = 0

TIME '00:00:00' = TIME '00:00:00.000' // TRUE
// H -> SECOND Field 
// DATE_DIFF(SECOND, TIME '00:00:00', TIME '00:00:00.000') = 0

// Time zone conversion
TIME '17:00:00-07:00' = TIME '00:00:00+00:00' // TRUE
// H -> SECOND Field 
// DATE_DIFF(SECOND, TIME '17:00:00-07:00', TIME '00:00:00+00:00')
// After converting to the same time zone
// DATE_DIFF(SECOND, TIME '00:00:00+00:00', TIME '00:00:00+00:00') = 0

TIMESTAMP '2023-06-01 00:00:00+00:00' = TIMESTAMP '2023-05-31 17:00:00-07:00' // TRUE
// H -> SECOND Field 
// DATE_DIFF(SECOND, 
//           TIMESTAMP '2023-06-01 00:00:00+00:00', 
//.          TIMESTAMP '2023-05-31 17:00:00-07:00') = 0

// implicit conversion
// assume session time zone is -07:00
TIMESTAMP '2023-06-01 00:00:00' = TIMESTAMP '2023-06-01 00:00:00-07:00' // TRUE
```

The same rule applies to the GROUP BY equivalence function `eqg` (PartiQL Spec section 11.1) and the `order-by less-than function` (PartiQL spec 12.2).

Example:
Consider the database

```
logs: [
    {'sensor':1, 'co':0.4, 'ts': TIMESTAMP '2023-07-01 04:05:06'},
    {'sensor':1, 'co':0.5, 'ts': TIMESTAMP '2023-07-01 04:05:07-07:00'},
    {'sensor':1, 'co':0.2, 'ts': TIMESTAMP '2023-07-01 04:05:06-07:00'},
]

-- GROUP BY 
-- assume session time zone is -07:00
SELECT VALUE 
     {'ts' : ts, 'co': (SELECT VALUE v.l.co FROM g as v)}
FROM logs as l 
GROUP BY l.ts as ts GROUP AS G

-- GROUP BY result
<< 
   // The ts may also be TIMESTAMP '2023-07-01 04:05:06'
   { 'ts' : TIMESTAMP '2023-07-01 04:05:06-07:00', co: 0.5 },
   { 'ts' : TIMESTAMP '2023-07-01 04:05:07-07:00', co: 
>> 

-- ORDER BY 
-- assume session time zone is -07:00
SELECT l.sensor, l.co, l.ts
FROM logs as l 
ORDER BY l.ts 

-- ORDER BY result
[
    // The position of the first two row is un-deterministic
    {'sensor':1, 'co':0.4, 'ts': TIMESTAMP '2023-07-01 04:05:06'},
    {'sensor':1, 'co':0.2, 'ts': TIMESTAMP '2023-07-01 04:05:06-07:00'},
    {'sensor':1, 'co':0.5, 'ts': TIMESTAMP '2023-07-01 04:05:07-07:00'}
]
```

#### Ion Serialization

PartiQL Value of DATE Type:.

```
$date::{
   year:  <year field>,
   month: <month field>,
   day:   <day field>
}
```

PartiQL Value of TIME WITHOUT TIME ZONE Type:

```
$time::{
   hour:   <hour field>,
   minute: <minute field>,
   second: <second field>
}
```

PartiQL Value of TIME WITH TIME ZONE Type:

```
-- If time zone is not unknown time zone
$time::{
   hour:   <hour field>,
   minute: <minute field>,
   second: <second field>, // in ion decimal
   offset: <timezone_hour field> * 60 + <timezone_minute field>
}

-- if time zone is unknown time zone
$time::{
   hour:   <hour field>,
   minute: <minute field>,
   second: <second field>, 
   offset: null.int
}
```

PartiQL Value of TIMESTAMP WITHOUT TIME ZONE Type:

```
$timestamp::{
   year:  <year field>,
   month: <month field>,
   day:   <day field>,
   hour:   <hour field>,
   minute: <minute field>,
   second: <second field>, // in ion decimal
}
```

PartiQL Value of TIMESTAMP WITH TIME ZONE Type get serialized to ION TIMESTAMP with fully extended date time fields.

For example:

```
date '2023-01-01' -> $date::{year:2023, month:1, day: 1}

time '00:00:00.00' -> $time::{hour:0, minute: 0, second: 0.00}
time '00:00:00+00:00' -> $time::{hour: 0, minute: 0, second: 0., offset: 0}
time '00:00:00-00:00' -> $time::{hour: 0, minute: 0, second: 0., offset: null.int}

timestamp '2023-01-01T00:00:00Z' -> 2023-01-01T00:00:00Z // not 2023T or 2023-01T
timestamp '2023-01-01T00:00:00-00:00' -> 2023-01-01T00:00:00-00:00
timestamp '2023-01-01T00:00:00' -> 
    $timestamp::{year:2023, month:1, day:1, hour:0, minute:0, second: 0.}

```

### Prior Art

* [SQL 1992](https://www.contrib.andrew.cmu.edu/~shadow/sql/sql1992.txt)
* [Ion Spec](https://amazon-ion.github.io/ion-docs/docs/spec.html)
* [RFC 3339](https://datatracker.ietf.org/doc/html/rfc3339)

### Unresolved Question

* The ability to syntactically declare an arbitrary precision TIME/TIMESTAMP type or TIME/TIMESTAMP value.

Without loss of generality, we use TIME as an example here.

To declare an arbitrary precision TIME type, one way is to have the users to type `TIME` without giving a precision.

```
-- DDL syntax for demostration purpose only
CREATE TABLE foo ( 
    arbitrary_time TIME 
)
```

Unfortunately, SQL spec specify a default value for precision in case of precision is omitted (In the case of time it is 0). It means the above way will introduce two different semantic meaning in SQL spec and PartiQL spec, breaking PartiQL’s promise of being SQL-compatible.

Similarly, one way to use the value constructor to create an arbitrary time value is

```
TIME '00:00:00.000'
```

But SQL spec suggests that the above expression create a TIME type whose precision is number of the digits after decimal point in SECOND field. (TIME(3) WITHOUT TIME ZONE).

We can resolve the issue with special parameter such as `TIME(+inf)` or additional constructor `ARBITRARY TIME`, but both of which seem somewhat unnatural.

### Future Possibility

* Define Interval Type in PartiQL.
* Define additional standard operation involved with datetime type as stated in SQL spec, such as overlaps, date time arithmetic.
* Support Time Zone ID. 

