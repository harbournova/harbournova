# HarbourNova Tensors: adaptable storage for real application data

*An introduction to the native Tensor collection in HN1.*

HN1 introduces **Tensor**, a native collection for data organized into an explicit shape. A Tensor can serve as a one-dimensional sequence, a two-dimensional table of values, or a higher-dimensional structure such as product × warehouse × month. A rank-0 Tensor represents a single scalar cell.

Its most useful feature is the separation between **the data your program sees** and **the representation HN uses to store it**. A Tensor has a defined shape and ordinary indexed access, while its storage can adapt to how much data is present and how often values repeat. The same Tensor can use dense or sparse storage, with direct values or dictionary encoding, without requiring the application to replace it with a different collection type. [1, 3]

For suitable workloads, this changes what is practical. A large coordinate space does not necessarily require a large allocation, and millions of repeated values do not necessarily require millions of full-sized value representations.

Tensors are useful for business applications, imported datasets, lookup structures, reporting, scheduling, and intermediate query results. Developers can use them without doing scientific computing or machine learning.

## 1. What a Tensor adds to an Array

An ordinary Array is a useful sequence. Nested Arrays can represent rows, trees, or other structures, including rows of different lengths.

A Tensor adds an explicit **rectangular coordinate space**. Its dimensions belong to one collection, and an access such as `t[row, column]` addresses one cell in that space. It is a genuine multidimensional operation, rather than a sequence of accesses through separately managed child Arrays. [1]

For example, a Tensor with shape `[1000, 12]` has 1,000 positions along its first axis and 12 along its second. Every valid first-axis coordinate has the same second-axis extent, even when most cells contain no value.

| Requirement | Ordinary Array or nested Arrays | HN Tensor |
| --- | --- | --- |
| A simple sequence of items | A natural fit | Also supported through rank 1 |
| Rows with different lengths | A natural fit for nested Arrays | The outer coordinate space is rectangular |
| One shape covering several dimensions | Requires application conventions around nested collections | Part of the collection's own structure |
| A huge coordinate space with little populated data | Usually requires a separate sparse representation | Sparse storage is integrated |
| Many repeated immutable values | Savings depend on the Array's value representation and sharing | Dictionary encoding is an integrated option |
| Storage that adapts as data changes | Typically requires additional application logic | Managed by the Tensor runtime |
| Preserving coordinates during a resize | Requires careful handling across nested structures | Defined Tensor behavior |

Here, “Array” refers to HN's ordinary Array collection and conventional nested-Array usage. Other ecosystems also provide multidimensional arrays and specialized encodings: NumPy has an N-dimensional array type, and Apache Arrow documents dictionary encoding for repeated values. HN's advantage is bringing these storage choices into its native collection, type, ownership, and concurrency model. [6, 7]

## 2. Rank, shape, and coordinates

**Rank is the number of axes. Shape is the size of each axis.** In this context, rank does not mean the linear-algebra rank of a matrix.

| Rank | Example shape | Meaning |
| --- | --- | --- |
| 0 | `[]` | One scalar cell, with no axes |
| 1 | `[1000]` | A sequence of 1,000 cells |
| 2 | `[1000, 12]` | A rectangular grid with 12,000 cells |
| 3 | `[100, 20, 12]` | For example, product × warehouse × month |
| 4 | `[100, 20, 12, 5]` | The same structure with a fourth axis, such as scenario |

HN1 uses a general rank-N foundation. It is not restricted to matrices or rank 2. A matrix is a useful name for a rank-2 Tensor; it does not require a separate runtime type. [1, 2]

For a shape with at least one axis, the logical cell count is the product of its dimensions. A `[100, 20, 12]` Tensor therefore has 24,000 logical cells, whether all, some, or none are populated.

The zero-sized cases matter:

- `[0]` is an empty rank-1 Tensor.
- `[100, 0, 12]` is a rank-3 Tensor with no cells.
- `[]` describes a rank-0 Tensor with one cell.

