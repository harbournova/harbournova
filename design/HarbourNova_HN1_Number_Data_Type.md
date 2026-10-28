# HarbourNova HN1: Why the Number Data Type Matters

*Exact decimal arithmetic, PostgreSQL-aligned capacity, and native support across variables, functions, collections, properties, and Globals.*

**Updated October 27, 2026.** The HN1 **Number core is implemented, validated, and merged into the main development branch**, including its integration with typed `GLOBAL` variables. This article describes that implementation and identifies the remaining integrations in their own sections.

The language type is named **`Number`**; “numeric” describes its purpose and its relationship to database types such as PostgreSQL `numeric`.

A database can preserve a decimal value perfectly, yet an application can lose some of its digits as soon as it reads that value into an unsuitable numeric type. A calculation can look correct on screen while carrying a small approximation internally. A large total can absorb a small adjustment so completely that formatting the final result cannot recover it.

HN1 addresses these problems with a native exact decimal type. `Number` supports ordinary decimal values and very large values through the same language type, uses variable-sized decimal storage, and keeps decimal arithmetic in the decimal domain.

Its capacity is substantial: **up to 131,072 digits before the decimal point and 16,383 after it**. But the benefit starts with something as small as `0.1`.

## What the completed Number core includes

The implementation covers the path from a decimal literal to an owned runtime value, through calculation, comparison, assignment, and formatting.

| Area | Implemented behavior |
| --- | --- |
| Literal values | Exact decimal-point and exponent literals, including ordinary typed and untyped expressions. |
| Arithmetic | Addition, subtraction, multiplication, decimal division, remainder, unary signs, integral powers, and supported compound and increment/decrement operators. |
| Typed contracts | `Number(p,s)` declarations, decimal coercion, checked range, and preservation of the previous destination after a rejected numeric store. |
| Language integration | Locals, statics, parameters, return values, references, class properties, Array elements, Hash values, and typed Globals. |
| Comparison and keys | Exact numeric equality and ordering, strict comparison, and compatible numeric Hash keys. |
| Conversions and output | Explicit numeric conversions, attempted conversions, and canonical decimal formatting. |

Decimal integration with Cursors and Local SQL, external database adapters, persistence, and advanced decimal math builds on this core. Those capabilities are described below as follow-on work; completion of the core does not imply their completion.

## A small example with a much larger implication

Consider:

```text
0.1 + 0.2
```

The mathematical result is exactly `0.3`. With ordinary IEEE binary64 arithmetic, the result is typically displayed at full round-trip precision as:

```text
0.30000000000000004
```

The reason is representation. Many finite decimal fractions, including `0.1`, have infinitely repeating binary expansions. A finite binary floating-point value must approximate them. A shorter display can hide that approximation, but it does not change the stored value. This is a general property of binary floating point, as explained in the [Python documentation on floating-point arithmetic][float-background].

HN decimal literals are exact `Number` values. Under HN1 semantics:

```harbour
local nFirst  Number := 0.10
local nSecond Number := 0.20
local nResult Number := nFirst + nSecond

// nResult is exactly three tenths.
// hn_ValueToText(nResult) produces "0.30".
```

The decimal values enter the expression exactly, addition preserves their exact decimal result, and canonical formatting retains the meaningful decimal scale.

This matters whenever a numeric result becomes a decision: whether an invoice balances, whether a quantity has reached a threshold, whether two calculated values compare equal, or whether an adjustment changes a total.

**Displaying the expected decimal value and calculating with that exact decimal value are separate things. HN's `Number` provides the latter.**

## The issue familiar to xBase developers

Legacy Harbour distinguishes integer values from floating-point values, but its ordinary fractional arithmetic uses C `double` operations. Its virtual machine source shows the integer and floating-point paths explicitly. See the [Harbour virtual machine implementation][harbour-vm].

Visual FoxPro developers will recognize the same underlying issue. Microsoft's [Numeric type documentation, maintained by the VFPX Help project][vfp-numeric], describes decimal-to-binary conversion and the resulting precision limitations. VFP also provides a [Currency type][vfp-currency], with four decimal places, for a more constrained decimal use case.

