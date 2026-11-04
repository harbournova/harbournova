# HarbourNova HN1 Temporal Data Types

**Developer Design Overview — Revision 002**

- **Design checkpoint:** November 4, 2026
- **Status:** HN1 architecture approved in substantial part; **not a declaration that the full feature family has been implemented or released**
- **Project:** [HarbourNova](https://github.com/harbournova/harbournova)

> **About this document.** This is a description of the intended HN1 temporal programming model. The existing HN runtime already has a limited `TimestampTZ` carrier and current-time APIs. The six-type model, nanosecond arithmetic, IANA/CLDR module, richer constructors/parsers, and associated SQL behavior described here are **HN1 design targets** until implementation and acceptance are completed. A few detailed API/persistence contracts remain open, as noted below.

## Why HarbourNova takes a different approach

Database applications frequently confuse three things that are not interchangeable: a *calendar date*, an *instant on the global timeline*, and a *calendar-relative period*. That confusion produces subtle defects around daylight-saving changes, historical time zones, month-end calculations, financial reporting, and database interchange.

HN1's temporal architecture separates those concerns. It adopts PostgreSQL-compatible calendar conventions where appropriate, increases fractional precision to nanoseconds, and deliberately preserves more original time-zone information than a PostgreSQL `timestamptz` alone can retain.

## The six HN1 temporal types

| Type | Meaning | Typical use |
| --- | --- | --- |
| `Date` | Gregorian calendar day, without time or zone | Birth date, invoice date, holiday |
| `Time` | Time of day without date, offset, or zone | Opening time, daily cutoff |
| `Timestamp` | Calendar date and time, without a globally resolved instant | A scheduled local appointment before zone resolution |
| `TimestampTZ` | Resolved UTC instant plus recorded original offset and optional named zone | Events, audit timestamps, globally comparable transactions |
| `Duration` | Exact elapsed nanoseconds | Timeouts, elapsed execution time, measuring an event |
| `Interval` | Months, calendar days, and elapsed nanoseconds as distinct components | Add one calendar month, schedule the next billing date |

There is **no independent native `TimeZone` data type in HN1**. Zone names such as `America/Los_Angeles` are `Text` identifiers, optionally retained in a `TimestampTZ`. `TimeTZ` is also excluded from HN1; the historic internal registry identity remains reserved.

## 1. `Date`: a calendar day, not an instant

`Date` uses a **proleptic Gregorian calendar** across its complete range, including dates before the Gregorian calendar was historically adopted. There is no region-dependent Julian/Gregorian cutover. HN1 supports BC dates and the familiar year sequence **1 BC → AD 1**, with no public year zero.

The intended finite range matches PostgreSQL `date`, approximately **4713 BC through 5,874,897 AD**, subject to exact endpoint validation. `Date` also supports explicit negative and positive infinity states.

```harbour
// Planned multi-result constructor
LOCAL dInvoiceDate Date
LOCAL oError HNError

dInvoiceDate, oError := hn_NewDate(2027, 2, 28)
IF oError:IsNotNull
    ? oError:Message
ENDIF
```

Invalid month/day combinations fail rather than being silently normalized. Negative constructor years denote BC years; year zero is invalid.

## 2. `Time`: a clock reading without a date

`Time` holds a time of day with nanosecond resolution and **no UTC offset**. It accepts ordinary times from `00:00:00` through `23:59:59.999999999`, as well as the **distinct exact endpoint `24:00:00`**. The latter is useful as the end of a day; invalid forms such as `24:00:01` are rejected.

```harbour
LOCAL tCutoff Time
LOCAL oError HNError

tCutoff, oError := hn_NewTime(17, 30, 0)
```

`Time` has no infinity states. Adding a finite elapsed quantity wraps to the 24-hour clock; subtracting one `Time` from another produces a **signed, non-wrapping difference**, represented as an `Interval` with zero calendar components. For example, `24:00 - 00:00` represents 24 elapsed hours, not zero.

## 3. `Timestamp`: a calendar time, not yet a resolved instant

`Timestamp` combines a date and time but intentionally contains **no zone or UTC instant**. `2027-11-07 01:30:00` is a valid local timestamp even where daylight-saving transitions make that clock reading ambiguous.

The intended finite range matches PostgreSQL timestamp's year-level domain, approximately **4713 BC through 294,276 AD**, again with the exact finite endpoints to be verified. Timestamp supports ±infinity.

```harbour
LOCAL tsMeeting Timestamp
LOCAL oError HNError

tsMeeting, oError := hn_NewTimestamp(2027, 11, 7, 1, 30, 0)
```

To turn a local timestamp into a globally resolved instant, the application must explicitly provide an IANA zone name or numeric offset. HN1 does not assume the machine's current local time zone as a hidden constructor default.

## 4. `TimestampTZ`: an instant with recorded context

A finite `TimestampTZ` stores **four separate facts**:

1. The exact **UTC/POSIX instant**, including nanoseconds.
2. The **original recorded UTC offset**, in signed whole seconds, from `-23:59:59` through `+23:59:59`.
3. An optional **original IANA zone identifier**, preserving its written spelling or recognized alias.
4. The recorded **fractional-second precision**, from zero to nine digits.

This differs deliberately from PostgreSQL `timestamptz`, which normalizes an instant to UTC and does not retain the original input offset or zone identifier. HN can preserve the originally recorded local representation even if governments later revise time-zone rules.

An example default representation is:

```text
2027-03-14 04:30:00-07:00 [America/Los_Angeles]
1890-01-01 12:00:00+00:19:32 [Europe/Amsterdam]
2027-03-14 12:30:00+00:00
```

The bracketed zone name is preceded by **one space**, and historical offsets may contain nonzero seconds. The zone identifier is optional; it is not silently inferred from an offset.

### Equality: same instant versus same recorded information

Two `TimestampTZ` values may have different offsets or names but represent the same instant:

- `==` and chronological ordering compare the **UTC instant**.
- `===` compares the **entire recorded representation**, including instant, original offset, zone identity/provenance, and fractional precision.

This permits chronological SQL grouping while still retaining complete audit information in each surviving value. It also means storage compression and lossless serialization must not discard different original representations merely because two instances compare equal with `==`.

### Construction and conversion

```harbour
LOCAL tsLocal Timestamp
LOCAL dtzEvent TimestampTZ
LOCAL oError HNError

// Resolve using named IANA rules
// (tsLocal is already a valid local Timestamp.)
dtzEvent, oError := hn_NewTimestampTZ( ;
    tsLocal, "America/Los_Angeles" )

// Resolve using an explicitly supplied numeric offset instead
dtzEvent, oError := hn_NewTimestampTZFromOffset(tsLocal, -25200)

// Represent the same instant in a new named zone
dtzEvent, oError := hn_TimestampTZInZone( ;
    dtzEvent, "Europe/Brussels" )

// Or represent it with a new fixed offset (seconds)
dtzEvent, oError := hn_TimestampTZInOffset(dtzEvent, 3600)
```

The `InZone` and `InOffset` operations return **new values** at the same instant; they do not convert currencies, shift elapsed time, or mutate the original. Extracting a zone-free local `Timestamp` uses the **recorded offset**, without recomputing past zone rules.

### Daylight-saving transitions

A local wall-clock time may occur twice during an autumn *fold*, or not at all during a spring *gap*.

For a fold, HN1 uses the **later occurrence (post-transition/typically standard-time offset)**, matching PostgreSQL's ordinary default. For a gap, HN1 **rejects by default**. An explicitly selected `HN_TZ_GAP_SHIFT_FORWARD` mode advances by the actual transition gap; `HN_TZ_GAP_REJECT` is the default.

An explicitly supplied numeric offset is authoritative even if the accompanying recorded zone name is currently unrecognized or its modern rules disagree. HN preserves the original facts rather than quietly replacing them.

## 5. `Duration`: elapsed time without calendar ambiguity

`Duration` represents an exact signed count of nanoseconds, using a logical **signed 128-bit** finite domain, plus ±infinity states.

A duration of 24 hours always means **86,400 elapsed seconds** on HN's POSIX-compatible timeline. It does *not* mean “the same local clock time tomorrow,” which can differ by an hour when daylight saving changes.

The default textual representation is ISO-oriented and elapsed-only:

```text
PT0S
PT1H30M
PT24H
-PT1.5S
PT0.000000001S
```

`Duration / Duration` produces an exact-decimal `Number` ratio where defined. Integer or exact `Number` scaling is allowed only when the resulting nanosecond count is an integer and remains within range. No implicit conversion through binary Float is permitted.

## 6. `Interval`: calendar-relative operations

`Interval` stores three independent components:

- Signed **32-bit months** (years normalize to months).
- Signed **32-bit calendar days** (weeks normalize to days).
- Signed **128-bit elapsed nanoseconds**.

A calendar day (`P1D`) is **not** the same as 24 elapsed hours (`PT24H`). Likewise, one calendar month is not interchangeable with 30 days. HN1 preserves mixed signs instead of silently converting calendar units to elapsed seconds.

```text
P1M             one calendar month
P1D             one calendar day
PT24H           24 elapsed hours, not one calendar day
P1Y2M3DT4H      14 months, 3 calendar days, 4 elapsed hours
P1M-3DT2H       mixed-sign components
```

When applying an interval, HN uses a stable order: **months first, calendar days second, elapsed nanoseconds third**. Month-end arithmetic clamps invalid target days: `2027-01-31 + 1 month = 2027-02-28`.

Intervals compare **structurally**. One year equals twelve months, but one month is not equal to thirty days. Distinct finite intervals generally have **no inherent ordering** without a reference calendar date and (when applicable) time zone. `Duration`, by contrast, has a natural total elapsed-time order.

## 7. Arithmetic and infinities

The most important approved operator relationships are:

| Expression | Result and interpretation |
| --- | --- |
| `TimestampTZ - TimestampTZ` | `Duration`, exact difference between UTC instants |
| `Timestamp - Timestamp` | `Interval`, signed civil difference with zero months |
| `Time - Time` | `Interval`, signed non-wrapping elapsed component |
| `Time ± Duration` | `Time`, wraps on a 24-hour clock |
| `Time ± Interval` | `Time` only if calendar months and days are zero |
| `Timestamp ± Duration` | `Timestamp`, using nominal 86,400-second days |
| `Date ± Duration` | `Timestamp`, beginning from midnight |
| `TimestampTZ ± Duration` | `TimestampTZ`, UTC-timeline arithmetic |
| `TimestampTZ ± Interval` | `TimestampTZ`, calendar-first arithmetic with zone resolution |
| `Duration / Duration` | `Number`, exact-decimal ratio |

For a named-zone `TimestampTZ`, nonzero duration arithmetic calculates the new UTC instant and resolves the new offset using the local time-zone database. An offset-only value retains its fixed numeric offset. **Adding zero** preserves the full original timestamp, including provenance and fractional precision, without requiring a provider lookup.

`Date`, `Timestamp`, `TimestampTZ`, `Duration`, and `Interval` support genuine ±infinity states, distinct from `NULL` and from finite extrema. Indeterminate arithmetic raises structured errors. `Time` has no infinity state, and HN defines no temporal NaN.

## 8. Parsing and formatting

HN1 will provide six dedicated strict native parsers:

```harbour
hn_ParseDate(cText)
hn_ParseTime(cText)
hn_ParseTimestamp(cText)
hn_ParseTimestampTZ(cText, iGapPolicy := HN_TZ_GAP_REJECT)
hn_ParseDuration(cText)
hn_ParseInterval(cText)
```

Each fallible parser has two results: `(value-or-NULL, HNError-or-NULL)`. Regional date strings such as `03/04/2027`, implicit `today`/`now` keywords, year zero, type-inappropriate components, and leap seconds `:60` are rejected rather than interpreted according to an ambient locale. Parsing a `TimestampTZ` with an explicit offset does not require resolving a zone name. Parsing a named-zone-only local timestamp does.

Default human-readable temporal output uses unambiguous ISO-oriented forms. For `TimestampTZ`, the default preserves the offset and optional bracketed name, but the shortest-default fractional formatting is **not by itself a lossless serialization format** because the originally written number of fractional digits participates in strict identity.

### HarbourNova Format Standard (HNFS) formatting

**HNFS stands for HarbourNova Format Standard.** It is HarbourNova's native, picture-based formatting standard for converting values into human-readable `Text`. In HN1, its intended scope includes numeric, date, time, and timestamp presentation. An HNFS picture controls the *textual presentation* of a value; it does not change the stored value, its numeric precision, its underlying UTC instant, or its recorded time-zone information.

HarbourNova's planned `hn_ValueToText()` formatting expansion uses **HNFS** whenever the format picture has no dialect prefix. For migration and interoperability, an explicit prefix selects another formatting convention: `PG:` for **PostgreSQL**, `HB:` for **Harbour (Harbour Legacy)**, `VFP:` for **Visual FoxPro**, and `PY:` for **Python**. Unsupported pictures or type/dialect combinations are errors rather than silently falling back to HNFS.

The approved HNFS date/time token direction includes `yyyy`, `MM` for month, `mm` for minute, `HH` for a 24-hour clock, `hh` for 12-hour time, `ss`, and repeated `S` tokens for up to nine fractional digits. Named weekday/month token casing controls their capitalization, and formatted time-zone presentation does not implicitly change the instant.

Omitted locale means deterministic, invariant English. An explicit locale uses the replaceable CLDR data provider, including official parent-locale inheritance, with strict errors if required data cannot be resolved. HN1 uses **Latin digits `0–9` for all locales**; non-Latin numbering-system output is deferred to HN2. The final full HNFS picture grammar remains under separate design review.

The approved *target* formatter family is:

```harbour
// Two-result API (not the existing phase-one signature)
cText, oError := hn_ValueToText(xValue, cFormat, cLocale, iRoundingMode)

// Single-result convenience APIs
cText := hn_ValueToTextOrThrow(xValue, cFormat, cLocale, iRoundingMode)
cText := hn_ValueToTextOrBlank(xValue, cFormat, cLocale, iRoundingMode)
```

The `OrThrow` variant raises a catchable error for ordinary formatting failures. The `OrBlank` variant returns empty UTF-8 Text for ordinary formatting failures, preserves `NULL` input as `NULL`, and does not suppress fatal errors. These are future HN1 API designs, not currently accepted executable functionality.

## 9. Local IANA/CLDR provider and deterministic behavior

HN1's planned time-zone and localization module is a **versioned native library** in the application's executable directory:

- Windows: `hn_tzdb.dll`
- Linux: `libhn_tzdb.so`

Core owns temporal calculations; the module supplies immutable IANA history, aliases, transitions, Windows-to-IANA mapping, and CLDR localization data. The module loads lazily when required, is shared across Tasks, remains fixed for the process lifetime, and can be replaced between application restarts. No network lookup is required.

A saved `TimestampTZ` stays readable and comparable even if the associated named zone cannot currently be resolved. The recorded UTC instant, numeric offset, name, and precision remain its original facts. A **new** named-zone calculation requiring unavailable rules returns an error; it does not use an undocumented OS fallback.

The current local clock has one deliberately narrow exception: if the system's named zone cannot reliably be identified, HN may record the operating system's actual numeric offset for the sampled instant, without inventing an IANA identifier. UTC clock retrieval needs no named-zone provider.

## 10. Clocks and elapsed timing

The existing HN1 `TimestampTZ` V1 work has accepted APIs for build time and current local/UTC time. The expanded temporal architecture preserves their intent and updates their future precision and offset handling.

Separately, the approved **`ElapsedTimer` system class** measures monotonic elapsed `Duration`, not calendar time or CPU usage. It supports `Start()`, `Stop()`, `Elapsed()`, `Reset()`, `Restart()`, and read-only `IsRunning`.

It starts stopped at zero, can be shared safely across Tasks, and offers two suspend-time policies: `HN_ELAPSED_TIME_EXCLUDE_SUSPEND` (default) and `HN_ELAPSED_TIME_INCLUDE_SUSPEND`. It is not serialized across processes. Exact class constructor syntax is still to be finalized.

## 11. SQL, persistence, and interoperability

Local SQL compares and orders `TimestampTZ` values by their UTC instant. `GROUP BY`, `DISTINCT`, and `UNION` therefore follow chronological instant equality; one original complete source value survives a deduplication/group representative choice, with unspecified tie choice. `UNION ALL` retains all values. A zone library is not necessary simply to compare two already resolved instants.

DataWharf can map one logical HN timestamp to several physical backend fields, conceptually:

```sql
occurred_at              timestamptz
occurred_offset_seconds  integer
occurred_zone_name       text
```

The schema must explicitly describe companion fields. PostgreSQL `timestamptz` alone cannot recover the original offset or named zone; importing only it is an intentionally **lossy** mapping. Correct reconstruction of a fully preserved `TimestampTZ` also requires its fractional precision and any naming-system provenance. PostgreSQL's native timestamp precision is narrower than HN's nanoseconds, and full-range interoperability needs separate checks.

HN1's binary/wire serialization layout and the precise rules for missing or NULL companion fields are **not yet finalized**. Applications should not treat a human-readable rendering as a complete substitute for a designed lossless persistence format.

## 12. Error handling, compatibility, and delivery status

Fallible temporal constructors, parsers, and conversion APIs return a value and a structured `HNError` as **separate return values**. Ordinary validation failures are returned, not silently coerced or converted to `NULL` without explanation. Some language operators can still raise a catchable exception for invalid operations or overflow.

This follows HN's implemented **Multiple Return Values** language feature. Calling a two-result API in an ordinary scalar expression is not equivalent to taking its first result; receivers must match the result count or deliberately use `_` to discard a position.

HN intentionally differs from PostgreSQL in several places: it preserves timestamp provenance, provides nanoseconds, uses `Duration` for instant subtraction, treats distinct finite intervals as unordered, rejects nonexistent local wall times by default, and has stricter native ISO parsing. Those are intentional HN contracts, not claims that PostgreSQL itself behaves identically.

### Status before implementation

**The architectural direction is approved; final implementation-readiness remains subject to specific decisions and verification**, including exact native lossless serialization, `hn_LocalTimeZone()` provider-failure precedence, exact finite boundary validation, final catalog/error identities and native `Duration`/`Interval` construction APIs, and the complete HNFS temporal formatting grammar. Codex implementation should proceed in bounded, separately authorized stages rather than treating the feature as already shipped.

## References

- [PostgreSQL date/time types](https://www.postgresql.org/docs/current/datatype-datetime.html)
- [PostgreSQL date/time functions and arithmetic](https://www.postgresql.org/docs/current/functions-datetime.html)
- [PostgreSQL ambiguous/invalid local times](https://www.postgresql.org/docs/current/datetime-invalid-input.html)
- [Unicode CLDR](https://cldr.unicode.org/)
- [HarbourNova design repository](https://github.com/harbournova/harbournova)

---

*HarbourNova is an evolving software project. This publication describes the HN1 design target, not a promise that every capability is available in the current build.*