The square-bracket shapes in this article are descriptive notation. In particular, `[]` is not a proposed HN declaration or access syntax for rank 0.

Coordinates are **one-based**. Coordinate 1 means the first position on an axis, and coordinate 0 is invalid. Negative coordinates address existing positions from the end: `-1` means the last position and `-2` the next-to-last. Shape and coordinate arguments use exact integer-family values; a numeric value such as `3.0` or a Text value such as `"3"` is not implicitly accepted as an integer coordinate. [1, 2]

Arbitrary rank also does not mean unlimited physical resources. Shape arithmetic, index calculations, and allocation requests remain checked.

## 3. A rank-1 Tensor can be used like an Array

A Tensor does not need several dimensions to be useful.

A rank-1 Tensor has one coordinate per cell:

```text
t[1] := "Queued"
t[2] := "Running"
t[3] := "Complete"
```

This looks familiar to anyone who uses indexed collections. The difference is that the collection also has Tensor storage behavior.

A rank-1 Tensor is especially useful when:

- Index positions have meaning, and most positions are empty.
- A long sequence contains relatively few distinct values.
- The sequence grows as data arrives.
- You want the same collection model for both one-dimensional and multidimensional data.

For example, a sequence indexed by a large numeric identifier may contain only a small number of active entries. A sparse rank-1 Tensor can retain that coordinate model without materializing every preceding position.

It remains a Tensor, with Tensor identity and semantics. Code that explicitly requires an Array should use the appropriate conversion rather than assume the two types are interchangeable. An ordinary Array remains a good choice for a short list or a deliberately irregular nested structure. [1, 2]

## 4. Support for the HN value system

HN Tensors are general data collections. Their cells participate in HN's value system, including numeric values, Boolean values, Text, Raw bytes, UUIDs, temporal values, and permitted references to collections and objects.

That makes them useful for application data that cannot sensibly be reduced to one machine-number format.

Two cell-contract models are available:

**An Any Tensor** can contain heterogeneous HN values, subject to the normal storage, ownership, and cycle rules. One cell can contain Text, another an integer, another a date, and another a reference to an Array or an object.

**A typed Tensor** has one common contract for its non-NULL cells. For example, a Text Tensor accepts Text values and NULL; a Tensor with a numeric contract enforces that contract throughout the collection. The contract belongs to the Tensor itself, so accessing it through an alias does not bypass its type rules. [1, 2]

This is a whole-Tensor cell contract, rather than a separate schema declaration for each column. A rank-2 Any Tensor can hold different value types in different columns, while a Cursor adds the named columns and per-column contracts appropriate to tabular records.

Automatic storage changes do not silently change the cell contract. They do not turn Text into numbers, round exact values to a cheaper approximate representation, or merge values whose HN semantics differ.

### NULL is different from zero, false, or empty Text

Every Tensor cell may be NULL, including cells in a typed Tensor.

Newly admitted cells start as NULL. In sparse storage, an absent physical entry represents that logical NULL.

These remain distinct values:

| Cell value | Meaning |
| --- | --- |
| `NULL` | No value |
| `0` | A populated numeric value |
| `FALSE` | A populated Boolean value |
| `""` | A populated empty Text value |

This distinction affects both correctness and storage. A grid full of numeric zeros is populated data; it is not automatically equivalent to an empty sparse grid. Repeated zeros may benefit from dictionary encoding, but they must retain their numeric meaning. [1, 2]

## 5. Two independent storage decisions

Tensor storage adapts along two different dimensions.

The first is **occupancy**: how many logical cells contain non-NULL values?

The second is **value repetition**: how many distinct eligible values occur among those populated cells?

These lead to four useful combinations. [1, 3]