HN extends exact decimal handling to a general language type. Quantities, percentages, exchange rates, unit costs, measurements expressed as exact decimals, and database numeric values can all use `Number`. They do not have to fit a fixed four-decimal currency representation.

The improvement also goes beyond the familiar `0.1 + 0.2` example. Consider this calculation, evaluated in the stated order:

```text
(10000000000000000.00 + 0.01) - 10000000000000000.00
```

Exact decimal arithmetic produces `0.01`. Ordinary binary64 arithmetic rounds away the cent during the addition and produces `0.0` after the subtraction. Formatting that answer to two decimal places gives `0.00`; the lost cent is already gone.

The example is deliberately large to make the failure visible. The principle applies whenever a representation lacks enough precision for the values being combined. More attractive output cannot restore information discarded earlier in the calculation.

## PostgreSQL-scale capacity in a language variable

An unconstrained HN `Number` provides the following finite-value capacity:

| Part of the value | Supported capacity |
| --- | ---: |
| Digits before the decimal point | Up to 131,072 |
| Decimal places after the decimal point | Up to 16,383 |

These bounds align with PostgreSQL's unconstrained `numeric` capacity, documented in its [numeric types reference][pg-numeric].

This is variable-sized capacity. Declaring a `Number` does not reserve space for the maximum possible number of digits. An ordinary value such as `19.95` does not acquire a hundred-thousand-digit allocation simply because the type can represent much larger values.

Large capacity serves several practical purposes:

- A high-precision database decimal can remain numeric in application memory.
- Large totals can retain small decimal adjustments.
- Calculations can retain useful intermediate digits before applying a business rounding rule.
- The same public type handles everyday values and values that exceed machine integer or conventional fixed decimal capacities.

There is no separate public `BigNumber` that application code must select when a value grows. HN1 incorporates that capability into `Number` itself.

The limits are finite, and enormous calculations still consume memory and processing time. If a result exceeds an applicable range or resource limit, HN reports the failure. It does not silently substitute a floating-point approximation to make the operation succeed.

## Storage adapts to the value

Supporting very large numbers does not require reserving the maximum possible storage for every value.

The implemented engine stores significant decimal information in groups of four decimal digits, with separate sign, position, and retained-scale information. A value such as `19.95` needs only a small significant representation. A value with thousands of meaningful digits requires more storage and more work.

The representation is often described as base 10,000. These digit groups are held and manipulated using machine integers. Storing an exact decimal representation in binary computer memory does not turn it into binary floating-point arithmetic.

Retained scale is distinct from significant digit storage. Values such as `1.2`, `1.20`, and `1.2000` can preserve their different scales without treating every displayed trailing zero as an additional significant digit group.

The engine also has specialized paths for cases such as zero operands, exact cancellation, and division by a short divisor. These reduce work where the operation permits it. There is no need to convert values to text or binary `Float` to use these paths.

This gives ordinary values much smaller representations than the maximum-capacity case while retaining one public type. Applications do not select a “large number mode” or rewrite their calculations when more digits are needed.

An additional inline small-coefficient representation remains a planned optimization. It is separate from the variable-sized storage and specialized operations already implemented. There is no published fixed small-value threshold or benchmark multiplier to claim for that future representation.

The important guarantee for application code is stable: storage choices must preserve the same decimal value, scale, and numeric behavior.

## Number is a native HN type

`Number` participates in normal language expressions and typed declarations:

```harbour
local nUnitPrice Number  := 19.95
local iQuantity  Integer := 3
local nSubtotal  Number  := nUnitPrice * iQuantity

// nSubtotal is exactly 59.85.
```

Application code uses the ordinary operators. It does not need a special string-arithmetic library or a different calling convention for every addition and multiplication.

The same type works in local and static variables, parameters, return values, reference parameters, class properties, Array elements, Hash values, and Globals. Passing a value through those supported locations preserves its decimal identity.

