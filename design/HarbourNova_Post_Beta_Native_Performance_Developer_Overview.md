<!--
HarbourNova
Copyright 2026 Eric Lendvai (WA-USA)
SPDX-License-Identifier: GPL-3.0-or-later
-->

# HarbourNova after the HN1 Beta: Native performance and a pragmatic C/Rust strategy

*Why I am retaining C at HarbourNova's core, how explicit types can improve performance, and where selective Rust components may eventually make sense.*

**Developer overview — revised November 13, 2026**

> **Scope:** This article presents my approved **post-Beta performance direction** and areas for future investigation. It is not a claim that these optimizations or Rust migrations have been implemented, a promise of a particular speedup, or an expansion of the HN1 Beta feature list. Any implementation will require separate evaluation and selection.

I want HarbourNova to make it practical to build substantial business and data-oriented applications in a language that is expressive, approachable, and efficient. Developers should be able to write a small function without thinking about its machine-code implementation. At the same time, the compiler should take advantage of information already available to it instead of introducing unnecessary runtime work.

**After the HN1 Beta release, I intend to measure where execution time and memory are spent, investigate the highest-value opportunities, and retain only optimizations that preserve HarbourNova's language behavior.**

This is an evolution of the current architecture—not a restart, a new language, or a commitment to rewrite the compiler in Rust.

## Why I am retaining C at the core

The HarbourNova compiler and much of its runtime are rooted in C. C is well suited to low-level systems work: it gives precise control over memory and platform interfaces, works with mature native compilers on Windows and Linux, and can produce highly efficient machine code.

Rust is also a capable systems language. Its ownership and type system help prevent many memory-management errors **in code that actually benefits from Rust's safe abstractions**. Rust is not inherently faster than C for an equivalent operation, and a compiler written in Rust does not automatically make the applications it generates memory-safe.

Replacing HN's established C compiler, runtime, garbage collector, calling conventions, and module infrastructure would mean rebuilding and revalidating a substantial amount of behavior. That is not a sensible prerequisite for releasing HN1 or improving the performance of HN applications.

I am not rejecting Rust. HarbourNova already uses Rust for **`hnmake`**, its native build coordinator. I also intend to consider Rust for isolated components where its safety guarantees offer a concrete advantage. But the C core and the existing HN value, runtime, and HNB module contracts remain the foundation for now.

## What I want to make faster after Beta

### 1. Direct calls when the function is known

A dynamic language runtime may need to look up a function, prepare arguments, construct an execution frame, and use a general dispatch mechanism. That flexibility is essential for genuinely dynamic calls. It can be unnecessary when the compiler can prove the exact target and the calling contract permits a simpler path.

I want to investigate generating **direct native calls** for statically bound HN functions, beginning with functions that accept no arguments. A zero-argument function is a useful first benchmark because it makes dispatch overhead easy to observe. It is not automatically safe to bypass all runtime machinery: error reporting, debugging, Task context, dynamic binding, and cleanup may still require an execution frame or adapter.

I will compare the ordinary HN execution path, a specialized generated-C path, and equivalent handwritten C under controlled conditions. The objective is to approach native call overhead where the compiler can prove that doing so is correct—not to promise a universal performance multiplier.

### 2. Optimize operations with explicitly declared datatypes

HarbourNova supports optional explicit datatypes. That information can help the compiler avoid repeated type checks, choose direct operations for known values, and reduce unnecessary conversions or temporary allocations.

Typed code can use a specialized path when justified, while deliberately dynamic code retains its complete runtime behavior. The compiler must not guess an untyped variable's type or treat a declared base class as proof that no derived method override exists.

**Speed must never come at the expense of numerical correctness.** The HN `Number` type provides exact decimal arithmetic. Faster arithmetic must preserve precision, scaling, range, widening, and error semantics—not silently substitute an ordinary binary floating-point result.

### 3. Make native properties, methods, and APIs lightweight

Common operations such as testing for NULL, getting Text length, comparing numbers, accessing Arrays or Hashes, and reading Cursor fields should not incur avoidable dynamic-dispatch costs when their implementation is known.

