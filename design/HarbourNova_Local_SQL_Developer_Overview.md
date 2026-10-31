# Local SQL in HarbourNova: bringing SQL to application data

**Developer overview of the HN1 Local SQL programming model — updated October 31, 2026**

> **Scope:** This article describes the approved HN1 programming model. Some features, including Cursor sharing and storage optimizations, are still being completed. Current SQL type and result-object limits are identified where relevant.

Database servers are very good at selecting, joining, and summarizing the data they manage. Applications often need another step: combine results from several queries, add information from another system, apply application logic, and produce the dataset required by a report, screen, export, or web response.

HarbourNova (HN) brings SQL into that part of the application through **Local SQL over native Cursors**.

A backend query can produce a Cursor. Application code can also create and populate one. The application accesses these datasets through scoped WorkArea bindings, and Local SQL uses the visible source names to select, join, and summarize their data. A query supplies its result through an explicit reference to a receiving variable: a WorkArea, a single row object, or an Array of row objects. LOCAL, STATIC, TASK STATIC, and application GLOBAL declarations give WorkArea bindings their normal visibility and storage lifetime. Jobs and Tasks extend this workflow by allowing independent backend requests to run concurrently before their results are combined locally.

The value comes from the combination: SQL, typed local data, ordinary HN code, and concurrent work all participate in one programming model.

## What “Local SQL” means

Local SQL executes inside the HN application against data that the application already holds. Names in `FROM` and `JOIN` resolve through the executing code's permitted WorkArea bindings. Local/formal bindings, visible retained bindings, and the owning application's globals participate according to scope. Library and HNB code retain their own boundaries; application globals are not a fallback namespace for their SQL. The query uses familiar SQL concepts such as `SELECT`, `FROM`, `WHERE`, joins, grouping, and ordering.

The reference dialect is PostgreSQL, with a defined supported subset and selected HN extensions. Two useful extensions are optional `TYPE` declarations on output columns and the ability to reference a selected column's alias in `HAVING`.

SQL is supplied as HN Text. It can be a fixed query, Text assembled by application code, or the output of a query builder. Application values are supplied through an ordinary binding Hash. The SQL specifies the result mode: `INTO CURSOR`, `INTO OBJECT`, or `INTO OBJECTS ARRAY`. The call supplies the actual receiving variable through `xResult=@destination`. Source names remain part of SQL; the HN compiler resolves the destination-variable expression under ordinary language visibility and reference-argument rules.

For example, a query can use `ListOfCustomers` and `ListOfOrders` even when those datasets came from different servers. Once materialized as Cursors and opened through those WorkAreas, their original locations are separate from the local relational operation.

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
| **Alias** | A name in a permitted scope that resolves through a binding to a WorkArea; a declared WorkArea binding can exist while its value is still NULL. |

Separate WorkAreas can refer to the same Cursor while navigating independently. Moving one view to another record does not move the other view. Within one execution, two references to the same WorkArea still refer to the same navigation state. When Main and Tasks share published read-only Cursor data, each reader uses its own WorkArea.

In everyday code, developers often call `ListOfCustomers` a Cursor name. More precisely, it is the WorkArea alias through which that Cursor is accessed. Local SQL resolves that name and obtains the underlying data automatically. The application does not supply a separate `aSources` collection or pass a Cursor handle to identify an already-open WorkArea.

The examples use descriptive WorkArea names directly. SQL relation aliases remain available when useful, such as naming the two roles in a self-join, but short names are not required.

Local SQL reads its sources independently of their current WorkArea record pointers. A report query therefore does not have to inherit whichever record a screen or another routine last visited.

A materialized SQL result is independent of its input WorkAreas. The receiving variable holds a reference to the new view, and that reference follows its declaration's lifetime. Other returned, captured, or retained references can keep the result alive after that variable leaves scope or is reassigned. Closing an input WorkArea does not invalidate the separately materialized result.

## Cursor access: private read/write or shared read-only

HN1 gives Cursor data two access states. An editable Cursor belongs to **one owner: Main or one Task**. Once the owner publishes the completed Cursor for sharing, **Main and Tasks may read it, and nobody may change its data**.

| Cursor state | Who may read its data? | Who may change its data? |
| --- | --- | --- |
| **Private read/write** | Only its owning Main execution or Task. | That same owner. |
| **Shared read-only** | Main and any participating Tasks that have received access to the published Cursor. | Nobody, including the original owner. |

### One owner while the data is editable

The owner can build, read, and update its private Cursor through the available Cursor operations. Other executions cannot read that private dataset while it remains editable. Having a reference in a LOCAL, STATIC, or GLOBAL variable does not change this rule: the access restriction belongs to the Cursor data itself.

Before publication, an explicit ownership handoff may transfer exclusive access to another execution. Read and write access move together; the handoff does not allow the old and new owners to use the private data at the same time.

### Publish completed data for shared reading

After filling and preparing the dataset, its owner can publish it for shared reading. Publication makes that Cursor's data permanently read-only through every reference. The original owner also loses permission to change fields, add or remove records, or alter the published schema.

Main and several Tasks can then query or scan the same published dataset concurrently. Each sees stable field values and record membership for that version. Giving another execution a reference still requires the normal application handoff; publication does not automatically create aliases or WorkAreas in every Task.

If preparation for publication fails, the Cursor keeps its previous contents and access state. Sharing begins only after publication completes successfully.

### Each reader keeps its own navigation

Read-only Cursor data remains fully navigable. Each reader has its own WorkArea, current record position, and active-order choice. A Task can move through its records without moving Main's record pointer or another Task's pointer.

The shared data is read-only; each reader's navigation state remains local to its view. Sharing a Cursor does not mean sharing one mutable WorkArea among all readers.

### Edit a new copy when the data must change

A published Cursor does not switch back to read/write. An execution that needs changes creates an independent private copy and edits that copy. It may then publish the completed replacement for readers.

Existing readers and WorkAreas keep the original version until they explicitly acquire the replacement. Publishing a new version does not redirect their current views or change records underneath an ongoing scan. Old and new versions can coexist while references to both remain; making a copy may therefore require additional memory.

For example, Main can publish a customer dataset for several reporting Tasks. A refresh Task prepares an updated private copy and publishes the replacement. Reports already using the earlier version can finish consistently, while later reports acquire the updated version.

### Query-result assignment and shared publication are separate

`hn_SQLSelect(..., xResult=@destination)` assigns a completed query result to an HN variable. That assignment does not itself publish a Cursor for cross-Task reading. An output written to a GLOBAL still follows the Cursor's access rules.

A query can read published input data and produce an independent result for its caller. Creating or assigning that result does not modify the input, make the published input editable, or replace the version held by other readers.

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

In the SQL-only examples below, `cSQL` denotes the complete SQL Text shown in the preceding block. The accompanying HN call supplies the destination and any required bindings. Check each call's return value before consuming its new result; later examples omit repetitive application-specific error handling.

For example, given the local WorkAreas `ListOfCustomers` and `ListOfOrders`:

```sql
SELECT ListOfCustomers.customer_id AS CustomerId,
       ListOfCustomers.customer_name AS CustomerName,
       COUNT(ListOfOrders.order_id) AS OrderCount
FROM ListOfCustomers
LEFT JOIN ListOfOrders
    ON ListOfOrders.customer_id = ListOfCustomers.customer_id
GROUP BY CustomerId, CustomerName
ORDER BY OrderCount DESC, CustomerId
INTO CURSOR
```

The receiving variable and call are:

```harbour
LOCAL ListOfCustomerOrderCounts WorkArea

nRows := hn_SQLSelect(cSQL, xResult=@ListOfCustomerOrderCounts)
```

On success, `ListOfCustomerOrderCounts` contains one row per customer, including customers with no matching orders. Assuming `order_id` is non-NULL on actual orders, `COUNT(ListOfOrders.order_id)` reports zero for an unmatched customer. `COUNT(*)` would count the NULL-extended join row instead, which is a useful example of why normal SQL semantics matter locally too.

The application can add:

```sql
HAVING OrderCount > 5
```

before `ORDER BY` to keep only customers with more than five orders. HN makes the selected alias `OrderCount` available in `HAVING`.