| Occupancy and values | Suitable representation | What it saves |
| --- | --- | --- |
| Most cells populated; values mostly unique | Dense + Direct | Avoids sparse-coordinate and dictionary overhead |
| Few cells populated; values mostly unique | Sparse + Direct | Avoids allocating an entry for every logical cell |
| Most cells populated; many repeated values | Dense + Dictionary | Uses compact value identifiers across the dense coordinate space |
| Few cells populated; many repeated values | Sparse + Dictionary | Combines sparse coordinates with repeated-value encoding |

These are internal representations of one Tensor. The application continues to use the same coordinates and values.

### Dense storage

Dense storage provides physical coverage of the logical coordinate space. It is a good candidate when most cells are populated and regular access is useful.

A suitable homogeneous scalar contract may allow compact direct storage. Reference-bearing or heterogeneous values still follow their appropriate HN representation and ownership rules.

### Sparse storage

Sparse storage records populated coordinates and their values without creating a full physical grid.

Reading an in-bounds coordinate with no stored entry returns NULL. The read does not allocate a value merely because the coordinate was examined.

Writing a non-NULL value populates a coordinate. Writing NULL can remove its sparse entry.

### Dictionary encoding

Dictionary encoding stores eligible distinct values in an internal value table. Cells then use identifiers referring to those values.

For example, ten million status cells might contain only `"Queued"`, `"Running"`, `"Complete"`, and `"Failed"`. The internal dictionary can represent those four values, while the cells hold compact references to dictionary entries.

The application still reads and writes the status values. It does not manage the identifiers. [1]

## 6. Huge sparse spaces: capacity is not allocation

Consider a Tensor whose shape is:

```text
[1,000,000, 10,000]
```

That describes **ten billion logical cells**.

Suppose only 20,000 cells contain values. The occupancy is 0.0002%.

For an illustrative dense representation using eight bytes per cell, the cell payload alone would require:

```text
10,000,000,000 × 8 bytes = 80,000,000,000 bytes
```

That is **80 GB in decimal units**, before metadata, allocation overhead, or separately stored payloads.

A sparse representation instead needs the shape, information locating the 20,000 populated coordinates, and their actual values. Its storage is driven primarily by populated data and indexing overhead, rather than ten billion individual cell slots.

This is a capacity calculation, not an HN benchmark or a promise of a particular sparse-entry size. Coordinates, indexes, reference ownership, and value payloads all consume memory.

The practical advantage is still substantial: the application can retain its natural coordinate model without paying the dense-grid cost.

Useful applications include availability maps, combinations of product and location, optional observations, configuration overrides, and event-position grids.

A Hash keyed by coordinates can also model sparse data. Tensor integrates the rectangular shape, coordinate validation, resizing, cell contracts, and representation changes that an application would otherwise need to implement around that Hash. [1, 3]

## 7. Repeated values: a different source of savings

Sparse storage helps when many cells are NULL. Dictionary encoding helps when many populated cells repeat the same eligible values.

A Tensor can therefore benefit even when every logical cell is populated.

Consider ten million statuses drawn from four values. As a simplified illustration:

| Representation assumption | Cell identifiers or references alone |
| --- | --- |
| One eight-byte reference per cell | 80 MB |
| One one-byte dictionary identifier per cell | 10 MB |

The second case also needs the dictionary, NULL representation where applicable, and other metadata. Actual HN storage chooses its own encoding; the example demonstrates the opportunity rather than promising one-byte identifiers or an exact compression ratio.

An ordinary Array may already share repeated Text payloads. It would therefore be misleading to assume that every Array element necessarily contains a separate copy of the entire string. Dictionary encoding can still reduce the representation needed for each cell and make repeated-value handling an explicit storage strategy.

### Mutable objects retain their identities

HN's dictionary optimization applies to eligible repeated **immutable scalar values**.

Two distinct mutable Arrays or object instances must not be combined just because their contents happen to compare equal. Doing so would change observable behavior when one is later modified.

