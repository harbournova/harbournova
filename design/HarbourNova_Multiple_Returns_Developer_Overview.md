# Multiple return values in HarbourNova: returning related results together

**Developer overview of HN1 multiple returns and matching-target assignment — October 31, 2026**

> **Scope:** This article describes the implemented HN1 feature for functions, methods, and APIs with an explicit multiple-result contract. Examples use application-defined helpers, existing HN APIs, and one explicitly identified proposed API.

An application operation often produces more than one useful answer. Allocating stock produces an allocated quantity and a backordered quantity. A lookup can produce a found/not-found indication together with the requested information. A validation operation can produce a usable value or structured information explaining why it could not produce one.

HarbourNova lets a function or method declare these related results directly. The caller receives them through **matching-target assignment**:

```harbour
iAllocated, iBackordered := SplitStock(12, 7)
```

The first returned value goes to `iAllocated`; the second goes to `iBackordered`. The function runs once. Its declaration specifies the number, order, and type contracts of the results, and the assignment must account for every position.

The feature is useful when several values form one operation's answer but the application does not need a separate object or collection to hold that answer. Each result remains an ordinary HN value, with its own type and assignment rules.

## Choosing how an operation returns its answer

Multiple returns complement the other ways an HN API can communicate results:

| Approach | When it fits | Tradeoffs for the caller and API designer |
| --- | --- | --- |
| **Multiple return values** | A small, fixed set of related outputs that callers normally use separately. | The declaration exposes the ordered result contracts. Callers must account for every position, including unwanted results. Changing the number or order changes the interface. |
| **One Array** | A sequence, a variable number of items, or an answer that should be stored and passed around as one collection. | The Array is one result. Its element count does not become the function's result count, and matching-target assignment does not unpack it. |
| **One object** | An answer with meaningful field names, behavior, or a useful identity of its own. | The object packages the answer for later use. Its members provide names instead of requiring the caller to remember result positions. |
| **Output/reference parameters** | An interface deliberately modifies caller-owned variables, possibly combining input and output behavior. | References make that mutation part of the input contract. The caller supplies writable destinations, and the API defines when those destinations change. |

An object is often clearer for a large or evolving result. An Array remains appropriate for a list of records or measurements. Multiple returns work particularly well for a stable pair or small group of values consumed immediately after a call.

The benefit is a direct interface and explicit checks. It is not a promise that multiple returns always run faster, allocate less memory, or avoid all internal allocation. Those properties depend on the implementation and the values involved.

## A complete first example

Consider a helper that divides a requested quantity into the amount available now and the amount to backorder. This example assumes non-NULL, nonnegative quantities; its purpose is to demonstrate the result contract.

```harbour
FUNCTION Main()
    LOCAL iAllocated Integer
    LOCAL iBackordered Integer

    iAllocated, iBackordered := SplitStock(12, 7)

    ? iAllocated       // 7
    ? iBackordered     // 5

    RETURN NULL

FUNCTION SplitStock(iRequested Integer Required, ;
                    iAvailable Integer Required) ;
    RETURNS (Integer, Integer)

    IF iRequested <= iAvailable
        RETURN iRequested, 0
    ENDIF

    RETURN iAvailable, iRequested - iAvailable
```

`RETURNS (Integer, Integer)` declares exactly two result positions. Both normal return paths supply two expressions in the same order:

| Position | Meaning | Receiving variable in this call |
| --- | --- | --- |
| 1 | Quantity allocated from available stock. | `iAllocated` |
| 2 | Quantity still required. | `iBackordered` |

The variable names help readers understand the application, but the matching itself is positional. Calling the variables `iNow` and `iLater` would not change which result goes where. Naming the first receiver `iBackordered` would not make HN reorder the results.

`Required` means that the caller must supply the argument. It does not, by itself, express the example's non-NULL or nonnegative preconditions. Parameter contracts and application validation remain separate concerns.

The semicolons in these examples continue an HN statement onto the next line. They are not statement terminators.