The same approach applies to reconciliation, duplicate detection, exception lists, reporting, and combining current and archived datasets. A developer expresses the relationship and desired result directly, instead of maintaining several nested loops and temporary lookup structures by hand.

## Passing application values into a query

Application values are supplied through an ordinary Hash. Each key names a SQL binding, and its value supplies the corresponding native HN value. No separate input type declaration is required.

Assuming `ListOfOrders` is already an open WorkArea and `iCustomerId` contains a non-NULL customer identifier, a typical call creates a local result:

```harbour
LOCAL ListOfCustomerOrders WorkArea

cSQL := "SELECT ListOfOrders.order_id, ListOfOrders.customer_id " + ;
        "FROM ListOfOrders " + ;
        "WHERE ListOfOrders.customer_id = :customerId " + ;
        "INTO CURSOR"

hBindings := { "customerId" => iCustomerId }

nRows := hn_SQLSelect(cSQL, ;
    hBindings=hBindings, ;
    xResult=@ListOfCustomerOrders)

IF nRows < 0
    errSQL := hn_SQLLastError()
    // Application-specific error handling uses errSQL.
ENDIF
```

The declaration registers `ListOfCustomerOrders` as a local WorkArea binding and initializes its value to NULL. The query reads `ListOfOrders`, materializes its result, and assigns a new WorkArea reference through `xResult=@ListOfCustomerOrders`. The output argument identifies the actual receiving variable even while its value is NULL. `hn_SQLSelect()` itself returns the number of result rows. A recoverable query failure returns `-1`, with the structured HN error available through `hn_SQLLastError()`. The output is a WorkArea; its separate `:CursorHandle` property exposes the underlying managed Cursor when needed.

### SQL names and HN variable names have separate roles

| Part of the example | Meaning |
| --- | --- |
| `ListOfOrders.customer_id` | The field being compared. |
| `:customerId` | The named placeholder in the SQL Text. |
| `"customerId"` | The matching Hash key, without the leading colon. |
| `iCustomerId` | The HN variable whose current value is supplied. |
| `@ListOfCustomerOrders` | The assignable receiving variable, resolved as an HN reference argument. Its name is not part of the SQL Text. |

The Hash key does not have to match the HN variable name. The same SQL can receive a value from a local variable, an object property, or a calculated expression. These names act as placeholders, with values bound directly through the native interface. Values are never pasted into the SQL Text, so quotes and other SQL-looking characters inside Text remain ordinary data.

Constructing the Hash evaluates its value expressions. The query then captures the supplied scalar values before scanning records or invoking callbacks. Reassigning the original variable does not automatically change an existing Hash entry, and changing the Hash during execution does not change a value already captured by that query.

The SQL Text needs no terminating semicolon. The semicolons outside the quoted SQL Text in the HN examples are line-continuation characters.

### Input types come from the native values

There is no binding-side `"Type"` property. HN obtains the type from the supplied native value; this works even when the source variable itself has no explicit type declaration.

An Integer remains an Integer. A Float remains a Float even when its current value is mathematically integral. Text containing digits remains Text. Supported adapters preserve the supplied value's type; the executor does not guess a type from the binding name or parse Text to make it resemble another type. The current scalar SQL subset supports Integer, Float, Boolean, Text, and NULL; native decimal Number integration remains separate work.

A value's type and its original variable declaration are separate information. For example, the current length of a Text value does not reveal the maximum length allowed by the original declaration. Available declared metadata can be preserved internally, but ordinary bindings do not require the developer to repeat it in a descriptive record.

### NULL tests are written in SQL

HN1 requires each supplied binding entry to contain a non-NULL native value, object reference, or callable reference. A missing referenced key or an entry containing NULL is an error detected before record processing. A query with no bindings can omit `hBindings` entirely.

To select orders without a customer, write the condition directly:

```sql
SELECT ListOfOrders.order_id, ListOfOrders.customer_id
FROM ListOfOrders
WHERE ListOfOrders.customer_id IS NULL
INTO CURSOR
```

Supply the output explicitly:

```harbour
LOCAL ListOfOrdersWithoutCustomers WorkArea

nRows := hn_SQLSelect(cSQL, xResult=@ListOfOrdersWithoutCustomers)
```

Use `IS NOT NULL` for the opposite test. An application that searches either for a particular customer or for missing customer identifiers chooses the appropriate SQL condition; only the first form needs the `customerId` binding.

This is a restriction on the supplied binding entry. NULL remains valid in source fields, SQL expressions, outer-join results, object properties, and nullable function or method returns. The keyword `NULL` is valid directly inside the SQL string. A local WorkArea destination starts at NULL simply from its declaration; it is separate from the input binding Hash.

## Function-local WorkAreas

Local WorkAreas let a function own its temporary datasets using the same names in HN statements and SQL. They also let a helper receive a source dataset without depending on the caller's choice of alias.

### A declaration supplies a local name

```harbour
LOCAL ListOfCustomerOrders WorkArea
```

This declaration initializes the variable to NULL and registers `ListOfCustomerOrders` in the current function invocation's WorkArea namespace. No initializer is required. The type annotation constrains a non-NULL value to a WorkArea reference; NULL remains permitted.

The declaration creates a binding, not a dataset. Until a WorkArea is assigned, there is no Cursor to query or traverse through that binding. The receiving slot is nevertheless available through `@ListOfCustomerOrders`. An output argument does not require an existing WorkArea or Cursor.

Typed WorkArea parameters participate in the same source-naming mechanism. Each invocation has its own local bindings, so recursive calls and separate functions can use readable names without colliding with one another. A declaration must still obey the rule against hiding a GLOBAL that is visible unqualified in that scope.

Source-name resolution follows the executing code's permitted scopes: the applicable local/formal bindings, visible retained bindings, and owner-wide bindings. A nearer WorkArea binding remains authoritative even while its value is NULL. Attempting to use it as a source produces an error; lookup does not skip it to find a different populated binding. The no-shadowing rule separately rejects a local or retained declaration that would hide an unqualified-visible GLOBAL.

### A reference identifies the destination even while NULL

`FROM ListOfOrders` resolves a source binding and reads its current WorkArea value. `INTO CURSOR` chooses the result representation, and `xResult=@ListOfCustomerOrders` identifies the receiving variable. Destination selection does not inspect the variable's initial NULL value or search for its name inside SQL Text.

The compiler resolves the HN destination expression, its visibility, and its assignability. It constructs a reference to the correct invocation-local, retained, Task-specific, or global cell. The actual storage is selected for the executing invocation or Task as applicable. This is independent of whether the SQL is a literal or assembled at runtime.

The executor still parses the SQL result mode and validates that the supplied output reference can receive that representation. Missing output references and known incompatible destinations fail before record processing. Arbitrary SQL Text does not become a compile-time SQL program merely because its destination variable is compiled.

The complete result is prepared before the final output assignment. On success, the destination receives a new WorkArea reference. If it already holds a WorkArea, the assignment replaces that reference; other owners of the previous view retain it normally. Assigning a new view does not rebind the Cursor inside the previous view or publish it for shared reading.

A successful zero-row query still assigns a valid WorkArea with the result schema. NULL means no WorkArea has been supplied; an empty result is a usable dataset containing zero records.

The native result form has one destination authority. `INTO CURSOR Name` is not the HN Cursor-output syntax, and supplying a name as well as `xResult` is rejected. A missing destination never implicitly creates a PUBLIC-style name or assigns a result in a wider scope.

Failure prevents the engine from assigning a partial replacement. Independent callback assignments and concurrent application writes retain their normal effects; the query does not roll them back. The normal reference-lifetime and checked-assignment path remains responsible for safe output assignment, including shared STATIC or GLOBAL destinations.

### Pass sources to a helper and return its local result

The following function receives a WorkArea under the local name `ListOfSourceOrders`, uses it in SQL, and returns a locally created result:

```harbour
FUNCTION BuildCustomerOrders(ListOfSourceOrders WorkArea Required, ;
                             iCustomerId Integer Required)

    LOCAL ListOfCustomerOrders WorkArea
    LOCAL cSQL Text
    LOCAL hBindings Hash
    LOCAL nRows Integer

    cSQL := "SELECT ListOfSourceOrders.order_id, " + ;
            "ListOfSourceOrders.customer_id " + ;
            "FROM ListOfSourceOrders " + ;
            "WHERE ListOfSourceOrders.customer_id = :customerId " + ;
            "INTO CURSOR"

    hBindings := { "customerId" => iCustomerId }

    nRows := hn_SQLSelect(cSQL, ;
        hBindings=hBindings, ;
        xResult=@ListOfCustomerOrders)

    IF nRows < 0
        // hn_SQLLastError() provides the structured query error.
        RETURN NULL
    ENDIF

    RETURN ListOfCustomerOrders
```

In a caller that already has a populated `ListOfOrders` and a non-NULL customer identifier:

```harbour
LOCAL ListOfSelectedOrders WorkArea

ListOfSelectedOrders := BuildCustomerOrders(ListOfOrders, iCustomerId)
```

The same input WorkArea is known as `ListOfOrders` in the caller and `ListOfSourceOrders` in the helper. The helper's result is initially named `ListOfCustomerOrders`, then retained by the caller under `ListOfSelectedOrders`. Passing and returning these values does not rename another function's bindings or copy the rows merely to change the local name.

The helper returns a WorkArea reference as an ordinary HN value. This is separate from `hn_SQLSelect()`, whose own return remains the numeric row count or `-1`. An empty successful result returns a valid WorkArea from this helper; a failed query returns NULL under the helper's stated error-handling choice.

`Required` means the caller must supply the argument. It does not itself prohibit explicit NULL. Operations still validate their inputs: SQL needs a usable source WorkArea, and the scalar `customerId` binding must satisfy the non-NULL binding rule.

An ordinary helper does not search its callers' private WorkArea namespaces. A generic SQL helper must receive the WorkAreas it needs and establish its own bindings. Passing only a string containing another function's private alias does not transport the WorkArea or identify the caller's scope.

### Names, references, and lifetime

The three forms below have distinct roles:

| Form | Meaning |
| --- | --- |
| `ListOfOrders` in an HN expression | The WorkArea value currently stored in that variable. |
| `"ListOfOrders"` | Text containing a name; resolving it requires a WorkArea namespace. |
| `@ListOfOrders` as a by-reference argument | The assignable variable slot; `hn_SQLSelect(..., xResult=@ListOfOrders)` can place a new WorkArea reference into it. |

The automatic alias is associated with the scoped binding. It is not a unique permanent name stored on the Cursor. Two local variables can hold the same WorkArea, and a function parameter can provide another local name for that view.

Local names disappear when their scope ends. Local references are released through normal scope cleanup, after applicable `FINALLY` handling. Returned or captured references and other owners can keep the WorkArea alive. A returned result is retained for the caller before the helper's local references are released.

Selection state belongs to the function's WorkArea context, so selecting a WorkArea in a helper does not permanently change the caller's selected WorkArea. Within the WorkArea's permitted owning execution, a helper and caller using that same WorkArea share its record position. Separate WorkAreas over the same Cursor provide independent navigation. Passing a WorkArea between routines does not by itself authorize another Task to use it.

Releasing one variable's reference is also different from explicitly closing a WorkArea. Other retained references can keep a view alive after one variable is cleared. Explicitly closing a view affects references to that same view; other readers' separate WorkAreas over published Cursor data remain separate views.

## WorkArea scope, retained state, and application globals

A WorkArea variable has both a declaration and a current value. Its declaration determines where code can name the variable and which storage cell that name identifies. Its value is either NULL or a native WorkArea reference. The WorkArea itself owns its navigation state and Cursor binding. Keeping these responsibilities separate makes local datasets, retained views, and deliberately shared application state follow the same language rules.

### Scope and storage are separate choices

The scope of a declaration controls name visibility. Its storage class controls whether each invocation or Task receives a separate cell, or whether authorized code in Main and Tasks shares one retained cell. Sharing a variable cell does not change the access state of the Cursor or WorkArea referenced by that cell.

| Declaration and location | Code that can name it | Storage ownership |
| --- | --- | --- |
| LOCAL or a formal parameter | The permitted lexical scope of the current routine invocation | Invocation-local binding; a by-reference parameter refers to the supplied cell under normal lifetime rules. |
| STATIC inside a routine or method | That routine or method's permitted scope | One retained cell for that declaration, shared across authorized Tasks. |
| TASK STATIC inside a routine or method | That routine or method's permitted scope | A separate retained cell for each logical Task. |
| STATIC outside routines in an `.hnp` file | Functions, methods, and event bodies defined in that source unit, subject to normal lexical rules | One source-unit-visible retained cell, shared across authorized Tasks. |
| TASK STATIC outside routines in an `.hnp` file | The same source-unit scope | A separate retained cell for each logical Task. |
| LIBLOCAL STATIC | Authorized source units of the defining library | One retained cell owned by the admitted library instance and generation. |
| LIBLOCAL TASK STATIC | Authorized source units of the defining library | A separate retained cell for each Task using that library instance. |
| GLOBAL in application code | The application's selected source units | One application-runtime-owned retained cell, shared across authorized Tasks. |

An `.hnp` filename does not create an independent runtime owner. A source unit belongs to the application or library into which it is built. An HNB Module is a compiled HN bytecode module and preserves its defining module and library boundaries.

TASK STATIC separates the variable cells. Copying a WorkArea reference into another cell does not create another view or authorize another Task to use that WorkArea. Each reader of published Cursor data uses its own WorkArea for independent navigation. An ordinary LOCAL already belongs to its invocation; no additional TASK LOCAL declaration is needed.

### GLOBAL is a general typed declaration

GLOBAL supports ordinary storable HN values under their normal type and ownership contracts. WorkArea is one useful application of that facility. Text, Integer, Boolean, and other supported values can also have global declarations.

Declarations appear outside routines and can be distributed among the application's selected source files:

```harbour
// Customers.hnp
GLOBAL ApplicationCustomers WorkArea
GLOBAL ApplicationTitle Text := "Order Management"

// Orders.hnp
GLOBAL ApplicationOrders WorkArea
GLOBAL ApplicationOrderSummary WorkArea
GLOBAL CompletedRequests Integer := 0
```

These declarations do not need to be collected into one special `.hnp` file. Each global has one authoritative definition in its owning application scope. Other application source units use that declaration without repeating it or adding MEMVAR declarations.

A WorkArea declaration starts with NULL and allocates no WorkArea or Cursor. When code assigns a WorkArea, the cell owns its reference through normal retention rules. Replacing or clearing the cell releases that ownership; other valid references can keep the previous view alive.

Initialization follows the retained-storage rules for the declared type. Source-unit GLOBAL initialization is established at first actual access, including a write or passing the variable by reference. Concurrent first accesses coordinate around the same initialization state. Initialization failures and dependency cycles follow the normal retained-variable contract. GLOBAL does not impose one universal default value on every datatype or make source-list order an initialization order.

The global cell belongs to one application runtime lifetime. Starting another Task does not copy its value or create another global cell. Separate application runtime instances have separate globals, including independently running application processes.

### Uniqueness and hiding are checked during the build

Two active definitions of the same global identifier in the same owner scope are an error, even when their types and initial values match. Identifier comparison follows HN's normal identifier rules. Only source units and conditional declarations selected for the target contribute to its declaration environment.

HN also rejects an ordinary narrower variable declaration that would hide a GLOBAL visible unqualified in that scope. This includes LOCAL declarations, nested lexical locals, formal parameters, routine or file STATIC declarations, TASK STATIC declarations, and applicable owner-wide retained declarations. An application GLOBAL named `ApplicationOrders` therefore cannot be hidden by a same-named parameter in an application function.

The rule concerns visibility in the actual owner scope. A separately compiled library can use its own local `ApplicationOrders` because the application's global is not visible there. Field selectors, explicitly qualified object members, and SQL column aliases have their own resolution contexts; they do not become conflicting variable declarations merely because their spelling matches a global.

These checks are independent of source-file order. They also apply to incremental builds. Adding a new GLOBAL can conflict with a declaration in an otherwise unchanged source unit, including a unit that has never referred to that global. The compiler's declaration information and build dependencies account for that change before cached compilation results are reused.

