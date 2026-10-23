# Local SQL in HarbourNova: bringing SQL to application data

Database servers are very good at selecting, joining, and summarizing the data they manage. Applications often need another step: combine results from several queries, add information from another system, apply application logic, and produce the dataset required by a report, screen, export, or web response.

HarbourNova (HN) brings SQL into that part of the application through **Local SQL over native Cursors**.

A backend query can produce a Cursor. Application code can also create and populate one. Local SQL then treats those Cursors as named sources, applies relational operations, and produces another result that HN code can use immediately. Jobs and Tasks extend this workflow by allowing independent backend requests to run concurrently before their results are combined locally.

The value comes from the combination: SQL, typed local data, ordinary HN code, and concurrent work all participate in one programming model.

## What “Local SQL” means

Local SQL executes inside the HN application against data that the application already holds. Its sources are explicitly supplied Cursors or WorkAreas, and the query uses familiar SQL concepts such as `SELECT`, `FROM`, `WHERE`, joins, grouping, and ordering.

The reference dialect is PostgreSQL, with a defined supported subset and selected HN extensions. Two useful extensions are optional `TYPE` declarations on output columns and the ability to reference a selected column's alias in `HAVING`.

SQL is supplied as HN Text. It can be a fixed query, Text assembled by application code, or the output of a query builder. The executor receives that Text together with named data sources, typed parameter bindings, and a destination for the result.

For example, a query can use `Customers` and `Orders` as source names even when those datasets came from different servers. Once materialized as Cursors, their original locations are separate from the local relational operation.

This gives an application a useful processing layer between backend retrieval and presentation. The server performs the work appropriate to its own data; HN performs the work that requires the application's combined local view.

## What is a Cursor?

An HN Cursor is an application-owned, typed table of data. It contains a schema and records, with named fields and declared types. In SQL terminology, records are rows and fields are columns.

A Cursor carries more information than a list of loosely described values. Its schema identifies the fields, their types, their NULL rules, and applicable value properties. The data engine also provides the record identity and access structures needed for navigation and indexing.

A Cursor can represent:

- The materialized result of a backend SQL query.
- Data assembled by application code.
- Data imported through an adapter from a file, service, or another source.
- The result of an earlier Local SQL query.

The term has a specific meaning here. A database-server cursor often represents a mechanism for fetching rows from an active server query. An HN Cursor holds the materialized local dataset. Once retrieval is complete, local processing can continue independently of the original fetch operation and its live connection.

### Cursor data and WorkArea navigation

HN separates the data from a particular navigation position:

| Concept | Responsibility |
| --- | --- |
| **Cursor** | The typed dataset: schema, records, field values, NULL states, and associated data structures. |
| **WorkArea** | A navigation view over a Cursor, with its own current record position and active order. |
| **Alias** | A name associated with a WorkArea in its execution context. |

Separate WorkAreas can refer to the same Cursor while navigating independently. Moving one view to another record does not move the other view. Two references to the same WorkArea, however, still refer to the same navigation state.

Local SQL reads its sources independently of their current WorkArea record pointers. A report query therefore does not have to inherit whichever record a screen or another routine last visited.

A materialized SQL result also has its own lifetime. Closing the input views does not invalidate a result that the application still retains.

## What Local SQL can do

The query surface covers a substantial range of everyday relational work:

| Operation | Examples |
| --- | --- |
| Select and shape columns | Field selection, expressions, `AS` aliases, `*`, and `alias.*`. |
| Filter records | Comparisons, `AND`, `OR`, `NOT`, `IS NULL`, and `IS NOT NULL`. |
| Express common predicates | `IN` lists, `BETWEEN`, `LIKE`, `ESCAPE`, and their corresponding `NOT` forms. |
| Combine related datasets | `INNER`, `LEFT`, `RIGHT`, `FULL`, and `CROSS JOIN`. |
| Group and inspect data | `GROUP BY`, `COUNT`, `MIN`, `MAX`, and `HAVING`. |
| Remove duplicates | `SELECT DISTINCT`. |
| Combine compatible query results | `UNION` and `UNION ALL`. |
| Order and page results | Multiple sort keys, `ASC`, `DESC`, `NULLS FIRST`, `NULLS LAST`, `LIMIT`, and `OFFSET`. |
| Define an output contract | Optional `TYPE` annotations on selected expressions. |
| Inspect a query's output schema | `hn_SQLDescribe()` without executing its row expressions. |

