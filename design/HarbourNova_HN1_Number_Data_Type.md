# HarbourNova HN1: Why the Number Data Type Matters

*Exact decimal arithmetic, PostgreSQL-aligned capacity, and consistent numeric behavior throughout the language.*

This article describes the approved behavior of `Number` in HN1. The language type is named **`Number`**; “numeric” describes its purpose and its relationship to database types such as PostgreSQL `numeric`.

A database can preserve a decimal value perfectly, yet an application can lose some of its digits as soon as it reads that value into an unsuitable numeric type. A calculation can look correct on screen while carrying a small approximation internally. A large total can absorb a small adjustment so completely that formatting the final result cannot recover it.

HN1 addresses these problems with a native exact decimal type. `Number` supports ordinary decimal values and very large values through the same language type, automatically selects an appropriate internal representation, and keeps decimal arithmetic in the decimal domain.

Its capacity is substantial: **up to 131,072 digits before the decimal point and 16,383 after it**. But the benefit starts with something as small as `0.1`.

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
local nFirst  Number := 0.1
local nSecond Number := 0.2
local nResult Number := nFirst + nSecond

// nResult is exactly 0.3.
```

The decimal values enter the expression exactly, and addition preserves their exact decimal result.

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

## Small values can take a faster internal path

Supporting large numbers does not require every calculation to use the most expensive representation.

HN1's design gives `Number` two complementary internal strategies. Eligible small values can use a compact representation based on an integer coefficient and a decimal scale or exponent. Larger values use variable-sized decimal storage and the multiprecision engine.

For example, the decimal value `123.45` can be understood conceptually as an integer coefficient of `12345` with a scale of `2`:

```text
12345 × 10^-2 = 123.45
```

Machine integers can store and manipulate decimal coefficients exactly.

When an operation qualifies for the compact path, HN can use checked native integer arithmetic and avoid overhead associated with the general large-number path. When it needs a larger representation, HN expands the value automatically through direct numeric conversion. Decimal text is not used as an intermediate arithmetic representation.

The public behavior stays the same across that transition. A value does not change its type, lose decimal digits, or acquire different equality rules because its internal representation grows.

This gives developers a useful combination: **lower overhead for eligible everyday values and much greater capacity when the calculation needs it**. The exact optimization depends on both the operands and the operation; it is not a promise that every small calculation uses the same path or that decimal arithmetic outperforms hardware floating point.

## Number is a native HN type

`Number` participates in normal language expressions and typed declarations:

```harbour
local nUnitPrice Number  := 19.95
local iQuantity  Integer := 3
local nSubtotal  Number  := nUnitPrice * iQuantity

// nSubtotal is exactly 59.85.
```

Application code uses the ordinary operators. It does not need a special string-arithmetic library or a different calling convention for every addition and multiplication.

The same type belongs in local variables, parameters, return values, object properties, collection elements, Tensor cells, Cursor columns, and Local SQL expressions. A function can accept a `Number`, calculate with it, return a `Number`, and place the result into a Cursor without changing the numeric domain.

Decimal-point and exponent literals are `Number` values. Thus `0.1`, `12.3400`, and `1e20` begin in the exact decimal domain. The compiler must preserve that meaning rather than first parsing the literal through a binary `double`.

HN also retains distinct integer types and an explicit `Float` type. Integers remain useful for counts and indexes. `Number` supplies exact decimal semantics. `Float` supplies IEEE binary64 semantics when an application intentionally needs that domain.

HN does not silently mix `Number` and `Float` arithmetic. Entering the approximate binary domain is an explicit choice, for example through `hn_Float(...)`.

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

`**` is HN's exponentiation operator. The remainder operator `%` follows truncation-toward-zero quotient semantics; the nonzero remainder has the dividend's sign.

### Division does not switch to Float

Some mathematical answers have no finite decimal expansion. For example:

```harbour
local nThird Number := 1.0 / 3.0
```

No finite decimal type can store an infinite sequence of threes. HN returns a rounded decimal `Number` according to its defined PostgreSQL-compatible division rules. It does not convert the operands to `Float`, perform approximate binary division, and convert that result back.

The distinction is essential: **rounding required by a finite decimal result is deliberate decimal behavior**. It is different from approximating a perfectly representable decimal input before the calculation even begins.

Storage capacity and an operation's result scale are also different. HN1's ordinary division rule calculates a result scale, capped at 1,000 fractional places; many divisions use far fewer. The 16,383-place capacity describes what a `Number` can retain, not a request to calculate every quotient to that many places. Even a terminating quotient can require rounding if it exceeds the selected result scale.

The example uses `1.0 / 3.0` deliberately. These are `Number` operands. HN's fixed-integer division has its own truncation rules.

### Decimal mathematics has defined rounding

The HN1 numeric design also includes decimal operations such as square roots, exponentials, logarithms, and powers through the math API. Their `Number` forms remain in the decimal domain. Results that cannot be represented exactly, such as the square root of two, require defined decimal rounding.

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

`Number(10,2)` permits eight integer digits and two fractional digits. Assignment applies the destination's decimal rounding rule, then checks whether the result fits.

By contrast:

```harbour
local nUnrestricted Number := 12.345