## Declaring a result contract

### One result or several

An ordinary function without a result annotation retains its single-result behavior. An explicitly typed single result uses `RETURNS Type`. Two or more results use a parenthesized list:

| Declaration form | Meaning |
| --- | --- |
| `FUNCTION Name(...)` | One ordinary result, without an explicit result datatype restriction. |
| `FUNCTION Name(...) RETURNS Any` | One result that may contain any supported HN value type. |
| `FUNCTION Name(...) RETURNS Text` | One result checked against the Text result contract. |
| `FUNCTION Name(...) RETURNS (Integer, Text)` | Two ordered results with separate contracts. |

HN1 supports **2 through 256 positions** in a multiple-result declaration. The upper limit is a language implementation limit, not a recommendation for API design.

The canonical single-result form has no surrounding result-list parentheses: write `RETURNS Text`, not `RETURNS (Text)`. An empty result list is also invalid. This feature introduces neither a zero-result tuple nor a new void-return form.

Each position can use the supported HN type constraints. For example, `Number(7,3)` describes one result even though its own type specification contains a comma. A declaration can combine constrained Number, constrained Text, Array, Error, or other supported types in different positions.

Result declarations contain types and constraints. They do not declare named result variables, supply result defaults, or return references to the callee's local variable slots.

### Any remains supported

**`Any` remains a supported datatype for both single-result and multiple-result declarations.** Use it when a result's datatype legitimately varies between calls. It can appear in one, several, or every result position, and can be mixed with more specific types in the same declaration.

For example, this application-defined helper returns either an Integer or Text in its first position, followed by a NULL Error value:

```harbour
FUNCTION ReadApplicationValue(lUseText Boolean Required) RETURNS (Any, Error)
    IF lUseText
        RETURN "ready", NULL
    ENDIF

    RETURN 42, NULL

FUNCTION DemonstrateAnyResults()
    LOCAL xValue Any
    LOCAL errValue Error

    xValue, errValue := ReadApplicationValue(FALSE)
    // xValue contains Integer 42; errValue is NULL.

    xValue, errValue := ReadApplicationValue(TRUE)
    // xValue now contains Text "ready"; errValue is NULL.

    RETURN NULL
```

The `Any` receiving variable preserves the actual returned value and its datatype. An `Any` result may also be NULL. Each call still returns exactly two values: `Any` describes the permitted value in one position, not a variable number of results or an automatically packed result list.

A typed receiving variable still enforces its own contract. If the first destination were declared `Integer`, receiving the Text outcome would fail the matching-target assignment before either result was written. Declaring a result as `Any` does not disable destination checks.

### The declaration is required

The function body does not implicitly define a multiple-result interface. This declaration is invalid:

```harbour
FUNCTION InvalidPair()
    RETURN 10, "ready"
```

It needs an explicit `RETURNS (Integer, Text)` clause. The same requirement applies to a wrapper that forwards several results from another function. An unannotated wrapper does not acquire a multiple-result signature from its body.

### NULL is a supplied result

NULL occupies a result position just as another value does. It neither removes that position nor shifts later results to the left.

In HN1, source-level `RETURNS` annotations admit NULL. For example, `RETURNS (Integer, Text)` declares two result positions, either of which may contain NULL. Each receiving variable still applies its own type and value constraints.

Here is a small lookup example. The helper recognizes one warehouse so that both outcomes are visible without requiring a database or another API:

```harbour
FUNCTION DemonstrateWarehouseLookup()
    LOCAL lFound Boolean
    LOCAL iAvailable Any
    LOCAL cWarehouse Text

    lFound, iAvailable, cWarehouse := FindWarehouseStock(1)
    // TRUE, 7, "Main warehouse"

    lFound, iAvailable, cWarehouse := FindWarehouseStock(99)
    // FALSE, NULL, NULL

    RETURN NULL

FUNCTION FindWarehouseStock(iWarehouseId Integer Required) ;
    RETURNS (Boolean, Integer, Text)

    IF iWarehouseId == 1
        RETURN TRUE, 7, "Main warehouse"
    ENDIF

    RETURN FALSE, NULL, NULL
```