For example, a normal function can accept and return exact decimal values:

```harbour
function AddExact(first Number, second Number)
   return first + second
```

Native Number values also work in ordinary collections:

```harbour
local aAmounts := { 19.95, 10.05 }

aAmounts[1] += 0.10

// aAmounts[1] is exactly 20.05.
```

The implementation includes supported compound assignments, prefix and postfix updates, and numeric loop handling. These paths retain numeric types and checked assignment behavior; they do not fall back to approximate arithmetic merely because a value is accessed through a collection or property.

Decimal-point and exponent literals are `Number` values. Thus `0.1`, `12.3400`, and `1e20` begin in the exact decimal domain. The compiler preserves their original decimal meaning without first parsing them through a binary `double`.

This also applies to ordinary untyped declarations such as `local nRate := 0.0825`. An explicit `Number` annotation is useful for a contract, but is not required merely to make a decimal literal exact.

HN also retains distinct integer types and an explicit `Float` type. Integers remain useful for counts and indexes. `Number` supplies exact decimal semantics. `Float` supplies IEEE binary64 semantics when an application intentionally needs that domain.

HN does not silently mix `Number` and `Float` arithmetic. Entering the approximate binary domain is an explicit choice, for example through `hn_Float(...)`.

### Number in Global variables

HN supports typed Globals declared outside routines:

```harbour
GLOBAL SalesTaxRate Number(7,5) := 0.08250

function CalculateSalesTax(nSubtotal Number)
   local nTax Number(12,2) := nSubtotal * SalesTaxRate
   return nTax
```

`SalesTaxRate` can be defined in one source file and used by name from other source files in the same application. Its precision and scale remain part of its declared contract. Assignments are checked and coerced using the same Number rules as local variables.

This is useful for application-wide rates, thresholds, limits, and amounts. It is a real declared binding: the compiler resolves uses of the name to the appropriate application-owned variable. Independent libraries retain their own ownership scopes.

Individual Global reads and assignments are coherent. A compound update such as `SharedTotal += amount` still consists of a read, a calculation, and a write; code that permits concurrent updates needs appropriate synchronization. Declaring a Number Global does not make a sequence of updates automatically atomic.

## Arithmetic stays decimal

For representable finite results, addition, subtraction, and multiplication preserve exact decimal values. Comparisons also operate on numeric value without first reducing it to a binary float.

Examples of `Number` arithmetic include:

| Expression | Result |
| --- | --- |
| `0.1 + 0.2` | Exactly `0.3` |
| `19.95 * 3` | Exactly `59.85` |
| `0.1 * 3 - 0.3` | Exactly `0` |
| `5.5 % 2.0` | Exactly `1.5` |
| `2.5 ** 3` | Exactly `15.625` |
| `1.0 / 8.0` | Exactly `0.125` |

The result column shows mathematical values. Canonical output can retain additional trailing decimal places, such as `0.30`.

`**` is HN's exponentiation operator. The implemented Number operation accepts integral exponents, including integral-valued Number exponents, and uses decimal division for negative powers. Fractional exponents belong to the separate advanced-math work.

The remainder operator `%` follows truncation-toward-zero quotient semantics; the nonzero remainder has the dividend's sign.

### Division does not switch to Float

Some mathematical answers have no finite decimal expansion. For example:

```harbour
local nThird Number := 1.0 / 3.0
```

No finite decimal type can store an infinite sequence of threes. HN returns a rounded decimal `Number` according to its defined PostgreSQL-compatible division rules. It does not convert the operands to `Float`, perform approximate binary division, and convert that result back.

The distinction is essential: **rounding required by a finite decimal result is deliberate decimal behavior**. It is different from approximating a perfectly representable decimal input before the calculation even begins.

Storage capacity and an operation's result scale are also different. HN1's ordinary division rule calculates a result scale, capped at 1,000 fractional places; many divisions use far fewer. The 16,383-place capacity describes what a `Number` can retain, not a request to calculate every quotient to that many places. Even a terminating quotient can require rounding if it exceeds the selected result scale.