For example, given local `Customers` and `Orders` Cursors:

```sql
SELECT c.customer_id AS CustomerId,
       c.customer_name AS CustomerName,
       COUNT(o.order_id) AS OrderCount
FROM Customers AS c
LEFT JOIN Orders AS o
    ON o.customer_id = c.customer_id
GROUP BY CustomerId, CustomerName
ORDER BY OrderCount DESC, CustomerId;
```

This produces one row per customer, including customers with no matching orders. Assuming `order_id` is non-NULL on actual orders, `COUNT(o.order_id)` reports zero for an unmatched customer. `COUNT(*)` would count the NULL-extended join row instead, which is a useful example of why normal SQL semantics matter locally too.

The application can add:

```sql
HAVING OrderCount > 5
```

before `ORDER BY` to keep only customers with more than five orders. HN makes the selected alias `OrderCount` available in `HAVING`.

The same approach applies to reconciliation, duplicate detection, exception lists, reporting, and combining current and archived datasets. A developer expresses the relationship and desired result directly, instead of maintaining several nested loops and temporary lookup structures by hand.

## Passing application values into a query

HN supplies query values through named, typed bindings. A placeholder such as `:customerId` refers to a supplied value; it is not a request to splice that value into the SQL Text.

Assuming `curAllOrders` already refers to a Cursor, a typical call looks like this:

```harbour
cSQL := "SELECT o.order_id, o.customer_id " + ;
        "FROM orders AS o " + ;
        "WHERE o.customer_id = :customerId"

aSources := { { "orders", curAllOrders } }

aBindings := { ;
    { "Name" => "customerId", ;
      "Value" => iCustomerId, ;
      "Type" => "Integer" } ;
}

curCustomerOrders := NULL
nRows := hn_SQLSelect(cSQL, aSources, @curCustomerOrders, aBindings)

IF nRows < 0
    errSQL := hn_SQLLastError()
    // Application-specific error handling uses errSQL.
ENDIF
```

`hn_SQLSelect()` returns the number of result rows on success. It publishes the result through the destination reference. A recoverable query failure returns `-1`, with the structured HN error available through `hn_SQLLastError()`.

Bindings preserve their declared types even when their values are NULL. Text values remain data when they contain quotes or other characters that would have special meaning inside SQL syntax. Scalar bindings are captured once for the query, so changing the original binding collection during execution does not change the values already captured.

Explicit source registration is equally useful. In this example, `orders` means the Cursor supplied in `aSources`. The query does not depend on an unrelated routine having selected the expected ambient alias.

## HN logic can participate in SQL expressions

Local SQL also integrates with HN function and supported method calls. This allows application-specific logic to participate in query expressions while the relational parts remain expressed as SQL.

For example, an application that defines `CustomerLabel()` can use it to construct a display value:

```sql
SELECT c.customer_id AS CustomerId,
       CustomerLabel(c.customer_code, c.customer_name)
           AS Label TYPE Text(MaxLength = 160)
FROM Customers AS c;
```

The function belongs to the application. The query specifies where its result belongs in the output, and the `TYPE` annotation specifies the output contract.

This is especially useful when formatting, classification, or validation logic already exists in HN. The application can reuse that logic rather than maintaining another implementation in a separate expression language.

For predictable query behavior, such functions should have clear input and output contracts. Functions that perform network requests or change application state also bring those costs and side effects into query execution. Pure calculations and transformations are easier to compose, reason about, and reuse.

## Optional `TYPE` properties on output columns

One of HN's useful Local SQL extensions is the ability to describe a selected column more precisely than its name alone.

An annotation follows an explicit output alias:

```sql
SELECT c.customer_id AS CustomerId TYPE Integer NOT NULL,
       c.customer_code AS CustomerCode
           TYPE Text(ASCII, MaxLength = 12) NOT NULL,
       c.customer_name AS CustomerName
           TYPE Text(MaxLength = 120, MaxStorageBytes = 480)
FROM Customers AS c;
```

These declarations make several promises explicit:

- `CustomerId` contains Integer values and cannot be NULL.
- `CustomerCode` contains ASCII Text, with a maximum of 12 TextChars, and cannot be NULL.
- `CustomerName` contains Text subject to both a logical length limit and a separate storage-byte limit.

The supported explicit SELECT type names are `Integer`, `Float`, `Boolean`, and `Text`. Text uses UTF-8 by default and accepts the optional `UTF8`, `ASCII`, `MaxLength`, and `MaxStorageBytes` properties. A `NOT NULL` suffix adds an explicit NULL restriction.

### A checked promise

`TYPE` checks the value produced by the selected expression. It does not request a conversion.

For example, `TYPE Integer` does not parse a numeric-looking Text value. `TYPE Float` does not silently turn an Integer into a Float. A Text limit does not truncate or pad the result. If conversion or normalization is required, the application expresses that operation explicitly and then checks its result.

The check uses the value already produced. HN does not call a function a second time merely to validate its return.

A failed query does not publish a partly populated replacement result. Under the query API's publication contract, the previous destination remains available when a recoverable execution failure prevents publication. Explicit side effects performed by application callbacks remain the application's responsibility.

### Character length and storage length are different

`MaxLength` counts HN TextChars: Unicode extended grapheme clusters. A visible character can consist of several Unicode code points. Combining accents, flags, and joined emoji make the distinction practical.

`MaxStorageBytes` independently limits the number of UTF-8 bytes. A value can satisfy the character limit and still exceed a storage or protocol byte limit.

The two properties let an application express both concerns directly. HN does not infer a byte limit by multiplying a character limit by an assumed number of bytes per character.

`ASCII` specifies the permitted character range. It is not an alphanumeric-only or printable-only validation rule; an application needing those restrictions supplies the appropriate validation logic.

### Most ordinary projections need no annotation

When a query directly selects a field, its declared value metadata follows that field into the result, including applicable type, NULL, and length information.

```sql
SELECT c.customer_name AS CustomerName
FROM Customers AS c;
```

Renaming the field does not make it lose its type contract. This remains true when the query returns no rows, or when every selected value is NULL. The schema comes from declarations rather than from inspecting the first row or measuring the longest value currently present.

An explicit `TYPE` supplies a replacement output contract. Omitted limits therefore do not silently inherit every source constraint. Use an ordinary direct projection when the complete source value descriptor is the desired result.

The engine also accounts for the query's meaning. An outer join can make a result column nullable. A join can repeat a source primary key, so copying its value type does not imply that the result column is unique. Source defaults, key constraints, and value generators do not automatically become result-column guarantees.

### Describe the schema before consuming records

`hn_SQLDescribe()` derives the output schema without scanning records or executing HN functions and methods from the query. `hn_CursorDescribe()` provides metadata for an existing Cursor or WorkArea.

This is useful for report engines, exporters, generic data components, and tooling. They can inspect names, types, nullability, and value properties even when a dataset is empty. Description does not prove that every later runtime value or callback will succeed, but it provides a stable schema independently of the current data.

The `TYPE` annotation is HN Local SQL syntax. A backend query builder must handle it as HN metadata rather than sending the annotation unchanged to a PostgreSQL server.

## Why parity with backend data types matters

Local processing becomes fragile when data loses its meaning on the way out of the database.

A precise decimal may be converted into an approximate floating-point value. A large identifier may pass through a representation that cannot distinguish adjacent integers. A timestamp may lose its temporal meaning. A NULL may be replaced with an empty string, zero, or a default date. Application code then has to compensate for decisions that were made before the actual business logic began.

HN's Cursor and native type model is designed for broad parity with the common data types used by PostgreSQL and other backend servers. That includes preserving type-specific metadata instead of reducing every value to a small set of legacy file-storage categories.