The unsuccessful lookup still returns three values. The Boolean distinguishes an unsuccessful lookup from a successful lookup whose quantity happens to be zero. That meaning belongs to this helper's documented convention; HN does not automatically interpret the first result as a success flag.

The example uses an `Any` receiving variable so it can hold either the Integer quantity or NULL. The function's second result remains declared as Integer. Choosing an `Any` destination does not change that result declaration or convert the returned value.

Native APIs retain their documented result contracts. Multiple returns do not relax those contracts, including when a caller discards a result.

### Every normal return supplies the complete shape

Every normal completion of a multiple-result function must supply its declared number of results. Different branches can return different values, but not different result counts.

For a two-result function:

| Return form | Meaning |
| --- | --- |
| `RETURN 7, 5` | Two supplied values. |
| `RETURN NULL, NULL` | Two supplied values, subject to the applicable contracts. |
| `RETURN NULL` | One supplied value; it does not fill two positions. |
| Bare `RETURN` | Does not supply the declared result list. |
| Reaching the end without a complete return | Does not supply the declared result list. |

Provable declaration and count violations are rejected during compilation. Execution still checks completion paths that cannot be proved statically, including an unproved conditional fallthrough. Missing results are never padded with NULL, and excess results are never silently dropped.

An exception is not a normal return. It follows HN's exception-handling rules and does not need to manufacture a result list.

Ordinary single-result functions and methods retain their established bare-return and fallthrough behavior: they produce one NULL value through the normal return and cleanup path. The complete-list requirement applies to a declared multiple-result interface.

## Receiving every position

### Exact count, explicit discards

Declare the receiving variables normally, then write a comma-separated list followed by `:=` and one top-level producer call. There are no parentheses around the receiving list and no `@` markers on its variables.

The right-hand side must be that call. This is invalid:

```harbour
iAllocated, iBackordered := 7, 5
```

Matching-target assignment does not introduce general parallel assignment of arbitrary expressions.

Using the earlier `SplitStock()` function and already declared Integer variables:

| Call or assignment | Behavior |
| --- | --- |
| `iAllocated, iBackordered := SplitStock(12, 7)` | Receives both values. |
| `iAllocated, _ := SplitStock(12, 7)` | Receives 7 and discards the second value. |
| `_, iBackordered := SplitStock(12, 7)` | Discards the first value and receives 5. |
| `_, _ := SplitStock(12, 7)` | Explicitly discards both positions. |
| `SplitStock(12, 7)` | A standalone call; discards the complete result list. |
| `iAllocated := SplitStock(12, 7)` | Invalid: one target cannot receive two results. |
| `iAllocated, iBackordered, iExtra := SplitStock(12, 7)` | Invalid: three targets cannot receive two results. |

A discard counts as a position. An empty position or an omitted trailing target is not a way to shorten the receiving list.

The exact identifier `_` is reserved for this purpose. It needs no declaration, stores no value, and can appear more than once because it has no variable identity. It cannot be declared or read as an ordinary variable, including as a catch binding. Longer identifiers such as `_temporary` and `customer_id` remain ordinary names; the Text value `"_"` remains ordinary data.

This feature does not add a standalone `_ := expression` statement. Use a standalone call to ignore all of a call's results.

**Discarding does not skip computation.** The producer still executes, every result expression is evaluated as required, and every returned position is checked against its result contract. An error while producing or validating a discarded result still fails the call.

### Supported destinations

Each non-discard target must resolve to a distinct, writable ordinary HN variable. Normal visibility, ownership, and reference rules still apply.