The example uses `1.0 / 3.0` to make the Number operands explicit. Fixed-integer division truncates toward zero when evaluated in the integer domain; a typed Number context can also establish decimal arithmetic for literal expressions.

### Further decimal mathematics

Decimal square roots, exponentials, logarithms, and nonintegral powers are planned math APIs. They are not part of the completed Number-core delivery. Their intended Number forms remain in the decimal domain, with defined rounding for results that cannot be represented exactly, such as the square root of two.

The precise promise is that HN preserves the chosen numeric domain and follows its documented rules. It is not a claim that every mathematical answer has a finite exact representation.

## Precision and scale can express a business contract

An unconstrained `Number` is useful when a calculation needs to retain its natural decimal precision. A declaration such as `Number(p,s)` adds an explicit precision and scale contract.

For the common case of a nonnegative scale no greater than precision:

- **Precision** is the total permitted number of decimal digits.
- **Scale** is the number of places after the decimal point.

For example:

```harbour
local nAmount Number(10,2)
nAmount := 12.345

// nAmount becomes 12.35.
```

`Number(10,2)` permits eight integer digits and two fractional digits. Assignment rounds halfway cases away from zero to the destination scale, then checks whether the result fits.

By contrast:

```harbour
local nUnrestricted Number := 12.345

// nUnrestricted retains exactly 12.345.
```

The constrained destination is what requires rounding. Unconstrained `Number` does not arbitrarily reduce every value to two or four decimal places.

HN's PostgreSQL-aligned declaration model permits precision from `1` through `1000` and scale from `-1000` through `1000`. It supports negative scale for rounding to positions left of the decimal point, as well as scale greater than precision for appropriately constrained fractional values. `Number(p)` means scale zero.

These declaration limits are separate from the much larger capacity of an unconstrained `Number`. The distinction matters when mapping PostgreSQL schemas into HN: a column's declared business constraints and the decimal engine's overall capacity are different pieces of information.

### Checked assignment preserves the previous value

A rejected numeric store leaves the previous destination intact:

```harbour
local nAmount Number(5,2) := 12.34

try
   nAmount := 1000.00
catch error HNError
   // The new value does not fit Number(5,2).
   // nAmount still contains exactly 12.34.
endtry
```

The same discipline matters for compound assignment. HN evaluates the complete arithmetic expression before applying the final destination contract:

```harbour
local nAmount Number(5,2) := 1.00

nAmount += 0.004 + 0.004

// The exact candidate is 1.008.
// One final constrained store produces 1.01.
```

The two small additions are not independently rounded to zero before they can contribute to the result. Range and resource failures use HN's structured error handling rather than changing the result to another numeric domain.

## Equality, retained scale, and numeric keys

The values `1.2`, `1.20`, and `1.2000` are mathematically equal, but their retained decimal scales can differ. HN preserves meaningful scale metadata while ordinary numeric equality compares the values.

Strict comparison also considers the numeric type and its significant attributes. For Number, retained scale is one of those attributes:

| Comparison | Result |
| --- | --- |
| `1.20 == 1.200` | True: the numerical values are equal. |
| `1.20 === 1.200` | False: the retained scales differ. |

The comparison does not depend on an allocation address or a private storage optimization.

HN also implements exact mixed numeric comparison. It can compare a Number with an integer or Float according to their actual represented values, without first rounding the Number to a Float:

```harbour
hn_Number(0.5) == hn_Float(0.5)   // True.
hn_Number(0.1) == hn_Float(0.1)   // False.
```

One half is exactly representable in binary64; one tenth is not. This comparison capability is separate from arithmetic: mixed Number/Float calculations still require an explicit choice of domain.

For finite values, numeric Hash keys follow compatible numeric equality. Integer `1`, Number `1.00`, and a Float representing exactly `1` identify the same numeric key. Exact decimal `0.1` and the nearest binary64 value to `0.1` remain distinct because their values differ.