// nUnrestricted retains exactly 12.345.
```

The constrained destination is what requires rounding. Unconstrained `Number` does not arbitrarily reduce every value to two or four decimal places.

HN's PostgreSQL-aligned declaration model permits precision from `1` through `1000` and scale from `-1000` through `1000`. It supports negative scale for rounding to positions left of the decimal point, as well as scale greater than precision for appropriately constrained fractional values. `Number(p)` means scale zero.

These declaration limits are separate from the much larger capacity of an unconstrained `Number`. The distinction matters when mapping PostgreSQL schemas into HN: a column's declared business constraints and the decimal engine's overall capacity are different pieces of information.

### Decimal scale has meaning without changing numeric equality

The values `1.2`, `1.20`, and `1.2000` are mathematically equal, but their retained decimal scales can differ. HN preserves meaningful scale metadata while ordinary numeric equality and compatible hashing compare the values consistently.

That allows a value to retain its intended decimal presentation without treating `1.20` as a different numerical quantity from `1.2`. Lossless persistence and database mappings must preserve the relevant scale as well as the value.

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

HN's rounding design includes choices such as half away from zero and half even. PostgreSQL-compatible constrained assignment uses the appropriate decimal coercion rule; an explicit rounding operation lets an application select its required policy.

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

HN therefore has capacity to represent the ordinary finite decimal values from these database types without reducing them to a conventional binary double.

For example, a database value such as:

```text
1234567890123456789012345678.90
```

can remain an exact numeric value while HN compares it, calculates with it, and passes it through application functions. Preserving it does not require keeping it as an opaque string.

Correct transport still matters. A decimal-aware adapter must preserve the original decimal value and applicable column metadata. If a driver or foreign-language component converts the value to a binary float before HN receives it, a later conversion to `Number` cannot reconstruct the discarded digits.

Likewise, writing a result back must respect the destination column. A value that fits an unconstrained HN `Number` may exceed a particular `decimal(18,2)` column. The database's precision, scale, and rounding rules remain part of the contract.

PostgreSQL alignment also does not mean every HN language rule is identical to SQL behavior. Special-value comparisons, NULL handling, error reporting, and other language rules remain explicitly defined by HN. Matching capacity provides the foundation; matching the relevant expression and conversion semantics is what makes an interchange reliable.

## The same numeric model in Cursors and Local SQL

The benefit becomes much greater when `Number` is used throughout the language, including HN's Local SQL engine.

A Cursor is a local structured collection of rows with named, typed columns. When a column contains `Number` values, its decimal identity must remain intact while HN filters, calculates, compares, groups, and aggregates those values.

Consider a Cursor named `InvoiceLines`, where `UnitPrice` is `Number` and `Quantity` is an integer. The following query illustrates the calculation:

```sql
SELECT
    CustomerId,
    SUM(UnitPrice * Quantity) AS InvoiceTotal
FROM InvoiceLines
GROUP BY CustomerId;
```

`UnitPrice * Quantity` stays in the decimal domain. The aggregate returns a `Number`. The Local SQL implementation does not need a parallel floating-point interpretation of the same business calculation.

HN's numeric aggregate contracts include:

| Aggregate input | Result type |
| --- | --- |
| `SUM(Number)` | `Number` |
| `AVG(Number)` | `Number` |
| `SUM(BigInteger)` | `Number` |
| `AVG(Integer)` or `AVG(BigInteger)` | `Number` |

An average may require rounded decimal division. A sum over exact decimal inputs does not become a float merely because it is evaluated by SQL.

This supports a consistent flow: a backend decimal becomes a `Number` in a Cursor, HN code and Local SQL calculate with it, and a result Cursor retains the same decimal type. Multiple data sources can contribute decimal values to local processing without requiring the application to abandon their precision.

The practical benefit is easier reasoning about the whole calculation. Moving an expression between procedural HN code and Local SQL should not introduce an accidental numeric-domain conversion. Matching business rounding boundaries still matters in both places.

## Preserving accuracy at the application boundaries

`Number` works best when the application preserves decimal meaning from input to output.

Original decimal text and decimal-aware database values can enter the exact domain directly. An already approximate binary float is a different input: it contains the binary value that was actually stored, which may differ from the decimal value a person originally typed.

HN makes that distinction explicit. Converting a `Float` to `Number` preserves the actual binary64 value exactly; it does not silently reinterpret a friendly display string as the original number. A conversion can preserve information that exists, but it cannot infer information that has already been lost.

The same principle applies to external libraries and serialization formats. An interface that represents all numbers as binary doubles has a narrower numeric contract. An exact decimal path must be selected when decimal fidelity is required.

Within HN, the compact or large internal representation remains an implementation detail. Lossless persistence preserves the semantic decimal value and retained scale, so a reload does not depend on whichever internal optimization happened to be active when the value was saved.

## Why this is an important HN1 feature

It is easy to judge a numeric type by the largest number it can hold. For most applications, the more consequential questions are much smaller:

- Does `0.1` mean exactly one tenth?
- Can a small adjustment survive when added to a large total?
- Does a database decimal retain its digits in application memory?
- Do a function and a Local SQL query use the same decimal semantics?
- Does rounding occur at the boundary the application chose?
- Can an ordinary value grow without forcing a public type change?

HN1's `Number` brings those concerns into one native language type. Its automatic compact representation addresses common-case overhead, its large decimal engine supplies capacity, and its integration across variables, collections, Cursors, and Local SQL protects the calculation as it moves through an application.

**The result is a stronger foundation for software whose numbers must retain their decimal meaning from the first input to the final result.**

## References

The HN behavior described above follows the approved HN1 numeric design. The references below document the external database comparisons and floating-point background.

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