The same reference may legitimately appear in several cells, but ordinary reference sharing is different from dictionary deduplication. Tensor storage preserves that distinction. [1]

## 8. Storage can change while the Tensor stays the same

Real datasets change over time.

A Tensor may begin mostly empty, become densely populated during an import, and later lose many values. A column-like sequence may initially contain four statuses and later accumulate mostly unique text.

HN can move between supported sparse, dense, direct, and dictionary representations as the data changes. [1, 3]

For example:

1. A large, mostly empty Tensor starts with sparse storage.
2. Repeated values make sparse dictionary encoding attractive.
3. Greater occupancy makes a dense representation attractive.
4. Increasing value diversity makes direct storage preferable.
5. Clearing much of the data makes sparse storage attractive again.

This sequence illustrates possible choices, not mandatory thresholds or a guaranteed conversion after each write.

Representation changes preserve the Tensor's identity, shape, values, cell contract, and normal alias relationships. Applications do not need to replace every reference because the storage strategy changed.

Optional hints can describe expectations such as mostly sparse data or many repeated values. These are advisory. Actual contents and available resources still determine what representation is appropriate.

Automatic adaptation is a policy, not a claim that every Tensor always occupies the mathematically smallest possible number of bytes. Conversions have costs, and dense storage can be the better choice for dense workloads.

## 9. Automatic sizing and growth

Automatic sizing has two related meanings:

- The logical shape can grow to include newly assigned coordinates.
- Physical storage can grow or change representation to hold the actual data.

A write beyond a positive bound expands the affected existing axes.

For example:

```text
Initial shape:       [10, 5]
Assignment:          t[25, 8] := "Ready"
Resulting shape:     [25, 8]
Logical cell count:  200
```

Previously stored values keep their coordinates. Newly admitted coordinates contain NULL, except for the cell receiving `"Ready"`.

A Tensor destination with a known Tensor contract can also establish its initial rank and minimum shape from its first indexed write. Once established, ordinary indexed writes can extend existing axes; they do not silently add axes. [1, 2]

The boundary rules are deliberate:

- Reads never grow a Tensor.
- Negative coordinates refer to the current shape and never request growth.
- Coordinate zero is invalid.
- Supplying extra coordinates does not turn a rank-2 Tensor into rank 3.
- Writing NULL beyond a positive bound still grows the logical shape.

That last case illustrates the difference between logical size and populated size. A NULL assignment can increase the number of valid coordinates without adding a populated value.

Growth and the assignment are committed as one mutation. A rejected value, invalid coordinate, or allocation failure must leave the previous Tensor intact. Automatic growth is not permission to weaken its type contract. [1, 2]

## 10. Resizing preserves coordinates

Explicit resizing changes the sizes of existing axes.

Suppose a Tensor has shape `[3, 4]`:

| Row | Column 1 | Column 2 | Column 3 | Column 4 |
| --- | --- | --- | --- | --- |
| 1 | A | B | C | D |
| 2 | E | F | G | H |
| 3 | I | J | K | L |

Resizing it to `[3, 2]` produces:

| Row | Column 1 | Column 2 |
| --- | --- | --- |
| 1 | A | B |
| 2 | E | F |
| 3 | I | J |

The surviving values keep their original multidimensional coordinates. HN does not simply preserve the first six entries of a flattened representation.

Values removed by shrinking are discarded according to normal ownership rules. If the Tensor later returns to `[3, 4]`, columns 3 and 4 contain new NULL cells; C, D, G, H, K, and L do not reappear.

Successful resize preserves Tensor identity. Other aliases to that Tensor see the resized collection. If a resize fails, the previous shape and values remain valid. [1, 2]

### Resizing and changing rank are different operations

Changing `[100, 12]` to `[200, 12]` changes a dimension's length while retaining rank 2.

Changing `[100]` into `[10, 10]` changes how coordinates describe the data. That requires an explicit structural transformation. The broader Tensor architecture distinguishes reshape, which preserves logical cell count, from resize, which preserves surviving coordinates.