This keeps lookup predictable when finite keys originate in different numeric types. Hash keys use a separate equivalence rule for NaN: all NaNs share one key even though ordinary NaN equality is false.

## Numeric conversions and exact output

The implemented public conversion family makes the numeric boundary explicit:

| API | Purpose |
| --- | --- |
| `hn_Number(xValue)` | Convert a present fixed integer, Number, or Float to Number. |
| `hn_Float(xValue)` | Explicitly convert a supported numeric value to binary64. |
| `hn_ConvertToNumber(...)`, `hn_ConvertToFloat(...)` | Attempt a numeric conversion, returning a Boolean and writing through the result reference. |
| `hn_ConvertToSmallInteger(...)`, `hn_ConvertToInteger(...)`, `hn_ConvertToBigInteger(...)` | Attempt an exact checked conversion to the selected bounded integer type. |
| `hn_ValueToText(number)` | Produce canonical decimal text that preserves the Number value and retained scale. |

**The current `hn_Number()` constructor accepts numeric values. It is not a general decimal-text parser.** For example, `hn_Number("19.95")` is not a supported Text conversion. The attempted numeric conversions also require numeric input. A public text-parsing API is a separate boundary.

This does not affect source literals: writing `19.95` directly in HN source already creates an exact Number.

### Converting an existing Float

`hn_Number(Float)` preserves the actual binary64 value exactly. It does not reinterpret the Float's friendly display text as the number a person originally intended.

For example:

```harbour
local fApproximate Float := hn_Float(0.1)
local nImported Number := hn_Number(fApproximate)
```

The exact decimal value imported into `nImported` is:

```text
0.1000000000000000055511151231257827021181583404541015625
```

The conversion is faithful: it preserves the value that was actually in the Float. It cannot recover an original decimal value that was discarded during an earlier approximate conversion.

Conversely, `hn_Float(Number)` explicitly rounds the exact value into binary64 using nearest-even rounding. Applications choose that boundary deliberately.

### Attempting a conversion

The attempted conversions use this common shape:

```text
hn_ConvertTo<Type>(xValue, @xResult [, @errError]) -> Boolean
```

For an ordinary representability failure, they return false and preserve the result variable:

```harbour
local iResult Integer := 7
local lConverted Boolean := hn_ConvertToInteger(12.5, @iResult)

// lConverted is false: 12.5 is not an exact Integer.
// iResult remains 7.
```

An optional error result can describe that failure. Invalid argument contracts and resource or allocation failures still use canonical HNError handling; these functions are not a way to suppress every possible error.

Canonical `hn_ValueToText(Number)` output preserves the exact decimal value and scale using a decimal point, without localized grouping or padding. For example, the exact sum of `0.10` and `0.20` formats as `0.30`. Number also supports the special values `NaN`, `Infinity`, and `-Infinity`, with HN-defined comparison and arithmetic rules.

## Exact arithmetic still needs a rounding policy

Decimal accuracy does not decide an application's business rules. Developers still choose whether to round each line, each tax calculation, or only the final total.

For example, suppose an application intentionally retains all decimal intermediates and rounds the final invoice amount to two places:

| Step | Value |
| --- | ---: |
| Unit price | `19.95` |
| Quantity | `3` |
| Subtotal | `59.85` |
| After a 10% discount | `53.8650` |
| Tax at 8.25% | `4.44386250` |
| Total before final rounding | `58.30886250` |
| Final total, rounded to two places | `58.31` |

All the intermediates in this example have finite exact decimal representations. HN can retain them until the selected rounding boundary.

An application that rounds the discounted line before calculating tax is applying a different policy and may obtain a different result. `Number` makes that policy explicit and reproducible by removing unintended binary representation error from the calculation.

The implemented constrained assignment uses PostgreSQL-compatible rounding, with halfway cases rounded away from zero. Public APIs for explicitly choosing among rounding modes, including half even, remain planned functionality. The current core already lets an application choose where to impose a declared scale.

## Database parity that matters in application code