### SQL reads scoped sources and receives an explicit destination

WorkArea source lookup follows the permitted lexical and owner environment: local bindings and parameters, applicable retained bindings, then the globals visible to that code. A registered NULL binding is an invalid source. Lookup does not skip it in search of a usable wider value. The no-hiding rule prevents narrower application declarations from concealing visible application globals in the first place.

Only WorkArea bindings participate in this alias mechanism. Defining a global Text or Integer does not make its identifier a SQL data source, and SQL Text does not gain general reflection over ordinary variables.

A result destination is supplied through an actual reference argument. The following example executes within the source WorkArea's permitted owning execution. Given a populated `ApplicationOrders`, application code can assign a summary to its declared global:

```harbour
cSQL := "SELECT ApplicationOrders.customer_id, COUNT(*) AS OrderCount " + ;
        "FROM ApplicationOrders " + ;
        "GROUP BY ApplicationOrders.customer_id " + ;
        "INTO CURSOR"

nRows := hn_SQLSelect(cSQL, xResult=@ApplicationOrderSummary)
```

The compiler identifies `ApplicationOrderSummary` as the receiving declaration. The runtime reaches its actual cell through the reference argument. The SQL executor resolves `ApplicationOrders` as a source in the application's permitted WorkArea scope. These operations have different purposes and need no duplicate destination name inside the SQL.

The newly created WorkArea becomes the global's value only through successful output assignment. A zero-row result still supplies a valid schema-bearing WorkArea. Assigning the new reference does not rebind the Cursor inside the previous WorkArea, redirect other holders of that view, or publish the new Cursor for cross-Task reading.

### Libraries and HNB modules preserve their own scope

Application globals are visible within application code. Independently owned libraries and HNB modules cannot discover or access those declarations by name. Static linking preserves this boundary just as dynamic loading does. Calling a library routine in an application's current Task does not import the application's variable scope into that routine.

The same restriction governs SQL executed in library or HNB scope. Its source resolver does not fall back to the host application's globals or search arbitrary caller frames. Passing a SQL string containing an application-private WorkArea name does not pass that WorkArea.

Application code supplies dependencies explicitly through parameters, objects, Callables, or other supported interfaces. A WorkArea parameter provides a local binding inside the receiving routine. An explicitly supplied output reference permits assignment to that particular receiving variable under its normal type, lifetime, and sharing rules. It does not expose the caller's other bindings or transfer the caller's private SQL source namespace.

Library-private retained state uses LIBLOCAL STATIC and LIBLOCAL TASK STATIC. A library's declared public interface is reached through its consumer alias and export rules. Neither the alias nor the existence of application GLOBALs makes private library declarations public. The application global namespace and a library's public export surface remain separate contracts.

### Shared cells and shared values require different guarantees

An individual read or direct assignment of a shared GLOBAL cell is coherent and ownership-safe across authorized Tasks. Each operation has one observable point of effect. A reader obtains a valid value; reference acquisition preserves the referenced value's lifetime against concurrent replacement of the cell.

This is the same basic cell guarantee used for shared scalar STATIC storage. It does not require one particular lock or CPU instruction for every datatype. Costs depend on representation, access patterns, contention, and lifetime handling. Recognizing a declaration through stable symbol metadata is distinct from acquiring or replacing its shared value.

Separate operations remain separate. A read followed by arithmetic and assignment is not an atomic increment, including ordinary shorthand compound assignment. A check followed by a later read can observe a changed global. Several assignments do not create a transaction or a coherent snapshot of multiple cells.

Within a WorkArea's permitted owning execution, code that needs to keep using the same reference after checking it can capture that reference once in a distinct local:

```harbour
LOCAL OrdersView WorkArea

OrdersView := ApplicationOrders

IF OrdersView != NULL
    ? OrdersView:RecordCount
ENDIF
```

The local keeps the WorkArea it acquired even if another Task later changes `ApplicationOrders`. This retains the view; it does not copy the dataset or create an independent record pointer. Capturing a reference does not grant a different Task permission to use that WorkArea. Readers in different executions use separate WorkAreas over published read-only Cursor data.

GLOBAL does not publish private Cursor data or make a WorkArea usable from another execution. An editable Cursor still has one owner; a published Cursor has read-only data for all readers. Other collections, objects, and native resources retain their own sharing, access, and lifetime rules.

SQL result assignment through a global output reference uses that cell's normal checked assignment path. Eliminating destination-name lookup does not eliminate synchronization or turn an entire query into a multi-Task transaction. A failed engine assignment does not supply a partial result; independent callback assignments and other application writes retain their normal effects.

### GLOBAL and ApplicationValues serve different needs

`ApplicationValues()` returns the application-runtime-owned managed container shared by participating HN Tasks. Its keys are selected at runtime, and its entries have explicit fixed type contracts. It is not an untyped substitute for variable declarations.

| Property | GLOBAL | ApplicationValues entry |
| --- | --- | --- |
| Identity in source | A declared identifier known to the compiler | A key supplied through the container interface |
| Type contract | The global declaration | An explicit fixed entry contract |
| Lifetime | The owning application's retained cell | The managed entry lifetime, including explicit removal and recreation |
| Individual access | Coherent, ownership-safe cell access | The container's managed entry operations |
| Compound numeric updates | Ordinary read/modify/write is not implicitly atomic | Specifically supported single-entry numeric updates are atomic complete operations |
| Mutable collection handling | The stored value retains its existing ownership and sharing contract | Supported Array/Hash inputs become independently owned managed state with protected nested access |

ApplicationValues distinguishes an absent entry from an existing entry containing NULL. Duplicate entry definition is an error. Removing and recreating an entry establishes a new lifetime; stale revisions or guards cannot silently retarget to it. Copy-out and checked replacement support deliberate updates without exposing an ordinary live mutable child that bypasses the container's protection.

Use GLOBAL for statically declared application state whose name and type belong in the program's contract. Use ApplicationValues when runtime-selected keys, managed entry lifetimes, or its entry-level operations fit the problem. No automatic mirroring connects a global declaration to a similarly named entry. The application-global visibility boundary does not disable this separate runtime service.

### Compiler metadata supports the normal build workflow

Each selected `.hnp` source unit contributes compiler-owned `.hnp.sig` metadata. Global declarations add owner, identity, declared type and constraints, and dependency information to that metadata. The build collects the selected declarations before compiling bodies against the completed environment. Duplicate definitions, visibility violations, incompatible uses, and global-hiding conflicts receive build diagnostics.

Applications continue using their ordinary source lists and build stages. They do not maintain a second global-declaration file or repeat declarations in every consumer. The compiler remains responsible for understanding HN source; the Rust build tool consumes its metadata rather than independently parsing HN declarations.

Compiled variable accesses can use their resolved declaration identities. SQL supplied as runtime Text still resolves its source names in the appropriate context and validates its result mode against the supplied output reference. This preserves dynamically assembled SQL without making destination selection depend on a string search.

### Inspecting names and live WorkAreas

Inspection distinguishes visible bindings from distinct live WorkArea values. Several permitted names can refer to one view, and several views can share one Cursor. A declared NULL WorkArea binding is a visible name without a currently supplied view. A retained view can remain alive after one of its local names has gone out of scope.

Inspection respects the caller's permitted owner scope. A membership snapshot can retain the WorkArea references it reports, while later assignments change the live bindings independently. Retention preserves reference identity and lifetime; it does not publish private data or grant cross-Task access. Published Cursor data is already immutable, while each WorkArea retains its own navigation state.

`WorkAreaNumber` identifies a view within its owning registry. It does not supply ownership, make a hidden name visible, or authorize access from another Task. Registry storage can grow as views are admitted; the language does not require every WorkArea declaration to reserve an eager dataset or a fixed preallocated numeric slot.

## SELECT, SCAN, and explicit field access

Local SQL results participate in ordinary HN record processing. In this example, `ListOfOrders` is bound to a Cursor containing a field named `Number`:

```harbour
SELECT ListOfOrders

SCAN ALL
    ? ListOfOrders->Number
ENDSCAN
```