| Destination | Supported by matching-target assignment? |
| --- | --- |
| A declared local variable or parameter. | Yes, when writable. |
| An accessible ordinary STATIC or GLOBAL variable. | Yes, under its normal scope and access rules. |
| A retained ordinary variable, including one captured by an existing closure. | Yes, while it remains a valid writable variable. |
| A reference parameter whose ultimate destination is an ordinary variable. | Yes; the ultimate variable's contract still applies. |
| An object property or setter, such as `oStock:Available`. | No. |
| An indexed Array or Hash destination, such as `aStock[1]`. | No. |
| A WorkArea field, such as `ListOfStock->Quantity`. | No. |
| A nested or starred receiving pattern. | No. |

Passing a reference to an indexed entry, property, or field through a parameter does not make that destination eligible. The rule follows the actual receiving variable, not just the parameter's name.

If a result must eventually reach a property or indexed entry, first receive it into ordinary variables, then perform the required assignments explicitly. Those later assignments have their own validation and failure behavior; they do not become part of the original matching-target assignment's guarantee.

Combined declaration-and-receiving syntax is not introduced. Keep declarations separate from the receiving statement.

### Different names must identify different variables

This receiving statement is invalid:

```harbour
iAllocated, iAllocated := SplitStock(12, 7)
```

The rule also covers different names that alias the same underlying variable. For example:

```harbour
FUNCTION DemonstrateAliasedTargets()
    LOCAL iQuantity Integer := 100

    ReceiveStock(@iQuantity, @iQuantity)
    // The receiving assignment inside ReceiveStock fails.

    RETURN NULL

FUNCTION ReceiveStock(@iAllocated Required, @iBackordered Required)
    iAllocated, iBackordered := SplitStock(12, 7)
    RETURN NULL
```

The reference parameters have different names, but both identify `iQuantity`. The inner matching-target assignment rejects the duplication before writing either result. Without a surrounding handler, that error propagates to the caller.

Known repeated names can be rejected during compilation. Aliases that become apparent only during execution are checked then. A runtime alias error does not promise that the producer has not already executed.

Two distinct variables may nevertheless receive the same value or a reference to the same object. **The restriction concerns destination-variable identity, not equality or identity of the returned values.**

## Calls in ordinary expressions

A multiple-return call does not automatically become an Array, a tuple object, or a scalar value. HN does not silently choose its first result.

Using `SplitStock()`:

| Context | Result |
| --- | --- |
| Matching-target assignment | Receives all positions, with optional explicit discards. |
| Standalone call | Ignores all results after normal production and validation. |
| Single-target assignment | Rejected for a multiple-result call, even if the target is untyped or declared Any. |
| Arithmetic or another scalar expression | Rejected; the call supplies several results where one value is required. |
| One argument of another call | Does not expand into several arguments; a multiple-result value is invalid in that scalar argument position. |
| Complete forwarding from a declared multiple-result wrapper | Supported, subject to the wrapper's result contracts. |

For example, neither of these expressions selects an allocated quantity:

```harbour
iTotal := 10 + SplitStock(12, 7)
SplitStock(SplitStock(12, 7), 20)
```

If a scalar expression needs just one answer, make that choice explicit. This complete wrapper uses the earlier helper:

```harbour
FUNCTION AllocatedQuantity(iRequested Integer Required, ;
                           iAvailable Integer Required) RETURNS Integer
    LOCAL iAllocated Integer

    iAllocated, _ := SplitStock(iRequested, iAvailable)
    RETURN iAllocated

FUNCTION DemonstrateScalarWrapper()
    LOCAL iCombined Integer

    iCombined := 10 + AllocatedQuantity(12, 7)
    // iCombined is 17.

    RETURN NULL
```

The wrapper returns one value, so it can participate in ordinary expressions. It still requests and validates the underlying function's complete result list.

### Returning an Array remains a single return

```harbour
FUNCTION GetStockPairArray() RETURNS Array
    RETURN { 7, 5 }

FUNCTION DemonstrateArrayResult()
    LOCAL aQuantities Array

    aQuantities := GetStockPairArray()
    // One Array containing two elements.

    RETURN NULL
```