HN's `Number` capacity is deliberately aligned with PostgreSQL, and its decimal kernel draws on PostgreSQL's numeric implementation. The HN integration owns its allocation, errors, value representation, and language contracts; using `Number` does not require a running PostgreSQL server.

Other major databases also provide exact decimal types:

| System and type | Decimal capacity | Important qualification |
| --- | --- | --- |
| **HN1 `Number`** | Up to **131,072 integer digits** and **16,383 fractional places** | Unconstrained capacity; declarations and operations impose their own applicable rules. |
| [PostgreSQL 18 `numeric` / `decimal`][pg-numeric] | Up to **131,072 integer digits** and **16,383 fractional places** | Explicitly declared precision is limited to 1,000. |
| [SQL Server `decimal` / `numeric`][sqlserver-numeric] | Up to **38 total digits** | Fractional digits are part of the total precision. |
| [MySQL 8.4 `DECIMAL`][mysql-decimal] | Up to **65 total digits**, including up to **30 fractional digits** | Scale cannot exceed total precision. |
| [Oracle `NUMBER(p,s)`][oracle-number] | Up to **38 significant digits of declared precision** | Scale has a separate range, including negative values; this is not a 38-integer-plus-38-fractional limit. |

HN therefore has capacity to represent the ordinary finite decimal values from these database types without reducing them to a conventional binary double. The completed core supplies that representation and its arithmetic; provider-specific decimal adapters and their end-to-end round-trip qualification remain follow-on work.

For example, a database value such as:

```text
1234567890123456789012345678.90
```

can remain an exact numeric value while HN compares it, calculates with it, and passes it through application functions. Preserving it does not require keeping it as an opaque string.

Correct transport still matters. A decimal-aware adapter must preserve the original decimal value and applicable column metadata. If a driver or foreign-language component converts the value to a binary float before HN receives it, a later conversion to `Number` cannot reconstruct the discarded digits.

Likewise, writing a result back must respect the destination column. A value that fits an unconstrained HN `Number` may exceed a particular `decimal(18,2)` column. The database's precision, scale, and rounding rules remain part of the contract.

PostgreSQL alignment also does not mean every HN language rule is identical to SQL behavior. Special-value comparisons, NULL handling, error reporting, and other language rules remain explicitly defined by HN. Matching capacity provides the foundation; matching the relevant expression and conversion semantics is what makes an interchange reliable.

## Next integration: Number in Cursors and Local SQL

HN already has a Local SQL engine. Connecting the completed exact Number core to decimal Cursor transport, SQL expressions, and decimal aggregates is a separate integration step.

A Cursor is a local structured collection of rows with named, typed columns. The intended Number integration preserves a decimal column's identity while HN filters, calculates, compares, groups, and aggregates its values.

For example, consider a Cursor named `InvoiceLines`, where `UnitPrice` is Number and `Quantity` is an integer. This query illustrates the intended decimal integration:

```sql
SELECT
    CustomerId,
    SUM(UnitPrice * Quantity) AS InvoiceTotal
FROM InvoiceLines
GROUP BY CustomerId;
```

The required decimal behavior is for `UnitPrice * Quantity` to remain Number and for the aggregate to return Number.

The approved aggregate directions include:

| Aggregate input | Intended result type |
| --- | --- |
| `SUM(Number)` | `Number` |
| `AVG(Number)` | `Number` |
| `SUM(BigInteger)` | `Number` |
| `AVG(Integer)` or `AVG(BigInteger)` | `Number` |

An average may require rounded decimal division; an exact decimal sum must not become a Float merely because SQL evaluates it.

The intended flow is valuable: a backend decimal enters a Cursor as Number, HN code and Local SQL calculate with it, and a result Cursor retains the same decimal type. Matching business rounding boundaries remains necessary.

The implemented Number core now provides the arithmetic and value semantics for that work. Completing the adapters and query paths will carry those guarantees through the full database-to-application workflow.

## Remaining integrations and planned extensions

The core's value model is the foundation for several additional capabilities:

| Area | Direction beyond the completed Number core |
| --- | --- |
| Cursor and Local SQL decimals | Native decimal transport, expression handling, result columns, and the appropriate aggregate contracts. |
| Database and DataWharf adapters | Preserve exact decimal values and applicable precision/scale metadata on input and output. |
| Persistence and reload | Preserve the semantic decimal value, retained scale, and special-value identity independently of private runtime storage. |
| Tensor cells | Carry native Number values through the Tensor implementation and its supported storage paths. |
| Advanced decimal math | Decimal square roots, exponentials, logarithms, and nonintegral powers with defined result precision and rounding. |
| Selectable rounding APIs | Explicit application selection among the approved rounding modes. |
| Public decimal-text parsing | A separate parsing interface; current numeric constructors remain numeric-only. |
| Additional compact storage | A possible inline small-coefficient path that preserves the existing public Number semantics. |

The boundary principle is the same in each case: preserve decimal meaning from the original input to the final output. An external component that reduces all numbers to binary doubles has already imposed a narrower contract. Converting its result back to Number cannot reconstruct discarded information.

For lossless persistence, a saved Number must describe its semantic value and scale. Reloading it should not depend on an allocation layout, an address, or whichever internal optimization was active when it was saved.

## Why this is an important HN1 feature

It is easy to judge a numeric type by the largest number it can hold. For most applications, the more consequential questions are much smaller:

- Does `0.1` mean exactly one tenth?
- Can a small adjustment survive when added to a large total?
- Does a function, property, collection, or Global retain the value's decimal identity?
- Does rounding occur at the boundary the application chose?
- Can a rejected assignment leave the previous valid value intact?
- Do numeric comparison and Hash lookup agree?
- Can the same type represent both an everyday price and a high-precision database amount?

The completed Number core addresses these concerns through the language's normal expressions and storage locations. Its capacity makes large values possible, its exact decimal arithmetic makes everyday calculations dependable, and its typed contracts make rounding and representability visible.

The remaining Cursor, SQL, adapter, and persistence work extends that foundation across more of the application. The core already supplies the decimal values and operations those integrations need.

**Number makes decimal accuracy a property of ordinary HN code—from the literal you write to the value you calculate, store, compare, and display.**

## References

The implementation status above reflects the Number core integrated on October 27, 2026. Planned integrations are identified separately. The references below document the external database comparisons and floating-point background.

1. [PostgreSQL 18: Numeric Types][pg-numeric]
2. [Microsoft SQL Server: decimal and numeric][sqlserver-numeric]
3. [MySQL 8.4: DECIMAL Data Type Characteristics][mysql-decimal]
4. [Oracle Database: Data Types][oracle-number]
5. [Harbour: Virtual Machine Source][harbour-vm]
6. [Microsoft Visual FoxPro 9 SP2 Help, VFPX Edition: Numeric Data Type][vfp-numeric]
7. [Microsoft Visual FoxPro 9 SP2 Help, VFPX Edition: Currency Data Type][vfp-currency]
8. [Python: Floating-Point Arithmetic — Issues and Limitations][float-background]

[pg-numeric]: https://www.postgresql.org/docs/18/datatype-numeric.html
[sqlserver-numeric]: https://learn.microsoft.com/en-us/sql/t-sql/data-types/decimal-and-numeric-transact-sql?view=sql-server-ver17
[mysql-decimal]: https://dev.mysql.com/doc/refman/8.4/en/precision-math-decimal-characteristics.html
[oracle-number]: https://docs.oracle.com/en/database/oracle/oracle-database/26/sqlrf/Data-Types.html
[harbour-vm]: https://github.com/harbour/core/blob/master/src/vm/hvm.c
[vfp-numeric]: https://vfphelp.com/vfp9/html/f0945c58-08e4-46fc-a15b-e1714a064d91.htm
[vfp-currency]: https://vfphelp.com/vfp9/html/caed22ae-e7e5-4c6b-8e07-a7c5599aaa20.htm
[float-background]: https://docs.python.org/3/tutorial/floatingpoint.html