| Data family | Meaning the HN model preserves |
| --- | --- |
| Integer families | Whole-number identity and the relevant size/range contract. |
| Exact decimal `Number` | Decimal meaning, with declared precision and scale where applicable. |
| `Float` | An explicitly approximate numeric value, distinct from exact decimal. |
| `Boolean` | Boolean values, with NULL represented separately where allowed. |
| Unicode `Text` | Text identity, encoding information, and applicable character/storage constraints. |
| `Raw` and bit strings | Binary bytes or logical bits, without treating them as character text. |
| Date, time, timestamps, and zoned forms | The distinction between a calendar date, a time, a local timestamp, and a zoned temporal value. |
| Intervals | A temporal quantity with its own semantics. |
| UUID | Identifier semantics rather than an arbitrary display string. |
| JSON | Structured JSON meaning and relevant backend representation information. |
| NULL | Absence of a value, distinct from zero, false, empty text, and defaults. |

This table describes the broader Cursor/native type model. The supported mappings and operators remain explicit: a type being representable in a Cursor does not imply that every SQL operation is defined for it. The scalar Local SQL examples in this article use Integer, Float, Boolean, Text, and NULL.

### Precision is part of correctness

PostgreSQL distinguishes exact `numeric` values from approximate floating-point values. HN's native type model likewise distinguishes exact `Number` values from approximate `Float` values. Keeping that distinction through materialization matters for prices, quantities, financial calculations, and comparisons. [PostgreSQL's numeric documentation](https://www.postgresql.org/docs/18/datatype-numeric.html) explains the underlying precision and scale concepts.

The same principle applies to identifiers. A key should retain its exact value throughout retrieval, local storage, comparison, and joining. It should not depend on the accidental precision of an intermediate representation.

### NULL and time retain their meaning

Local SQL uses SQL's three-valued logic. A comparison involving NULL can be unknown; `WHERE` and `HAVING` retain rows for which the condition is true. `IS NULL` expresses a NULL test directly. Preserving NULL from the backend prevents an unknown value from being mistaken for an actual zero, false, or empty Text value. See [PostgreSQL's comparison semantics](https://www.postgresql.org/docs/18/functions-comparison.html).

Temporal fidelity likewise requires explicit distinctions. A date is different from a timestamp, and a local timestamp differs from an instant interpreted with time-zone rules. Backend mappings must preserve the information actually supplied by the source. For example, PostgreSQL `timestamptz` does not retain the originally entered time-zone name; an adapter cannot reconstruct metadata the server no longer holds. See [PostgreSQL's date/time types](https://www.postgresql.org/docs/18/datatype-datetime.html).

### Broad parity keeps backend differences visible

Backend systems still have their own domains, collations, specialized types, extensions, and representation rules. JSON is a useful example: PostgreSQL `json` and `jsonb` preserve different aspects of the original representation. See [PostgreSQL's JSON documentation](https://www.postgresql.org/docs/18/datatype-json.html).

HN's advantage is a rich common model with explicit mappings. The application can retain the meaning needed for local work and deliberate writeback, while handling backend-specific details where they belong.

## Jobs and Tasks: acquire independent datasets concurrently

Many useful local queries depend on several independent backend requests. Fetching those results sequentially makes the application wait for each request before starting the next.

Jobs and Tasks let HN overlap that work. A Job coordinates a group of Tasks; each Task performs its own operation and has its own completion or error outcome. Concurrency limits keep the number of active requests appropriate for the application and its servers.

Consider a customer-service report:

| Task | Example source | Materialized Cursor |
| --- | --- | --- |
| Retrieve customer details | PostgreSQL CRM database | `Customers` |
| Retrieve orders for the reporting period | SQL Server order system | `Orders` |
| Retrieve open support tickets | MariaDB service system | `Tickets` |

These are illustrative source choices, using the application's supported backend adapters. The Local SQL processing does not depend on those particular database products.

The workflow is:

1. Start the independent Tasks within a Job.
2. Each Task runs its backend request in the appropriate connection context.
3. Each Task materializes its result into a typed Cursor.
4. The Job waits for the required results and handles failures.
5. The application supplies the completed Cursors to Local SQL as named sources.
6. Local SQL produces the dataset required by the report or response.

Materializing a result separates the data handed to the next stage from the live connection used to fetch it. Sharing Cursor data and coordinating its ownership are distinct from sharing a backend connection or transaction. Independent WorkAreas provide separate navigation; the Cursor ownership and synchronization rules govern access to the data itself.

When requests are independent and resources permit, the acquisition stage can approach the duration of the slowest request plus coordination and materialization overhead, instead of the sum of all request durations. Actual results depend on backend load, connection limits, transferred data, and the work each request performs.

Parallel retrieval also has a consistency boundary: several server queries do not automatically represent one atomic snapshot across all systems. A report that requires a common cutoff must establish that cutoff or the appropriate source transaction rules deliberately.

## Combining the results without multiplying records

Once the three Cursors are ready, the application can build a local processing pipeline.

The customer/order query shown earlier produces `CustomerOrderCounts`. A second query produces one ticket summary per customer:

```sql
SELECT t.customer_id AS CustomerId,
       COUNT(*) AS OpenTicketCount
FROM Tickets AS t
WHERE t.status = :openStatus
GROUP BY t.customer_id;
```

Here, `:openStatus` is a typed Text binding. The result can be supplied to another query as `TicketCounts`:

```sql
SELECT o.CustomerId,
       o.CustomerName,
       o.OrderCount,
       t.OpenTicketCount
FROM CustomerOrderCounts AS o
LEFT JOIN TicketCounts AS t
    ON t.CustomerId = o.CustomerId
ORDER BY o.OrderCount DESC, o.CustomerId;
```

Each summary has at most one row per customer, so their join preserves that intended result shape. A customer with no matching ticket summary has NULL in `OpenTicketCount`; the presentation or an explicit application transformation can supply zero when that is the required report convention.

This staged approach also avoids a common reporting mistake. Joining a customer's four orders directly with their three tickets produces twelve matching combinations. Counting after that join would count the combinations unless the query accounts for them. Aggregating each relationship first makes the intended cardinality clear.

Every intermediate result remains a normal local dataset. It can be inspected during debugging, reused for another report, or queried again without repeating the original backend requests.

## Results remain useful to ordinary HN code

A Cursor result is a natural choice when subsequent work needs named fields, schema inspection, record navigation, or another Local SQL operation.

Local SQL also offers object-oriented result shapes:

| Result mode | Application result |
| --- | --- |
| Default Cursor result | A materialized Cursor with declared output schema. |
| `INTO OBJECT` | One read-only system DataObject; NULL for no result rows; an error if more than one final row is produced. |
| `INTO OBJECTS ARRAY` | An Array of read-only DataObjects, one per result row; an empty Array for an empty result. |

Object properties use the selected column names or aliases and retain the supported scalar types and NULL states.

This lets the application choose a result shape appropriate to the next operation. A query pipeline can retain a Cursor; code consuming one logical record can use a DataObject; code processing a collection of records can use an Array of DataObjects.

## How this differs from approaches in other languages

Local querying is a well-established idea, and several ecosystems provide strong solutions. The useful comparison is how each approach fits into the surrounding application.

| Approach | How it relates to HN Local SQL |
| --- | --- |
| Handwritten loops and lookup structures | Give direct procedural control. HN also offers SQL for expressing relational selection, joins, grouping, and ordering over typed datasets. |
| C# LINQ | Provides language-integrated queries over collections. HN uses SQL Text over its native Cursor model, including explicit output schemas and HN-specific column contracts. |
| SQLite embedded in an application | Provides an in-process SQL database. HN's Local SQL operates within HN's own Cursor, value, and application-call model. |
| DuckDB with DataFrames or other local sources | Provides SQL over local data, including direct DataFrame integration. HN concentrates on integration with HN Cursors, native values, and application workflows. |

These alternatives are substantive capabilities: [LINQ queries collections](https://learn.microsoft.com/en-us/dotnet/csharp/linq/get-started/introduction-to-linq-queries), [SQLite runs in-process](https://www.sqlite.org/about.html), and [DuckDB can query Pandas DataFrames directly](https://duckdb.org/docs/2.0/guides/python/sql_on_pandas). External engines and libraries may also provide efficient integration without copying every input value.

HN's distinguishing benefit is the consistency of the complete workflow. Backend results become native typed data. SQL consumes that data. HN logic can participate in expressions. Result schemas remain available to subsequent processing. Jobs and Tasks coordinate the acquisition of independent inputs.

For developers familiar with the xBase tradition of working with local datasets, this retains a productive style of application development while extending it with a richer type model, explicit ownership, Unicode-aware Text contracts, and concurrent work.

## Future Cursor storage built on the Tensor storage foundation

The next storage direction reuses the lower-level storage foundation developed for HN Tensors. **This is future Cursor storage work.** It preserves the public role of a Cursor as a schema-aware record engine.

The shared foundation can support four useful physical representations:

| Representation | Useful when |
| --- | --- |
| Dense direct | Most positions contain values and repeated values offer little storage benefit. |
| Sparse direct | Many logical positions are absent or NULL, while present values are mostly distinct. |
| Dense dictionary | Most positions contain values, with substantial exact repetition. |
| Sparse dictionary | Many positions are absent or NULL and the present values also repeat. |

### Dictionary storage can reduce repeated payloads

A Cursor may contain a large number of repeated country names, statuses, categories, or other immutable values. Dictionary storage can retain an exact value once and use compact references from the places that contain it.

The benefit depends on the data. Repeated long values can offer substantial savings; mostly unique small values may be better served by direct storage. The storage foundation can make that choice without requiring application code to adopt a different logical schema.

Lossless representation is essential. Two values must not be merged merely because a case-insensitive comparison treats them as equal. Mutable objects must retain their identity. Compression of representation must preserve the values the application can observe.

### Sparse storage can avoid unnecessary payload allocation

Wide datasets often contain many optional fields. Sparse storage can represent those logical positions without allocating a complete value payload for every cell. Its storage benefit depends on the density of populated values and the overhead of tracking them.

The meaning of each position remains intact: NULL, zero, empty Text, and a default value are different states. A sparse representation must preserve those distinctions. It also preserves records; sparse values do not mean that a row disappears.

### Storage reuse preserves Cursor behavior

A Cursor still needs field contracts, record identity, indexes, navigation, and coordinated updates. Those responsibilities remain with the Cursor engine even when it shares storage mechanisms with Tensors.

The implementation therefore does not require a public Tensor per column, or one giant two-dimensional Tensor exposed behind the Cursor. Applications continue to work through Cursor rules, and storage representation changes preserve logical identity and contents.

Potential benefits include a smaller memory footprint for suitable datasets, reduced allocation and copying of repeated payloads, and reuse of tested storage-management code. Performance and memory savings depend on the workload; dictionary or sparse storage alone does not establish a particular SQL execution speed.

The Tensor integration also includes a route to explicit rank-2 SQL result materialization, shaped as rows by columns. Such positional output serves different consumers from a Cursor's named fields and record-navigation model. It belongs to the continuing Tensor integration, alongside the shared storage work.

## Where the practical advantage appears

Local SQL is especially useful when the application needs a combined view that no single backend conveniently owns.

Examples include comparing records across systems, combining independently retrieved datasets, producing several reports from one retrieval, enriching server data with local information, and preparing a web response from several completed Tasks.

The application can keep its stages explicit: retrieve the useful source data, preserve its types in Cursors, apply local relational transformations, and pass the result to ordinary HN code. Each stage has a clear purpose and an inspectable result.

Good backend queries still matter. Filtering, projecting, and aggregating at the source can reduce the amount of data that must cross the network. Local SQL then handles the work that benefits from the application's broader context. Local Cursor changes and results are persisted to a backend through an explicit write operation.

The practical result is less repetitive data-handling code, reusable query results, clearer schemas, and fewer avoidable conversions between the database and the application. Developers can combine SQL's relational model with HN's native values and procedural logic, using Jobs and Tasks to obtain the inputs efficiently and Cursors to keep those inputs useful throughout the application.