`iAllocated, iBackordered := GetStockPairArray()` would be a result-count mismatch. The Array's two elements are not two callable results. An Array may also occupy one position in a multiple-result contract, such as `(Array, Integer)`.

The same distinction applies to an object or WorkArea returned as one value. The number of fields, members, or records inside it does not determine the function's result count.

## Evaluation, checks, and assignment failures

### What runs, and in what order

For a normal matching-target assignment, the observable sequence is:

1. **Evaluate the call's arguments once, in source order.** Named arguments keep their written evaluation order even when that order differs from the parameter declaration. A failing argument prevents later arguments and the producer body from running.
2. **Execute the producer once.** In an explicit `RETURN expression1, expression2, ...`, evaluate the result expressions once, from left to right. An exception while evaluating one expression prevents later expressions from running.
3. **Complete the callee's return and cleanup rules.** Values are captured before required `FINALLY` cleanup. Successful cleanup completes before final result-contract validation and delivery. Nested calls during cleanup do not replace the pending outer results.
4. **Validate the receiving assignment.** Check the complete result shape, destination validity and distinctness, and each destination's type and constraints. Prepare all permitted conversions before writing any results.
5. **Write the prepared results.** Cleanup associated with replaced values happens after all receiving writes have completed.

Compile-time checks can reject invalid code before any execution. The sequence above describes the execution obligations that remain; it does not require every possible check to be postponed until runtime.

Result-expression evaluation and result-contract validation are separate stages. A value can be computed successfully but later fail its declared return contract. Do not rely on that contract failure occurring before subsequent result expressions or required `FINALLY` code run.

A pending return is also not a snapshot of every reachable object. Reassigning a local variable during `FINALLY` does not replace an already captured value. Mutating an object referred to by that value remains an ordinary mutation of the shared object.

### Result contracts and receiving contracts are different checks

The function's result declaration says what it may deliver. A receiving variable says what that variable may store. Both matter.

This example deliberately chooses a destination that is too short for the returned label:

```harbour
FUNCTION ReadStockStatus() RETURNS (Integer, Text)
    RETURN 7, "available"

FUNCTION DemonstrateReceivingFailure()
    LOCAL iAvailable Integer := 99
    LOCAL cStatus Text(MaxLength=4) := "old"
    LOCAL lCaught Boolean := FALSE

    TRY
        iAvailable, cStatus := ReadStockStatus()
    CATCH errReceiving HNError
        lCaught := TRUE
    ENDTRY

    // lCaught is TRUE.
    // iAvailable is still 99, and cStatus is still "old".

    RETURN NULL
```

The producer's Text result contract accepts `"available"`. The receiving variable's four-character limit does not. The assignment therefore writes neither result: it does not update `iAvailable` and then discover that `cStatus` is invalid.

Passing a typed destination through an untyped reference parameter does not remove its real constraints. An Any result likewise cannot bypass a typed destination's checks.

### Existing conversions still apply

Multiple returns use HN's established type-specific conversion and constraint rules. They do not create general Text parsing, silent truncation, or a new numeric-conversion policy.

For example, an exact Number result can undergo the normal declared-scale rounding when received into a Number variable with a smaller scale:

```harbour
FUNCTION QuotedAmount() RETURNS (Number(7,3), Text)
    RETURN hn_Number(1.235), "quoted"

FUNCTION DemonstrateNumberReceiving()
    LOCAL nAmount Number(5,2)
    LOCAL cStatus Text

    nAmount, cStatus := QuotedAmount()
    // nAmount is the exact Number 1.24; cStatus is "quoted".

    RETURN NULL
```

Here `hn_Number()` is HN's existing Number constructor. The producer accepts the value at its declared scale of three; the receiving variable applies its scale of two. A value that cannot satisfy an applicable numeric range or precision constraint instead causes a checked failure. All receiving conversions must succeed before any destination receives a result.