HN1's rank-N foundation supports data with any number of axes from the beginning. That foundation should be distinguished from the larger set of reshape, axis-manipulation, view, and mathematical APIs in the continuing Tensor design. Ordinary indexing never guesses that the programmer intended a different rank. [1, 2]

## 11. Native properties and methods

Tensor participates in HN's native value member model.

Properties expose information about a value. Methods perform an operation. The familiar member forms are:

```text
value:Property
value:Method(...)
```

These are language-native members; the application does not need to create a wrapper class just to inspect or operate on a Tensor.

The following table describes the member responsibilities. Its labels describe capabilities rather than fixing public API spellings. [1, 4]

| Member responsibility | What it tells you or does |
| --- | --- |
| Rank observation | Reports the number of axes |
| Shape observation | Reports the size of each axis |
| Logical-size observation | Reports the number of logical cells |
| Populated-size observation | Reports how many cells are non-NULL |
| Cell-contract observation | Reports the Tensor's non-NULL type contract |
| Storage inspection | Reports relevant current representation and approximate allocation information |
| Resize operation | Changes existing axis lengths while preserving surviving coordinates |
| Copy operation | Produces a separate Tensor using the shallow-copy contract described below |

These distinctions matter when inspecting a large sparse Tensor. “Size” might otherwise be mistaken for logical capacity, populated values, or allocated memory.

Storage information is diagnostic. Application correctness should follow logical Tensor semantics rather than depend on a particular encoding remaining active.

Native member chaining follows the return value of each member. A method that returns a suitable value can participate in a chain; a mutating method should not be assumed to return the original Tensor unless its API explicitly says so. [4]

## 12. Iteration still has logical meaning

Sparse storage reduces what must be stored. It does not change the meaning of visiting every logical cell.

Ordinary Tensor iteration follows logical cell order and includes NULL cells. The default order is **last axis fastest**. For a rank-2 Tensor, that means visiting the columns of one row before proceeding to the next row.

For shape `[2, 3]`, the order is:

```text
[1,1], [1,2], [1,3], [2,1], [2,2], [2,3]
```

Changing the physical representation must not change that order. A rank-0 Tensor has one logical cell to visit; a Tensor with a zero-sized axis has none. [1, 2]

This is important for huge sparse spaces. A request to visit all ten billion logical positions is still a request for ten billion visits, even if storage contains only 20,000 populated entries.

The wider architecture distinguishes populated-only traversal and search from ordinary logical traversal. When designing algorithms for sparse data, use operations that work on populated coordinates or retain the known coordinates of relevant data. Do not assume a general full-grid loop becomes inexpensive merely because its backing storage is sparse. [1]

## 13. References, copies, and contained objects

Two variables may refer to the same Tensor. Mutating the Tensor through one writable alias changes the collection seen through the other.

An explicit shallow copy gives the application a different Tensor with independent shape and cell bindings. Replacing a cell in the copy does not replace the corresponding cell in the source.

However, reference-bearing cell values remain shared under a shallow copy.

If a cell contains an Array, both Tensors may still refer to that same child Array. Mutating the child can therefore be visible through both Tensors, even though replacing a Tensor cell is independent.

The broader design defines deeper copy modes separately. A shallow copy should never be treated as proof that an entire nested object graph is independent. These rules also explain why dictionary encoding must preserve reference identity and why concurrency rules must account for contained mutable values. [1, 4]

## 14. Tensors, Jobs, Tasks, and multithreading

Tensor is a collection object, so its concurrency behavior must cover more than reading or writing one memory location.

A write can affect a cell, a sparse-coordinate index, a value dictionary, allocated storage, or the shape itself. Automatic representation changes also affect shared internal structure.