`SELECT ListOfOrders` resolves the visible WorkArea binding and selects its current WorkArea. `SCAN ALL` traverses the applicable records from the beginning under the WorkArea's ordering and record-visibility rules. On each iteration, `ListOfOrders->Number` reads the field from the current record, and `?` prints that value to the normal standard-output stream.

The same name resolution is used by `SELECT ListOfOrders`, `ListOfOrders->Number`, and SQL `FROM ListOfOrders`. No parentheses or additional source map are needed merely because the WorkArea is local.

Field access is explicit. `->` identifies a field in a record; `:` identifies a native property or method on the WorkArea value. This prevents a data field from colliding with a WorkArea property:

```harbour
? ListOfOrders->RecordCount    // A data field named RecordCount
? ListOfOrders:RecordCount     // The Cursor's record count
```

The source of a scan is retained for the traversal. Selecting another WorkArea in the body does not redirect its continuation; the originating WorkArea is restored as required for subsequent iterations and completion or unwind. A valid empty Cursor executes the body zero times. A local that still contains NULL is not a usable scan source.

### Ordinary SCAN and explicit PRESCAN

Ordinary `SCAN ALL` uses the normal streaming traversal. HN1 also provides explicit `SCAN PRESCAN` behavior for work that needs its record membership and visit order established before the body begins.

PRESCAN prepares the ordered record identities, evaluates its preparation predicates, and then executes the body against that prepared sequence. For an editable private Cursor, body operations may change field values without changing the prepared membership or order. For a published Cursor, its data already remains unchanged for the lifetime of that published version.

Records already deleted before preparation are excluded. If a prepared record is deleted before its scheduled visit, `ON DELETED ERROR` is the default; `ON DELETED SKIP` explicitly skips it. For example:

```harbour
SELECT ListOfOrders

SCAN PRESCAN ALL ON DELETED SKIP
    ? ListOfOrders->Number
ENDSCAN
```

The deletion option applies to prepared records. It does not permit another Task to modify published data or turn unrelated errors into skipped records.

The two forms have different observable timing, so ordinary SCAN is not silently converted into PRESCAN. Both use the same WorkArea identity and explicit field-access model.

## Read-only WorkArea properties

WorkArea properties expose the navigation view and its underlying dataset directly:

| Property | Meaning |
| --- | --- |
| `ListOfOrders:WorkAreaNumber` | The numeric identifier of this WorkArea in its owning runtime registry. |
| `ListOfOrders:RecordNumber` | The WorkArea's current record number under HN navigation rules. |
| `ListOfOrders:RecordCount` | The total record count of the bound Cursor, or `-1` for an actual WorkArea with no usable Cursor bound. |
| `ListOfOrders:CursorHandle` | A managed Cursor reference to the underlying dataset, or NULL when the WorkArea has no bound Cursor. |

These properties are read-only observations. Reading them does not select the WorkArea, move its record pointer, or copy its records. `:CursorHandle` returns a retained Cursor reference with normal ownership.

### WorkAreaNumber identifies the view

The number belongs to the WorkArea, independently of the local names through which it is accessed. Passing the same view to a helper under a different parameter name preserves its WorkArea number. Within one registry, two independent views over the same Cursor have different WorkArea numbers.

Its scope is the owning runtime registry. A number does not make a local alias globally visible, retain the WorkArea, or grant access from another Task. Numbers can be reused after their registrations are released, so they are useful for inspection and compatible numeric addressing rather than permanent identity or ownership transfer.

### Record position and record count answer different questions

`:RecordNumber` reports the record number in the Cursor's navigation model. It is not the value of an application field such as `Number`, and it is not the number of iterations already completed by a scan. An active index can change visit order without making those concepts equivalent.

Normal row-navigation boundaries distinguish position `0` for an unpositioned or before-first view from `N + 1` after the last record of a Cursor with `N` records. An empty Cursor remains at position `0`. Application code must not treat every position outside an actual record as the same state.

`:RecordCount` reports the underlying Cursor's total record count independently of the view's position or active order. It does not count the rows remaining after the current position or the rows accepted by an arbitrary `SCAN FOR` predicate. Record retention and visibility continue to follow the Cursor's deletion policy.

Two WorkAreas over the same Cursor therefore observe the same dataset while maintaining independent positions. For a published Cursor version, its records and record count remain unchanged. For private editable data, the owner's intervening operations can change the count; separate property reads do not combine several application operations into a transaction.

### NULL, an unbound WorkArea, and an empty Cursor

These states have different meanings:

| State | `:RecordCount` | `:CursorHandle` |
| --- | --- | --- |
| The variable contains NULL | Invalid NULL receiver; check the variable before property access. | Invalid NULL receiver; check the variable before property access. |
| An actual WorkArea value has no usable Cursor bound | `-1`. | NULL. |
| A WorkArea is bound to a valid empty Cursor | `0`. | A valid managed Cursor reference. |
| A WorkArea is bound to a Cursor containing records | The record count. | A valid managed Cursor reference. |

Allowing a variable to contain NULL does not make ordinary `:` access on NULL valid. Invalid NULL-receiver access follows HN's controlled runtime-error path. For a local binding, a status check can account for its initial state explicitly:

```harbour
IF ListOfOrders == NULL
    nRecordCount := -1
ELSE
    nRecordCount := ListOfOrders:RecordCount
ENDIF
```

The `-1` result distinguishes an actual unbound WorkArea from a valid empty dataset. It is not a substitute for every possible execution error. The same distinction applies to a retained WorkArea value whose Cursor binding has been closed or detached under its lifetime rules.

### CursorHandle exposes shared Cursor identity

`:CursorHandle` returns a native value of type `Cursor`. It exposes neither a raw memory address nor a numeric storage handle that application code must manage.

For two bound WorkAreas, the following expression is true when both refer to the same logical Cursor storage:

```harbour
ListOfOrders1:CursorHandle == ListOfOrders2:CursorHandle
```

WorkArea identity and Cursor identity answer separate questions:

| Relationship | WorkArea identity comparison | Cursor identity comparison |
| --- | --- | --- |
| Two references to the same bound WorkArea | True. | True. |
| Two independent WorkAreas over the same Cursor | False. | True. |
| WorkAreas over independent Cursors containing identical rows | False. | False. |

Cursor equality here compares logical storage identity. It does not compare row contents, temporary reference-wrapper addresses, or the physical location of a reallocatable buffer. Separate property reads referring to the same unchanged Cursor therefore preserve equality.

When an actual WorkArea can be unbound, check that a Cursor is present before interpreting equality as shared Cursor identity. Assuming the two WorkArea receivers themselves are non-NULL:

```harbour
LOCAL curFirst Cursor
LOCAL curSecond Cursor
LOCAL lSameCursor

curFirst := ListOfOrders1:CursorHandle
curSecond := ListOfOrders2:CursorHandle

lSameCursor := .F.

IF curFirst != NULL
    lSameCursor := curFirst == curSecond
ENDIF
```

Two NULL values comparing equal do not establish that two views share a real Cursor.

Retaining the returned Cursor keeps its storage alive independently of the original WorkArea. If that view is closed or rebound, a saved Cursor reference still owns the original dataset. Rebinding an existing WorkArea preserves the view's identity while a subsequent `:CursorHandle` access observes its newly bound Cursor.

Most application code can continue using readable WorkArea names. This explicit property supplies the underlying Cursor reference when an operation needs storage identity, ownership, or another view over the same dataset. Obtaining a reference does not grant cross-Task access to private data. An independent editable copy has a different Cursor identity, and publishing a replacement never redirects WorkAreas that retain the original.

## HN logic can participate in SQL expressions

Local SQL integrates with HN functions and object methods. Application-specific formatting, classification, calculations, and validation can participate in query expressions while the relational work remains expressed as SQL.

### Call an application function

An application that defines `CustomerLabel()` can use it to construct a display value:

```sql
SELECT ListOfCustomers.customer_id AS CustomerId,
       CustomerLabel(ListOfCustomers.customer_code,
                     ListOfCustomers.customer_name)
           AS CustomerLabel TYPE Text(MaxLength = 160)
FROM ListOfCustomers
INTO CURSOR
```