### The precise failure guarantee

**If the call cannot deliver valid results, or a required check fails before receiving writes, the matching-target assignment writes none of its results.**

This is a guarantee about those receiving writes. It does not restore a snapshot of the application before the statement started.

| Failure point | Effect on the receiving result writes | Other effects |
| --- | --- | --- |
| An argument expression fails. | None occur. | Earlier argument effects remain; later arguments and the producer do not run. |
| The producer or a return expression raises an exception. | None occur. | Earlier effects remain, and normal exception cleanup applies. |
| Required callee cleanup fails. | None occur. | Cleanup effects remain; the call has not delivered normal results. |
| A result or destination contract, count, or alias check fails before assignment. | None occur. | Producer effects and independent callback writes remain. |
| Cleanup of replaced values fails after assignment. | The receiving writes have already completed. | The new values remain; the cleanup error can interrupt subsequent execution. |

For example, a producer may receive `@iAvailable` as an input reference and change it while running. If the later receiving assignment fails, that explicit change is not undone. The same applies to changes made by callbacks, file operations, database operations, or other side effects.

Consequently, catching an exception from a matching-target statement does not always mean that every involved variable still has its earlier value. The failure stage and independent side effects matter.

The feature also does **not** promise that several variables become visible to other threads as one atomic transaction. Shared variables continue to require the application's normal synchronization. The all-or-none receiving-check guarantee should not be used as a substitute for that synchronization.

### Returned references keep their ordinary meaning

Returning an Array, object, Error, WorkArea, or another managed reference does not deep-copy its contents. Distinct receiving variables may refer to the same underlying value, just as they can after ordinary reference-valued assignments.

Pending results remain valid through nested calls and required cleanup. Discarding a result does not destroy a value that is still retained elsewhere. These lifetime guarantees allow the caller to use returned values normally; they do not introduce a public result-container object.

## Functions, methods, and forwarding

### Forwarding a complete result list

A wrapper can return the complete result list from one call:

```harbour
FUNCTION AllocateFromMainWarehouse(iRequested Integer Required) ;
    RETURNS (Integer, Integer)

    RETURN SplitStock(iRequested, 7)

FUNCTION DemonstrateForwarding()
    LOCAL iAllocated Integer
    LOCAL iBackordered Integer

    iAllocated, iBackordered := AllocateFromMainWarehouse(12)
    // 7 and 5, forwarded from one SplitStock() invocation.

    RETURN NULL
```

The wrapper must declare its own complete result contract. Its result count must match, and the values must satisfy its contracts through the permitted checks and conversions. A wrapper may apply different compatible constraints; forwarding does not bypass its own checks.

`RETURN Producer()` forwards the complete list. Each expression in an explicit comma-separated `RETURN` list, however, must produce one value. A form such as `RETURN SplitStock(12, 7), 0` does not splice the pair into a three-result return.

The same restriction applies to ordinary argument lists. Multiple results do not implicitly expand into several arguments.

### Methods use the same result rules

A native class declares its method contracts structurally. Independent method implementations repeat the corresponding contract. The following class uses the previously defined `SplitStock()` helper:

```harbour
CLASS StockService
PUBLIC
METHOD Allocate(iRequested Integer Required, ;
                iAvailable Integer Required) ;
    RETURNS (Integer, Integer)
STATIC METHOD MainWarehouse(iRequested Integer Required) ;
    RETURNS (Integer, Integer)
ENDCLASS

METHOD StockService:Allocate(iRequested Integer Required, ;
                             iAvailable Integer Required) ;
    RETURNS (Integer, Integer)

    RETURN SplitStock(iRequested, iAvailable)

STATIC METHOD StockService:MainWarehouse(iRequested Integer Required) ;
    RETURNS (Integer, Integer)

    RETURN SplitStock(iRequested, 7)

FUNCTION DemonstrateMethods()
    LOCAL oService AS CLASS StockService := StockService()
    LOCAL iAllocated Integer
    LOCAL iBackordered Integer

    iAllocated, iBackordered := oService:Allocate( ;
        iAvailable = 7, iRequested = 12)
    // 7 and 5, using named arguments on a known receiver.

    iAllocated, iBackordered := oService:MainWarehouse(12)
    // 7 and 5, returned by the declared static method.

    RETURN NULL
```