The HN API catalog and compiler-owned signatures make direct native dispatch possible in principle. However, user-defined class overrides must remain observable. For example, a user-defined `IsNull` property cannot simply be replaced with a built-in test when an override may apply.

### 4. Reduce unnecessary allocation and value handling

Allocating temporary objects, copying values, converting representations, and managing reference counts all cost time. I intend to investigate whether the compiler can safely avoid some of this work through better lifetime and escape analysis.

Such changes must preserve HN's existing sharing and garbage-collection semantics. Arrays, Objects, codeblocks, and other managed values must keep their correct lifetimes even when shared or referenced in cycles. Replacing those semantics with move-only behavior for the sake of a benchmark is not part of this plan.

### 5. Measure applications, not only microbenchmarks

Small loop and function-call benchmarks help isolate overhead, but a business application may spend most of its time performing SQL operations, formatting exact values, accessing Tensors or Cursors, waiting for I/O, or running Tasks.

I intend to include representative application workloads in the post-Beta measurements. For long-running FastCGI services, predictable response latency, stable memory consumption, and correct collection behavior may be more important than the speed of an isolated arithmetic operation.

## Where Rust may provide a different kind of benefit

There is a separate reason to consider Rust: **memory safety in components that handle complex ownership, untrusted input, concurrency, or pointer-intensive data structures**.

Calling a C component *higher-risk* does **not** mean that HarbourNova's current C implementation is defective. It means an error in that kind of subsystem could be difficult to detect or reproduce, or could have serious consequences. It also does not mean that rewriting it in Rust would automatically make the result safe. Rust's benefits depend on the design and how much code can remain within safe Rust rather than relying on raw pointers and `unsafe` interfaces.

The following table identifies areas I may evaluate. **These are potential risk areas and investigation priorities, not a list of discovered defects or an approved migration schedule.**

| Component | Why it can be risky in C | Potential Rust advantage | Initial assessment |
| --- | --- | --- | --- |
| **Binary file parsers** | Malformed offsets, lengths, and buffer boundaries can trigger out-of-bounds access. | Checked indexing, slices, and explicit error paths. | **Very strong candidate** for an isolated implementation. |
| **Network protocols and TLS-related handling** | Untrusted packets, buffers, connection state, and complex protocol transitions. | Safer parsing and ownership of protocol state. | **Very strong candidate**, preferably using mature, reviewed libraries. |
| **Compression and decompression** | Corrupt streams, integer overflow, and unexpected allocation sizes. | Checked arithmetic and safer buffer management. | **Strong candidate** with a narrow input/output boundary. |
| **Serialization and deserialization** | Untrusted nested data, recursion limits, and allocation pressure. | Structured parsing with explicit resource limits. | **Strong candidate** for selected formats. |
| **Tensor storage operations** | Resizing, reallocation, views, aliases, and potentially stale references. | Safer ownership and bounds handling. | **Worth studying**, but sharing and view semantics make migration complex. |
| **Task queues and timers** | Races during cancellation, shutdown, and callback cleanup. | Thread-safe ownership and synchronization primitives. | **Worth studying**, but highly integrated with the runtime. |
| **Database drivers** | Native handles, network buffers, and result lifetimes cross library boundaries. | Deterministic resource management and safer wrappers. | **Evaluate individually**; an established native client may already handle much of the risk. |
| **HNB module loading** | Parsing binary structures, resolving symbols, and handling unload lifetimes. | Safer decoding and validated module metadata. | **Selective candidate** for parsing; runtime symbol ownership remains difficult. |
| **GC and HN value management** | Shared object graphs, reference cycles, finalizers, and native interoperation. | Some opportunities for safer internal structures. | **High-risk code to change; poor early rewrite candidate.** |
| **Compiler parser and front end** | Complex source inputs and internal lifetime relationships. | Stronger memory safety for new implementations. | **Possible long-term candidate**, but a major migration with uncertain payoff. |