The application receives the result through:

```harbour
LOCAL ListOfCustomerLabels WorkArea

nRows := hn_SQLSelect(cSQL, xResult=@ListOfCustomerLabels)
```

The function belongs to the application. The selected alias names its output column, and the optional `TYPE` annotation states that column's contract. Callable lookup follows the executing code's permitted function/module scope; the SQL does not grant access to otherwise hidden application declarations.

### Supply a callable through the binding Hash

A Hash can also supply a function reference under a query-specific name:

```harbour
hBindings := { "customerLabel" => @CustomerLabel }
```

The SQL invokes the supplied function explicitly:

```sql
SELECT ListOfCustomers.customer_id AS CustomerId,
       :customerLabel(ListOfCustomers.customer_code,
                      ListOfCustomers.customer_name)
           AS CustomerLabel TYPE Text(MaxLength = 160)
FROM ListOfCustomers
INTO CURSOR
```

The output is explicit here too:

```harbour
LOCAL ListOfBoundCustomerLabels WorkArea

nRows := hn_SQLSelect(cSQL, ;
    hBindings=hBindings, ;
    xResult=@ListOfBoundCustomerLabels)
```

Here, `:customerLabel(...)` means call the supplied callable with the SQL expression arguments. A plain scalar placeholder such as `:customerId` reads its supplied value. Supplying a function reference does not itself execute the function.

This differs from putting a function call on the right-hand side while constructing the Hash:

```harbour
hBindings := { "customerId" => CurrentCustomerId() }
```

`CurrentCustomerId()` runs when that Hash is constructed, and the Hash stores its result. Reusing the Hash does not automatically call the function again. Under the binding rule, that stored result must be non-NULL.

### Bind the object instance for a method call

An application can supply an object directly:

```harbour
hBindings := { ;
    "customerId" => iCustomerId, ;
    "orderFormatter" => oOrderFormatter ;
}
```

The SQL then uses the bound receiver and the method name:

```sql
SELECT ListOfOrders.order_id AS OrderId,
       :orderFormatter:FormatOrder(ListOfOrders.order_id)
           AS OrderLabel TYPE Text(MaxLength = 160)
FROM ListOfOrders
WHERE ListOfOrders.customer_id = :customerId
INTO CURSOR
```

The application supplies the typed receiving variable:

```harbour
LOCAL ListOfFormattedCustomerOrders WorkArea

nRows := hn_SQLSelect(cSQL, ;
    hBindings=hBindings, ;
    xResult=@ListOfFormattedCustomerOrders)
```

The Hash key is `"orderFormatter"`. `FormatOrder` is the method selected on that object; the application does not need a separate Hash entry for every method. HN's normal method visibility, argument rules, synchronization behavior, and errors continue to apply.

The receiver is retained for the query's lifetime. Retention keeps the object alive without freezing its mutable state. A non-NULL object may contain nullable properties, and a method with a nullable return contract may return NULL.

### Prepare the call once; evaluate it when required

For a fixed bound receiver, the method name is available from the SQL and the receiver's actual class is available when bindings are prepared. The execution design resolves and retains the call information at that stage, allowing repeated evaluations to reuse it. This avoids searching the binding Hash and resolving the same textual method name for every record. Retained call information follows HN's method and module lifetime rules.

The method still runs when SQL evaluates the expression. Caching the resolution does not cache the returned value, and a callback is not invoked early merely to discover its return type. Output-schema description uses available declarations and expression metadata; an explicit output `TYPE` can supply a contract when necessary.

Method calls retain their normal invocation and argument-handling costs. For very small operations evaluated across a large dataset, those costs can matter. This integration makes application logic convenient to reuse; it does not imply that every callback has the cost of a built-in SQL operator or establish a particular execution speed.

For predictable queries, callbacks should have clear input and output contracts. Functions that perform network requests or change application state bring those costs and side effects into query execution. Pure calculations and transformations are easier to compose and reuse.

## Optional `TYPE` properties on output columns

One of HN's useful Local SQL extensions is the ability to describe a selected column more precisely than its name alone.

These declarations belong to the output schema. They are independent of the simple input binding Hash, which needs no `"Type"` entries.

An annotation follows an explicit output alias:

```sql
SELECT ListOfCustomers.customer_id AS CustomerId TYPE Integer NOT NULL,
       ListOfCustomers.customer_code AS CustomerCode
           TYPE Text(ASCII, MaxLength = 12) NOT NULL,
       ListOfCustomers.customer_name AS CustomerName
           TYPE Text(MaxLength = 120, MaxStorageBytes = 480)
FROM ListOfCustomers
INTO CURSOR
```

The receiving binding is supplied by the HN call:

```harbour
LOCAL ListOfValidatedCustomers WorkArea

nRows := hn_SQLSelect(cSQL, xResult=@ListOfValidatedCustomers)
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

A failed query does not assign a partly populated replacement result. The engine makes no result assignment when a recoverable execution failure prevents completion. Explicit callback side effects and concurrent application assignments retain their own effects; failure does not restore an earlier snapshot of the receiving variable.

### Character length and storage length are different

`MaxLength` counts HN TextChars: Unicode extended grapheme clusters. A visible character can consist of several Unicode code points. Combining accents, flags, and joined emoji make the distinction practical.

`MaxStorageBytes` independently limits the number of UTF-8 bytes. A value can satisfy the character limit and still exceed a storage or protocol byte limit.

The two properties let an application express both concerns directly. HN does not infer a byte limit by multiplying a character limit by an assumed number of bytes per character.

`ASCII` specifies the permitted character range. It is not an alphanumeric-only or printable-only validation rule; an application needing those restrictions supplies the appropriate validation logic.

### Most ordinary projections need no annotation

When a query directly selects a field, its declared value metadata follows that field into the result, including applicable type, NULL, and length information.

```sql
SELECT ListOfCustomers.customer_name AS CustomerName
FROM ListOfCustomers
INTO CURSOR
```

The result is received with:

```harbour
LOCAL ListOfCustomerNames WorkArea