Both instance and static methods use the same ordered receiving and forwarding rules. The named arguments in the first call illustrate ordinary HN named binding; their order does not reorder the method's results.

The `STATIC METHOD` declarations establish `MainWarehouse` as a static method; `oService:MainWarehouse(12)` is a supported invocation form for that method.

The structural declaration and method implementation must agree on result count, order, types, and constraints. Inherited methods retain their declared contracts. An override must be explicit and preserve the inherited result contract; this feature does not introduce broader or narrower override return types.

Supported dynamically resolved positional method calls check the actual target's result shape during execution. They cannot silently select one result either. Named-argument calls still require the supported statically known signature; multiple returns do not add a new way to invoke a saved Callable.

Constructors retain their existing single SELF/Object result. They cannot be declared as multiple-result producers.

### APIs retain their declared interfaces

Native HN APIs can expose multiple results through an explicit result contract. Application code receives those results with the same matching-target syntax and checks.

Adding language support does not automatically migrate existing APIs. A single-result API remains single-result; an output-reference parameter remains part of its interface until that API is explicitly redesigned. In particular, this feature does not change `hn_SQLSelect()` into a multiple-return API. Its documented row-count and output-reference behavior remains the applicable interface; see the [Local SQL Developer Overview](HarbourNova_Local_SQL_Developer_Overview.md).

Changing an API's result count, order, or contract is an interface change. Even callers that discard a position still depend on the complete declared shape. Rebuild affected callers through the normal HN build process when that contract changes.

## Returned errors and exceptions

`Error` is a normal result type using HN's established structured error value. `HNError` remains the accepted spelling in a catch declaration. Returning an Error value does not, by itself, throw an exception, invoke unhandled-error reporting, or preserve other receiving variables.

The examples in this section assume the NULL-safe native properties `:IsNull` and `:IsNotNull` are available on Error variables. `oError:IsNull` is TRUE when `oError` contains NULL and FALSE when it contains an Error. `oError:IsNotNull` gives the opposite result. These are properties, so the checks do not use parentheses.

An API can deliberately define a `(value, error)` convention. The following application helper treats a checked Text-length failure as a returned error. It uses a small scalar helper so the example needs no invented error-construction API:

```harbour
FUNCTION WarehouseCode(cCode Text Required) RETURNS Text(MaxLength=8)
    RETURN cCode

FUNCTION TryWarehouseCode(cCode Text Required) RETURNS (Text, Error)
    TRY
        RETURN WarehouseCode(cCode), NULL
    CATCH errValidation HNError
        RETURN NULL, errValidation
    ENDTRY

FUNCTION DemonstrateCodeValidation()
    LOCAL cStoredCode Text := "WH-01"
    LOCAL cCandidate Text
    LOCAL errValidation Error

    cCandidate, errValidation := TryWarehouseCode("WAREHOUSE-TOO-LONG")

    IF errValidation:IsNotNull
        cStoredCode := cCandidate
    ENDIF

    // cCandidate is NULL and errValidation contains the caught Error.
    // cStoredCode remains "WH-01" because this caller updates it only on success.

    RETURN NULL
```

`WarehouseCode()` returns Text constrained to eight characters. The long argument cannot satisfy that result contract. `TryWarehouseCode()` explicitly catches the resulting HNError and normally returns two values: NULL and the caught Error.

With a valid argument such as `"WH-02"`, the first result is that Text and the second is NULL. In this helper, a NULL error means success because the helper defines that convention. The language does not infer it from the position's type.