HN's collection model distinguishes locally confined data from data intended for shared access. A local mutable collection belongs to one logical HN execution. Shared mutable data follows the shared-collection contract. The local/shared policy is selected when the collection is created and remains fixed for its lifetime. This distinction belongs to the collection; it is independent of whether a variable happens to be declared LOCAL or stored in a STATIC binding. [4, 5]

### Individual operations are synchronized

The Tensor design provides automatic operation-level synchronization associated with Tensor identity.

For a Tensor used under the shared-access contract, supported operations must preserve a coherent shape, coordinate mapping, values, and storage representation. Another worker must not observe a half-completed resize or a partially replaced dictionary.

All aliases to the same Tensor participate in the same synchronization domain. Different coordinates do not remove this requirement: writes to different cells can still affect common sparse or dictionary metadata. [2, 4, 5]

This describes the HN1 concurrency contract, without prescribing a public lock implementation or promising lock-free access.

### A sequence of safe operations is still a sequence

Suppose two Tasks increment the same cell:

```text
Read current value
Add one
Write updated value
```

If both read 10 before either writes, both can write 11.

Each read and write can be safe individually while the overall increment still loses an update.

Compound actions therefore need an appropriate synchronization boundary covering the complete operation. HN's SYNC model supplies the broader principle: protect the short sequence whose intermediate state must not be interleaved with another worker. [4, 5]

The same reasoning applies to checking a value before changing it, updating several related cells, or reading shape and cell data that must describe the same moment.

Protected work should remain short. Waiting for another Task, doing blocking I/O, or holding protection while fetching data from a backend belongs outside that protected region.

### Traversal needs a stable contract

A collection traversal is a multi-step activity.

Per-operation safety alone does not make a long iteration a snapshot. A worker that needs a stable traversal must use the applicable protected-iteration or independent-copy contract.

Likewise, a read-only reference prevents mutation through that reference; it does not by itself stop another writable alias from changing the Tensor.

### Contained objects keep their own rules

Safely reading an object reference from a Tensor does not automatically synchronize later calls on that object.

A shared Tensor containing Arrays, Hashes, other Tensors, or class instances must respect the sharing and ownership rules of those values as well. A thread-safe outer container is not a recursive exemption from the rules governing its contents.

### Independent Tensors are often a useful parallel pattern

Jobs and Tasks can also divide work without having every worker mutate one shared Tensor.

For example:

1. Several Tasks fetch or transform independent data partitions.
2. Each Task builds its own permitted local results.
3. Results pass through the appropriate HN result or communication contract.
4. A coordinating stage assembles the combined data.
5. A shared collection is created where the application actually needs shared access.

This can reduce contention during construction. It also keeps remote I/O outside short synchronized mutations.

Passing a reference alone does not convert a local collection into a shared one. The handoff must follow the collection and communication contracts. [4, 5]

## 15. Why this matters for application data and Cursors

A rank-2 Tensor is a useful rectangular data container. It can represent imported values, intermediate query results, report grids, or data being prepared for another processing stage.

A Cursor adds record-oriented semantics such as named columns, column metadata, record navigation, and indexes. A Tensor supplies coordinate-oriented storage and a common cell contract. The two are useful at different levels.

HN's Cursor direction also allows the reusable Tensor storage foundation to support future Cursor storage integration. Sparse values and repeated values are common in business datasets, so sharing that foundation can avoid implementing the same storage machinery twice.

The benefit does not require a Cursor and a Tensor to become the same public type. Cursor schemas and row behavior remain meaningful, while Tensor storage can provide reusable mechanisms underneath. [3]

For developers converting code that uses rectangular Arrays, Tensor also provides a natural destination with explicit dimensions and predictable coordinate-preserving growth.

## 16. Serialization and reload: planned persistence

**Serialization and reload are part of the wider Tensor design. This capability is described here as planned; it is separate from the HN1 storage features above.**

The planned model converts a Tensor to an HN Raw byte representation and reconstructs a Tensor from those bytes. The bytes can then be used for file storage, caching, transport, or a suitable database binary field. [1]