nRows := hn_SQLSelect(cSQL, xResult=@ListOfCustomerNames)
```

Renaming the field does not make it lose its type contract. This remains true when the query returns no rows, or when every selected value is NULL. The schema comes from declarations rather than from inspecting the first row or measuring the longest value currently present.

Computed outputs can need more information. A callable may have an unspecified return contract, and a bare NULL does not identify an output column's base type. An output expression such as `NULL AS Note TYPE Text` supplies that missing contract explicitly. Rejecting NULL input binding entries does not remove this separate output-schema question.

An explicit `TYPE` supplies a replacement output contract. Omitted limits therefore do not silently inherit every source constraint. Use an ordinary direct projection when the complete source value descriptor is the desired result.

The engine also accounts for the query's meaning. An outer join can make a result column nullable. A join can repeat a source primary key, so copying its value type does not imply that the result column is unique. Source defaults, key constraints, and value generators do not automatically become result-column guarantees.

### Describe the schema before consuming records

`hn_SQLDescribe()` derives the output schema without scanning records or executing HN functions and methods from the query. `hn_CursorDescribe()` provides metadata for an existing Cursor or WorkArea.

`hn_SQLDescribe()` does not receive or populate `xResult`. The output-reference requirement belongs to execution with `hn_SQLSelect()`; description creates no result WorkArea, assigns no result, and does not select or reposition a view.

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

**Current SQL boundary:** The native Number type is implemented in the language, while its full Local SQL integration remains separate work. The current SQL evaluator supports its documented Integer/Float subset, uses Float for division, and does not yet support `SUM` or `AVG`. The broader type model described here is the integration goal; applications should use the operators and result types supported by the SQL engine they are running.

### NULL and time retain their meaning

Local SQL preserves NULL values retrieved from a backend, produced by an outer join, or returned by a nullable expression. Its three-valued logic allows a comparison involving NULL to be unknown; `WHERE` and `HAVING` retain rows for which the condition is true. Use `IS NULL` or `IS NOT NULL` to test for missing values, rather than `= NULL` or `<> NULL`. Preserving NULL prevents an unknown value from being mistaken for an actual zero, false, or empty Text value. See [PostgreSQL's comparison semantics](https://www.postgresql.org/docs/18/functions-comparison.html).

This SQL behavior is distinct from the HN1 rule that input binding entries themselves must be non-NULL. A nullable source column retains its declared type even when every stored value is NULL, and a named result WorkArea retains its schema when no rows match.

Temporal fidelity likewise requires explicit distinctions. A date is different from a timestamp, and a local timestamp differs from an instant interpreted with time-zone rules. Backend mappings must preserve the information actually supplied by the source. For example, PostgreSQL `timestamptz` does not retain the originally entered time-zone name; an adapter cannot reconstruct metadata the server no longer holds. See [PostgreSQL's date/time types](https://www.postgresql.org/docs/18/datatype-datetime.html).

### Broad parity keeps backend differences visible

Backend systems still have their own domains, collations, specialized types, extensions, and representation rules. JSON is a useful example: PostgreSQL `json` and `jsonb` preserve different aspects of the original representation. See [PostgreSQL's JSON documentation](https://www.postgresql.org/docs/18/datatype-json.html).

HN's advantage is a rich common model with explicit mappings. The application can retain the meaning needed for local work and deliberate writeback, while handling backend-specific details where they belong.

## Jobs and Tasks: acquire independent datasets concurrently

Many useful local queries depend on several independent backend requests. Jobs and Tasks let HN overlap that acquisition work. A Job coordinates a group of Tasks; each Task performs its own operation and has its own completion or error outcome. Concurrency limits keep the number of active requests appropriate for the application and its servers.

Consider a customer-service report:

| Task | Example source | WorkArea name in the consuming context |
| --- | --- | --- |
| Retrieve customer details | PostgreSQL CRM database | `ListOfCustomers` |
| Retrieve orders for the reporting period | SQL Server order system | `ListOfOrders` |
| Retrieve open support tickets | MariaDB service system | `ListOfTickets` |

These are illustrative source choices, using the application's supported backend adapters. The local relational work is independent of which backend supplied each dataset.

The HN1 sharing model gives this workflow a clear sequence:

1. Each Task retrieves and prepares its dataset privately.
2. A completed dataset is published read-only before other executions read it.
3. The Job coordinates completion and handles failures.
4. Main, or another consuming Task, receives the published Cursor references and establishes its own WorkAreas and local source names.
5. Local SQL reads those published datasets and produces an independent result for the report or response.

Several consumers can read the same published datasets. Each uses its own WorkAreas, so their record positions and active-order choices remain independent. A Task that needs to change a published dataset works on a new private copy; it cannot change records that another reader is using.

Materializing results separates the local data from the backend connections used to fetch it. Publishing a Cursor does not share a connection or transaction. It also does not create an atomic snapshot across separate database servers; reports that need a common cutoff must establish the appropriate source-query or transaction rules.

Invocation-local and TASK STATIC bindings stay in their respective scopes. The consuming execution establishes its own names after receiving access to published data. A typed WorkArea parameter can supply a local source name within its permitted execution. GLOBAL visibility alone does not publish a Cursor or supply another reader's WorkArea.

Once those source bindings exist, SQL uses their names directly, and `xResult` identifies the result destination. An independently created query result can be retained by its caller and explicitly published for other readers when appropriate.

When requests are independent and resources permit, acquisition can approach the duration of the slowest request plus coordination and materialization overhead. Actual performance depends on backend load, connection limits, transferred data, and the work each request performs.

## Combining the results without multiplying records

Once the three datasets are available through the consuming context's WorkAreas, the application can build a local processing pipeline.

The customer/order query shown earlier creates `ListOfCustomerOrderCounts`. A second query produces one ticket summary per customer:

```sql
SELECT ListOfTickets.customer_id AS CustomerId,
       COUNT(*) AS OpenTicketCount
FROM ListOfTickets
WHERE ListOfTickets.status = :openStatus
GROUP BY ListOfTickets.customer_id
INTO CURSOR
```

The application supplies the parameter and output:

```harbour
LOCAL ListOfTicketCounts WorkArea

hBindings := { "openStatus" => "Open" }

nRows := hn_SQLSelect(cSQL, ;
    hBindings=hBindings, ;
    xResult=@ListOfTicketCounts)
```

This assumes `"Open"` is the status value used by that source. The native Text value supplies its type. After both summaries succeed, the next query reads their WorkArea bindings directly:

```sql
SELECT ListOfCustomerOrderCounts.CustomerId,
       ListOfCustomerOrderCounts.CustomerName,
       ListOfCustomerOrderCounts.OrderCount,
       ListOfTicketCounts.OpenTicketCount
FROM ListOfCustomerOrderCounts
LEFT JOIN ListOfTicketCounts
    ON ListOfTicketCounts.CustomerId = ListOfCustomerOrderCounts.CustomerId
ORDER BY ListOfCustomerOrderCounts.OrderCount DESC,
         ListOfCustomerOrderCounts.CustomerId
INTO CURSOR
```

Its receiving variable is:

```harbour
LOCAL ListOfCustomerServiceSummary WorkArea

nRows := hn_SQLSelect(cSQL, xResult=@ListOfCustomerServiceSummary)
```

Each summary has at most one row per customer, so their join preserves that intended result shape. A customer with no matching ticket summary has NULL in `OpenTicketCount`; the presentation or an explicit application transformation can supply zero when that is the required report convention.

This staged approach also avoids a common reporting mistake. Joining a customer's four orders directly with their three tickets produces twelve matching combinations. Counting after that join would count the combinations unless the query accounts for them. Aggregating each relationship first makes the intended cardinality clear.

Every intermediate result remains a normal named WorkArea over local Cursor data. It can be inspected during debugging, reused for another report, or queried again without repeating the original backend requests. Declaring the intermediate targets as local WorkAreas gives the pipeline scoped ownership; returning, capturing, or otherwise retaining a result lets that result outlive the function.

## Results remain useful to ordinary HN code

The SQL Text selects the result representation. Every result mode supplies its value through an actual HN receiving variable passed as `xResult=@destination`.

| Result mode | Application result | Output-reference argument | Successful execution return |
| --- | --- | --- | --- |
| `INTO CURSOR` | A WorkArea over a newly materialized Cursor, including a valid schema-bearing empty result. | Required; the destination must accept a WorkArea. | Number of result rows. |
| `INTO OBJECT` | One read-only system DataObject; NULL for no rows; an error if more than one final row is produced. | Required; the destination must accept the object/NULL result. | `0` or `1`. |
| `INTO OBJECTS ARRAY` | An Array of read-only DataObjects, one per row; an empty Array for no rows. | Required; the destination must accept the Array. | Number of result rows. |

The exact Array-mode spelling is `INTO OBJECTS ARRAY`. A recoverable execution error returns `-1` through the established structured-error path.

Read-only DataObjects describe the current result implementation. The approved future direction allows validated editing of typed, NULL-aware pseudo-properties with explicit backend persistence. Row identity, mappings, change tracking, transactions, conflicts, and joined or computed fields still need their detailed update contracts. Assigning a property will not silently update a backend.

### WorkArea results

A WorkArea result is a natural choice when subsequent work needs schema inspection, record navigation, indexes, or another Local SQL operation. Its declared HN binding supplies a source name after the successful output assignment. Application code can also pass or return the WorkArea as an ordinary native value.

`hn_SQLSelect()` returns the row count and assigns the WorkArea through `xResult`. It does not return a raw Cursor storage handle. A separate `:CursorHandle` access obtains the managed Cursor reference when needed.

All output assignments follow the receiving variable's ordinary type, ownership, and storage rules. Successful assignment replaces its current reference, and other owners of the previous view retain their view. A result assigned to a local, retained, or global destination has the lifetime of its actual owners. There is no implicit SQL-created wider-scope name.

The output is assigned after successful execution and materialization. The engine does not expose a partly built replacement. Independent application side effects remain observable, and one output assignment does not turn the complete query or several shared variables into one transaction. Closing an input WorkArea does not invalidate the independent result. Creating the result does not automatically select it or publish its Cursor for shared reading.

### Reusing the same variable as a source and destination

A query can read the existing WorkArea and replace its receiving binding after successful materialization:

```harbour
nRows := hn_SQLSelect( ;
    "SELECT * FROM ListOfOrders " + ;
    "WHERE ListOfOrders.customer_id IS NOT NULL INTO CURSOR", ;
    xResult=@ListOfOrders)