The candidate variable is significant. If the caller wrote directly into `cStoredCode`, a normal failure result of `NULL, errValidation` would assign NULL to that variable. Returning an Error is a successful language-level return of ordinary data, even when the data reports an unsuccessful application operation.

A Boolean status can also be useful, especially when “not found,” “unchanged,” or “cancelled by the user” are ordinary application outcomes. HN does not enforce any relationship between a Boolean result, another returned value, and an Error result. Document those relationships as part of the API.

Multiple returns do not replace exceptions. A function that raises an exception and does not convert it into ordinary results follows the established exception path. Discarding an Error position does not catch such an exception. Existing `TRY`, `CATCH`, `FINALLY`, rethrow, and primary/suppressed-error rules continue to apply.

### An illustrative HN API with an Error result

A new HN API can expose the same convention directly. Consider a proposed `hn_ReadTextFile()` API with this interface:

```text
FUNCTION hn_ReadTextFile(cPath Text Required) RETURNS (Text, Error)
```

The API name and signature are illustrative; this example does not identify an already implemented file API. It uses the same assumed NULL-safe native Error properties as the preceding example.

For this proposed API, the contract defines the following outcomes:

| Outcome | First result: Text | Second result: Error | `oError:IsNull` | `oError:IsNotNull` |
| --- | --- | --- | --- | --- |
| The file is read successfully. | The file contents. | NULL. | TRUE. | FALSE. |
| An empty file is read successfully. | Empty Text, `""`. | NULL. | TRUE. | FALSE. |
| An expected read failure occurs, such as a missing or inaccessible file. | NULL. | An Error describing the failure. | FALSE. | TRUE. |

The application receives both results from one call and checks the returned Error:

```harbour
FUNCTION DisplayTextFile(cPath Text Required) RETURNS Boolean
    LOCAL cContents Text
    LOCAL oError Error

    cContents, oError := hn_ReadTextFile(cPath)

    IF oError:IsNotNull
        // oError contains the structured failure information.
        // Application-specific handling can inspect or report that Error.
        ? "Could not read the file:", cPath
        RETURN FALSE
    ENDIF

    ? cContents
    RETURN TRUE

FUNCTION DemonstrateFileRead()
    LOCAL lDisplayed Boolean

    lDisplayed := DisplayTextFile("settings.txt")
    RETURN NULL
```

`oError` is the second **returned value**, not an output parameter supplied to the API. The receiving statement associates the file contents and any failure information with the same invocation. No `@` argument is needed for either result.

Both properties are safe to evaluate when no Error was returned. For this API's convention, `IF oError:IsNull` selects the success path, while `IF oError:IsNotNull` selects the reported-failure path. An empty file is still a successful read because its Error result is NULL; empty contents do not imply failure.

On a reported read failure, the matching-target assignment completes normally and assigns both NULL to `cContents` and the Error to `oError`. The subsequent `IF` handles the unsuccessful operation. An exception that propagates instead still follows HN's normal exception path; the property check does not catch it.

## Boundaries to keep in mind

Matching-target assignment receives the fixed results of **one top-level call**. Nested scalar calls and ordinary argument expressions remain useful within that call. The feature does not add general parallel assignment, variable swaps, Array unpacking, nested receiving patterns, variable-length native result lists, or a public tuple/result-bundle datatype.

Canonical saved `Callable`/`CodeBlock` acquisition and invocation, arbitrary Array-expression argument spread, and protected-region garbage-collection scheduling are separate workstreams. This article does not rely on them or describe them as capabilities delivered by multiple returns. No new multiple-result CodeBlock declaration syntax or automatic foreign-language tuple conversion is introduced here.

For an application API, document what every result position means, when it may be NULL, and which outcomes are normal returns versus exceptions. Keep small result lists stable and readable. When callers need a named, extensible value that they can retain or pass around as a unit, an object or collection remains a useful return type.

With those choices made explicitly, multiple returns let the function declaration describe the whole answer and let the receiving statement show exactly how the caller intends to use it.