The serialized form preserves logical information such as rank, shape, cell contract, values, and NULLs.

It can choose an appropriate compact encoding from the same broad combinations:

- Dense + Direct.
- Dense + Dictionary.
- Sparse + Direct.
- Sparse + Dictionary.

Reload does not have to reproduce the original in-memory layout. The runtime can choose a different live representation while preserving the same logical Tensor.

Advisory storage hints can travel with the data, but remain advisory after reload.

### In-memory eligibility and serialization eligibility differ

A Tensor can hold values that do not have a portable serialized representation.

The planned serialization contract recursively accepts values with defined portable HN encodings. This includes suitable scalar values and supported nested Arrays, Hashes, and Tensors when their reachable contents are also serializable.

Class instances, executable values, database connections, file handles, WorkAreas, and similar runtime-bound values are excluded unless a future specification explicitly defines a safe portable encoding.

If an unsupported value occurs anywhere in the reachable contents, serialization fails as a whole. It does not silently omit that value.

Legitimate sharing between serializable nested collections is preserved. If two cells reference one Array, reload should produce two references to one reconstructed Array, rather than silently turn it into two unrelated copies. [1]

## 17. Choosing between an Array and a Tensor

The useful question is which collection expresses the data and its expected growth most clearly.

| Situation | Usually the better starting point |
| --- | --- |
| A short, ordinary sequence of items | Array |
| A tree or nested structure with deliberately uneven child lengths | Array or other appropriate collections |
| A rectangular grid with explicit dimensions | Tensor |
| One-dimensional data with a very large, mostly empty index space | Rank-1 Tensor |
| Large volumes of repeated immutable values | Tensor, where dictionary encoding is beneficial |
| Several independent coordinate axes | Tensor |
| Records requiring named columns, metadata, navigation, and indexing | Cursor |
| Scientific tensor algebra or specialized accelerator workloads | Evaluate the specific numerical facilities required; rank-N storage alone does not supply them |

A Tensor can have more metadata and coordination cost than a small Array. Dense direct storage can also outperform a more elaborate representation when data is dense and mostly unique.

The strongest cases for Tensor are those where dimensions, sparsity, repetition, and changing data volumes are meaningful parts of the application. In those cases, HN provides a native collection that can adapt while the developer continues to work with stable coordinates and HN values.

## Design references

HN-specific descriptions are based on the HarbourNova project design and owner decision records listed below. They describe HN1 feature behavior; the serialization section explicitly identifies the wider planned capability. Public member names remain governed by the API catalog.

1. **HarbourNova Tensor Architecture Review Checkpoint, October 7, 2026** — rank and shape, cell contracts, NULL, adaptive storage, growth, resizing, iteration, copy semantics, and planned serialization.
2. **HN1 Tensor Foundation specification, October 13–14, 2026** — rank-N foundation, indexing edge cases, coordinate-preserving resize, failure atomicity, and the per-Tensor synchronization direction.
3. **HN1 Tensor storage scope and Cursor sequencing decisions, October 19, 2026** — adaptive sparse/direct/dictionary storage in HN1 and Tensor-first Cursor integration.
4. **HN collection and native value member decisions, October 9–17, 2026** — local/shared collections, copying, compound synchronization, and native methods and properties.
5. **HN Concurrency Overall Architecture, September 30, 2026, and subsequent collection decisions** — Tasks, Jobs, communication, and short protected operations.
6. **NumPy: [The N-dimensional array](https://numpy.org/doc/stable/reference/arrays.ndarray.html)** — background comparison for multidimensional array terminology.
7. **Apache Arrow: [Dictionary-encoded layout](https://arrow.apache.org/docs/format/Columnar.html#dictionary-encoded-layout)** — background comparison for repeated-value encoding.

The numerical storage examples in this article are illustrative calculations using stated assumptions, not measured HN performance results.