For example, a binary decoder can often accept a byte buffer and return validated data through a small, stable C interface. That gives Rust a well-defined ownership boundary without exposing the layout of HN's internal values. The GC, by contrast, is deeply intertwined with those values, objects, Tasks, and foreign-language interfaces. Rewriting that subsystem would carry much more risk.

The goal is to **adopt Rust selectively where the safety benefit is demonstrable**, not replace working C code merely because another implementation language is fashionable. A faster application is not guaranteed by that migration, and safe Rust code can still be surrounded by an unsafe foreign-function boundary that requires careful validation.

## What must not change

Any performance improvement or implementation-language change must preserve these HN contracts:

- **Exact values remain exact.** Optimizations must respect decimal arithmetic, numeric widening, and specified errors.
- **Dynamic execution remains available.** The VM and HNB module system keep their supported loading, unloading, and binding behavior.
- **Sharing and lifetime semantics remain intact.** There is no implicit conversion to incompatible Rust-like move-only behavior for HN collections or objects.
- **Overrides and observable calls remain correct.** Argument evaluation, multiple returns, NULL handling, and user-defined methods retain their meaning.
- **Diagnostics remain useful.** Debugging, stack information, and required error behavior must not disappear because a call becomes faster.
- **Concurrency stays correct.** Tasks, shared values, synchronization, and GC safe points must respect their contracts.

If the compiler cannot prove a fast path safe, it should continue to use the general execution path.

## How I intend to proceed after Beta

The engineering plan is organized into possible phases. **None is automatically started by the Beta release**, and no speedup is currently claimed.

| Phase | Investigation | Evidence required |
| --- | --- | --- |
| **Measure HN1 Beta** | Establish reproducible timing, executable-size, allocation, and memory baselines. | Defined compiler/runtime revisions, toolchain settings, and representative workloads. |
| **Direct-call experiments** | Start with proven zero-argument calls, then examine small typed functions. | Identical semantics and measured improvement over the existing HN path. |
| **Typed and native operations** | Evaluate explicit-type fast paths, native properties, and catalog-resolved APIs. | Correct behavior on boundary conditions and observable overrides. |
| **Value and memory costs** | Identify avoidable copies, allocations, conversions, and reference-management work. | Reduced overhead without GC, lifetime, or sharing regressions. |
| **Application workloads** | Study Tensors, Cursors, HNB, FastCGI, and Task-based execution. | Benefits on realistic workloads, including memory stability and latency. |
| **Selective Rust evaluation** | Investigate isolated, high-risk native components where safe Rust could improve reliability. | A clear interoperability boundary and demonstrated safety/maintenance benefit. |

The native-performance investigations are an approved **post-Beta** direction. The Rust candidates are examples for future evaluation, not a newly scheduled HN1, HN2, or HN3 rewrite.

## Can HarbourNova approach C or Rust execution speed?

For a simple operation whose types and call target are fully known, I believe it is technically realistic for HN-generated C to approach the cost of equivalent handwritten native C. That is an objective to test, not an established measurement for HarbourNova today.

Whole applications are more complex. Dynamic dispatch, exact decimal values, object sharing, I/O, database operations, and runtime services have legitimate costs. Comparisons are meaningful only when programs perform equivalent work with equivalent semantics.

**My standard will be to measure HN against a validated HN baseline, preserve behavior, and publish results with their limitations.**

## A personal commitment to the project

HarbourNova is a **single-developer initiative**. I use AI-assisted development tools, but the architectural decisions, technical review, priorities, and responsibility for what is released are mine. I do not present HarbourNova as the work of a development team.

My immediate priority is completing and releasing the HN1 Beta. After that, I intend to improve performance where measurements justify it, preserve the dynamic features that make HN useful, and introduce additional Rust components only where their advantages are clear.

**The goal is not to make HarbourNova look like C or Rust. It is to make HarbourNova applications efficient and dependable without giving up what makes HarbourNova useful.**

---

For more information, browse the [HarbourNova design documents](https://github.com/harbournova/harbournova/tree/main/design). To follow development or share feedback, visit the [HarbourNova users group](https://groups.google.com/g/harbournova-users).