```

The executor resolves and retains the source before processing, creates an independent result, and assigns the new WorkArea only after success. It does not clear or rebind the source view before reading it. Other owners of the old WorkArea keep that view.

This protects source identity and lifetime. A published input Cursor remains unchanged; replacing this receiving variable does not switch other readers to the new result. Private source data remains subject to its owner's permitted operations and the query's access guards.

### An Array of row objects

For code that wants a collection of row objects, SQL selects `INTO OBJECTS ARRAY` and the call supplies the receiving variable:

```harbour
cSQL := "SELECT ListOfOrders.order_id AS OrderId, " + ;
        "ListOfOrders.customer_id AS CustomerId " + ;
        "FROM ListOfOrders " + ;
        "WHERE ListOfOrders.customer_id = :customerId " + ;
        "INTO OBJECTS ARRAY"

hBindings := { "customerId" => iCustomerId }

aCustomerOrders := NULL

nRows := hn_SQLSelect(cSQL, ;
    hBindings=hBindings, ;
    xResult=@aCustomerOrders)

IF nRows < 0
    errSQL := hn_SQLLastError()
    // Application-specific error handling uses errSQL.
ENDIF
```

On success, `aCustomerOrders` contains one DataObject per result row. Its properties use the selected column names or aliases, here `OrderId` and `CustomerId`, and retain supported scalar types and NULL states. Zero rows produces an empty Array. A recoverable failure prevents assignment of a partial engine result.

The SQL string specifies the representation; `@aCustomerOrders` identifies the actual HN receiving variable. This is the same destination mechanism as Cursor mode. Ordinary scalar, object, and Array variable names are not discovered by looking for their spelling in SQL Text.

`INTO OBJECT` follows the same reference-destination pattern for a single logical row. Its zero-row result is NULL, which is valid output and independent of the non-NULL input-binding rule.

## HNConvert: preserving behavior while making scope explicit

HL and VFP applications often depend on aliases whose lifetimes extend beyond one routine. Conversion must preserve that intended ownership and visibility while making the HN declaration explicit.

### Cursor destinations

A statically named legacy destination such as:

```sql
-- Legacy source form for conversion, not native HN output syntax.
SELECT ... INTO CURSOR ListOfOrders
```

maps to SQL ending in `INTO CURSOR` and an HN call supplying `xResult=@ListOfOrders`. The converter creates or uses an appropriate typed WorkArea declaration. A temporary local result can use LOCAL; a result retained across application routines can require a retained or GLOBAL binding. These declarations govern names and lifetime. Sharing the data across Main and Tasks additionally requires read-only publication and separate reader WorkAreas.

A converter must not turn every legacy alias into a short-lived LOCAL. It must also detect duplicate global definitions and narrower declarations that would hide a visible GLOBAL. Distributed declarations are collected for the whole selected application before these checks are finalized.

Runtime-generated alias names, macro expressions, and arbitrary SQL Text need explicit dynamic-name handling or a clear conversion diagnostic. A Text value cannot automatically become a statically resolved HN variable. Passing a variable reference solves the static destination case without recreating dynamic PUBLIC or PRIVATE variables.

### Native property names and command counts

| Legacy intent | HN expression or treatment |
| --- | --- |
| Obtain a WorkArea/Cursor record count | Use the selected view's `:RecordCount`, with the required receiver-state checks. |
| Obtain its current record number | Use `:RecordNumber`. |
| Obtain the result-row count of a Local SQL query | Use the Integer returned by `hn_SQLSelect()`. |
| Read VFP `_TALLY` | Preserve the last relevant command's processed/affected/result count; translate according to the command and its lifetime. |

VFP `_TALLY` is not a universal synonym for a dataset's current record count. Its command-count role is documented in the [VFP SET TALK reference](https://vfphelp.com/vfp9/_5WN12PFMO.htm) and [SELECT reference](https://www.vfphelp.com/help/_5wn12penw.htm). For a successful SELECT, the return value from `hn_SQLSelect()` supplies the result-row count. Other commands require their own corresponding count contract.

The native property names are `RecordCount` and `RecordNumber`. Changing these property names does not rename data fields, SQL output aliases, or unrelated Text values in the original application.

### Global scope and module boundaries

An application GLOBAL remains part of the application owner. Conversion must not make it ambiently visible in a library or HNB module. Where legacy code relies on that coupling, the converted interface passes the required WorkArea, data, callback, or supported output reference explicitly.

Similarly, passing an alias spelling or WorkArea number does not transfer a dataset to another Task. Converted code must preserve exclusive access to private editable data and explicitly publish completed data for shared reading. Each reader establishes its own WorkArea; a global declaration alone does not provide that view or access.

## How this differs from approaches in other languages

Local querying is a well-established idea, and several ecosystems provide strong solutions. The useful comparison is how each approach fits into the surrounding application.

| Approach | How it relates to HN Local SQL |
| --- | --- |
| Handwritten loops and lookup structures | Give direct procedural control. HN also offers SQL for expressing relational selection, joins, grouping, and ordering over typed datasets. |
| C# LINQ | Provides language-integrated queries over collections. HN uses SQL Text over its native Cursor model, including explicit output schemas and HN-specific column contracts. |
| SQLite embedded in an application | Provides an in-process SQL database. HN's Local SQL operates within HN's own Cursor, value, and application-call model. |
| DuckDB with DataFrames or other local sources | Provides SQL over local data, including direct DataFrame integration. HN concentrates on integration with HN Cursors, native values, and application workflows. |

These alternatives are substantive capabilities: [LINQ queries collections](https://learn.microsoft.com/en-us/dotnet/csharp/linq/get-started/introduction-to-linq-queries), [SQLite runs in-process](https://www.sqlite.org/about.html), and [DuckDB can query Pandas DataFrames directly](https://duckdb.org/docs/2.0/guides/python/sql_on_pandas). External engines and libraries may also provide efficient integration without copying every input value.

HN's distinguishing benefit is the consistency of the complete workflow. Backend results become native typed data. SQL consumes that data. HN logic can participate in expressions. Result schemas remain available to subsequent processing. Local WorkArea bindings support reusable functions and predictable ownership, while `SELECT`, `SCAN`, and explicit field access remain available for procedural work. Jobs and Tasks coordinate the acquisition of independent inputs.

For developers familiar with the xBase tradition of working with local datasets, this retains a productive style of application development while extending it with a richer type model, explicit ownership, Unicode-aware Text contracts, and concurrent work.

## Efficient storage for NULLs and repeated values

Cursor storage will be optimized for datasets containing many NULL cells or repeated values. Examples include wide records with many optional fields and large result sets that repeatedly contain the same status, country, or category. These optimizations can reduce memory use where the data permits.

The application continues to see the same records, fields, exact values, and NULL states. NULL remains distinct from zero and empty Text. Developers use the same WorkAreas and SQL; they do not need to manage the internal storage representation. The amount of memory saved depends on the dataset.

## Where the practical advantage appears

Local SQL is especially useful when the application needs a combined view that no single backend conveniently owns.

Examples include comparing records across systems, combining independently retrieved datasets, producing several reports from one retrieval, enriching server data with local information, and preparing a web response from several completed Tasks.

The application can keep its stages explicit: retrieve the useful source data, preserve its types in Cursors, apply local relational transformations, and pass the result to ordinary HN code. Each stage has a clear purpose and an inspectable result.

Good backend queries still matter. Filtering, projecting, and aggregating at the source can reduce the amount of data that must cross the network. Local SQL then handles the work that benefits from the application's broader context. Local Cursor changes and results are persisted to a backend through an explicit write operation.

The practical result is less repetitive data-handling code, reusable query results, clearer schemas, and fewer avoidable conversions between the database and the application. Jobs and Tasks can prepare datasets privately in parallel, then share completed read-only data with Main and other Tasks. Independent WorkAreas keep each reader's navigation separate while Local SQL and ordinary HN code make use of the same typed data.

