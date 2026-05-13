---
label: HEP001
description: A column-oriented storage layout for tabular data in HDF5, with first-class support for per-column datatypes, chunking, compression, row indexes, and query-accelerating search indexes.
date: 2026-05-12
tags:
  - draft
keywords:
  - HDF5
  - tabular data
  - columnar storage
  - dataframe
authors:
  - name: Aleksandar Jelenak
    affiliation: HDF Group
funding:
  statement: This material is based upon work supported by the U.S. Department of Energy, Office of Science, Office of Fusion Energy Sciences, under Award Number​ DE-SC0024442.​
  id: DE-SC0024442
numbering:
  heading_2: true
  heading_3: true
---

% -----------------------------------------------------------------------------
% Open design points still to resolve:
%   * Does `column-order` become mandatory when the table has more than one
%     column, or always optional?
%   * `_index` and `INDEX_COLUMNS` entries are now defined as local column
%     names within the table group (matching Anndata). Paths into deeper
%     hierarchies are out of scope; the table group is the resolution scope.
%   * For the chunk min/max search index of floating-point columns, should
%     NaN handling be normative (e.g. NaNs never update min/max; a separate
%     `nan_count` field records them)?
% -----------------------------------------------------------------------------

(hep001-title)=
# HEP001: Column-Oriented Tabular Data in HDF5

```{attention}
This document is a **work-in-progress draft**. The data model and attribute names below are intended
to be stable, but normative language (MUST / SHOULD / MAY) will be tightened
during review. Comments and pull requests are welcome on the
[HEP repository](https://github.com/HDFAlliance/heps).
```

```{note} Acknowledgments and disclosures
Development of this document was supported by the U.S. Department of Energy,
Office of Science, Office of Fusion Energy Sciences, under Award Number​ DE-SC0024442.

No contractual or commercial obligations constrain the content of this document.
This work is contributed to the HDF community under the
[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) license.
```

```{tip} Help choose the name for this convention
The proposed convention/format does not yet have an official name. Possible candidates:

- **Pillar**

  Cleanest columnar metaphor. One syllable, no ambiguity, works as a filename stem (`*.pillar`), API (`pillar.open(...)`), or prose ("a Pillar table").

- **Strata**

  Layered, geological. Plays nicely with HDF5's hierarchy and with the idea that each column is its own layer.

- **H5Col**

  Bare-bones and unambiguous. Reads cleanly as a suffix (`.h5col`), package (`h5col`), class (`H5Col.Table`), and spec shorthand.

- **H5DataFrame**

  Explicit, Anndata-aligned. Good if interoperability is the sell; trades brevity for clarity.
```

## Introduction

HDF5 has stored tabular data from its earliest days, and several distinct
idioms have grown up around that use case. HEP001 proposes a **column-oriented
storage layout**, in the spirit of Apache Parquet, Apache Arrow, and Feather,
that lives natively as an HDF5 group and combines cleanly with the
multidimensional array datasets that are HDF5's traditional strength.

### A short overview of tabular data in HDF5

The first adopted idiom is the [HDF5 Table
specification](https://support.hdfgroup.org/documentation/hdf5/latest/_t_b_l_s_p_e_c.html)
that ships with the HDF5 High-Level (`H5TB`) library. A table is a single
one-dimensional dataset whose datatype is an HDF5 compound (record) type,
decorated with attributes such as `CLASS="TABLE"`, `VERSION`, `TITLE`,
`FIELD_0_NAME … FIELD_N_NAME`, `FIELD_0_FILL … FIELD_N_FILL`, and `NROWS`.
Rows of the logical table become elements of the dataset; columns become
fields of the compound datatype. This layout is simple and portable, but it
is fundamentally row-oriented: every row occupies contiguous bytes, every
column shares the same chunking, and changing a single column's datatype,
chunk shape, or compression filter requires rewriting the entire dataset.

The second influential idiom is [PyTables](https://www.pytables.org), a Python
package that has layered a rich query engine on top of HDF5. PyTables likewise
stores a table as a single one-dimensional dataset of a compound type decorated
with its own `CLASS="TABLE"`, `VERSION`, `TITLE`, `FIELD_N_FILL`, `NROWS`, and
`PYTABLES_FORMAT_VERSION` attributes. This adds a rich family of companion
structures for indexing, in-kernel queries, and compression with [Blosc](https://blosc.org/). PyTables
popularized the idea that a useful table format needs more than data bytes:
it needs indexes, metadata, and conventions that let tools reason about the
data.

The third and most recent influence is
[Anndata](https://anndata.readthedocs.io/), which treats a table (_dataframe_)
as an HDF5 group rather than a single compound dataset. Each column of the
dataframe is stored as its own one-dimensional dataset inside that group. The
group carries attributes that tell Anndata how to reassemble the columns into a
dataframe — `encoding-type="dataframe"`, `encoding-version`, `column-order` (a
UTF-8 string array giving the column order), and `_index` (the name of the
column that supplies row labels). Anndata extends this convention with dedicated
encodings for nullable integers, nullable booleans, categoricals, sparse
matrices, and more. This layout is column-oriented, and it is the closest
existing HDF5 practice to what modern analytical engines expect.

### Why columnar, and why now

Row-oriented tables pack every value of every column together. That
packing is ideal for use cases that scan whole rows (appending rows to a
log, reading a few complete records by index) but it imposes three
practical limits:

1. **Uniform chunking and filtering.** Every column in the table shares the
   same chunk shape and the same filter pipeline, because both are properties
   of the single compound dataset. A wide table that mixes a dense float
   column (well-served by shuffle + Zstd) with a high-cardinality string
   column (well-served by a dictionary encoding or Blosc bitshuffle) is
   forced to compromise.
2. **Whole-row I/O for column queries.** Selecting one column out of a
   hundred still reads every column's bytes, because those bytes are
   interleaved within each chunk. Analytical workloads routinely scan a few
   columns of a wide table, and row orientation amplifies I/O proportional
   to row width.
3. **Schema evolution.** Adding or removing a column changes the compound
   datatype of the whole dataset, which in HDF5 terms means rewriting every
   chunk. Columnar layout makes schema evolution a matter of creating or
   deleting a sibling dataset.

Columnar formats such as Parquet, ORC, and Arrow Feather have become the
lingua franca of analytical tabular data precisely because they decouple each
column's physical storage. HEP001 brings the same property to HDF5 while
keeping everything the HDF5 ecosystem already has: a single container
file, portable self-describing metadata, hierarchical groups, and
lossless access to every tool in the HDF5 stack.

A decisive advantage of HDF5 over dedicated columnar formats is that tabular
data does not have to live alone. For example, an HDF5 file can hold:

* a table group with the experiment's per-event observations,
* a multidimensional dataset of raw sensor images addressed by those events,
* a dense array of calibration coefficients indexed by detector channel,
* and any number of nested groups carrying derived products.

Links, object references, and region references can tie rows of the table
to slabs of the image cube without duplicating bytes. Analysts can query
the table to identify events of interest and then dereference the rows into
pixel-space regions in the same file, on the same storage, in a single API.
Columnar tools and array tools meet in the middle.

```{mermaid}
graph TD
  F[("HDF5 File")]
  F --> R[["/ (root group)"]]
  R --> T[["/events (table group, CLASS=COLUMN_TABLE)"]]
  R --> I[("/images (3-D dataset, detector cube)")]
  R --> C[("/calibration (2-D dataset)")]
  T --> c1(["ts (int64)"])
  T --> c2(["energy (float32)"])
  T --> c3(["pixel_ref (object reference)"])
  c3 -. "dereferences to slabs of" .-> I

  classDef rootGroup  fill:#F4ECD4,stroke:#B8A468,stroke-width:1.5px,color:#000
  classDef tableGroup fill:#FFF4D4,stroke:#D4B86A,stroke-width:2px,color:#000
  classDef dataCol    fill:#D4EBF8,stroke:#7FA9D0,stroke-width:1px,color:#000
  classDef otherData  fill:#EDE5DC,stroke:#A89B8B,stroke-width:1px,color:#000

  class F otherData
  class R rootGroup
  class T tableGroup
  class I,C otherData
  class c1,c2,c3 dataCol
```

### Scope and non-goals

HEP001 specifies:

* the structure of a group that holds a column-oriented table,
* the identifying attributes that let software recognize such a group,
* how individual columns are represented as HDF5 datasets,
* how row-label indexes relate columns and the table group,
* how search indexes accelerate queries over column values, including a
  normative chunk-min/max index and framework definitions for sorted-row,
  bitmap, and per-chunk Bloom-filter indexes.

HEP001 *does not* specify:

* a query language or execution engine,
* semantic conventions for domain-specific column units (outside the
  `units` / `units_vocabulary` attribute pair, which is descriptive only),
* how tables are committed, versioned, or synchronized — those are the
  province of the storage layer and, perhaps, future HEPs.


## Conformance

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY**
in this document are to be interpreted as described in [RFC 2119] and
[RFC 8174] when, and only when, they appear in all capitals.

[RFC 2119]: https://datatracker.ietf.org/doc/html/rfc2119
[RFC 8174]: https://datatracker.ietf.org/doc/html/rfc8174

A file, group, or dataset is HEP001-conformant when it satisfies every MUST in
the section that applies to it. A producer is HEP001-conformant when every table
group it writes is conformant; a consumer is conformant when it can read any
conformant table group without data loss.


## Terminology

The following terms are used throughout this specification.

Table group
: An HDF5 group that represents one column-oriented table. Identified by the
`CLASS` attribute (see {ref}`hep001-class`).

(hep001-dset-name)=
Dataset name
: The name of the HDF5 hard link that connects an HDF5 dataset with its parent
HDF5 group.

Column dataset
: An HDF5 dataset of rank 1 that lives directly under a table group and
  represents one column of the table. The dataset's name is the column name.

Row index column
: A column dataset whose name appears in the table group's `INDEX_COLUMNS`
  attribute and which therefore supplies row labels for the table. Row index
  columns are otherwise indistinguishable from any other column dataset; the
  designation is made at the table-group level, not on the column itself.

Row
: A position `i` in a column dataset. Every column dataset in the same
  table group MUST have the same length, so the same `i` refers to the
  same logical row everywhere.

Search index dataset
: An HDF5 dataset stored under the `SEARCH_INDEXES` child group of a table
  group that accelerates queries over one or more column datasets. Each kind
  of search index is specified in {ref}`hep001-search-indexes`.


## Data model overview

A HEP001 table is an HDF5 **group** whose direct children are the table's
columns (one or more of which MAY be designated as row index columns via
the table group's `INDEX_COLUMNS` attribute) and, optionally, a reserved
`SEARCH_INDEXES` subgroup that holds query-acceleration structures.

```{mermaid}
graph TD
  TG[["/my_table (table group)<br/>CLASS=COLUMN_TABLE, VERSION=1.0<br/>INDEX_COLUMNS = [row_id]"]]
  TG --> c0(["row_id (1-D column dataset,<br/>row index)"])
  TG --> c1(["ts (1-D column dataset)"])
  TG --> c2(["energy (1-D column dataset)"])
  TG --> c3(["label (1-D column dataset,<br/>categorical)"])
  TG --> c4[("label__CATEGORIES (1-D dataset)")]
  TG --> SI[["SEARCH_INDEXES (group)"]]
  SI --> mm[("ts__chunk_minmax (1-D compound)")]
  SI --> sr[("energy__sorted_rows (1-D uint64)")]
  SI --> bm[("label__bitmap (1-D compound)")]
  SI --> bf[("ts__chunk_bloom (2-D uint8)")]

  classDef tableGroup fill:#FFF4D4,stroke:#D4B86A,stroke-width:2px,color:#000
  classDef siGroup    fill:#E5DAF5,stroke:#9D85C5,stroke-width:2px,color:#000
  classDef dataCol    fill:#D4EBF8,stroke:#7FA9D0,stroke-width:1px,color:#000
  classDef indexCol   fill:#FAD4E1,stroke:#D08FB0,stroke-width:1px,color:#000
  classDef catData    fill:#FAE0D4,stroke:#D09F90,stroke-width:1px,color:#000
  classDef siData     fill:#D7F0D8,stroke:#7CB78A,stroke-width:1px,color:#000

  class TG tableGroup
  class SI siGroup
  class c0 indexCol
  class c1,c2,c3 dataCol
  class c4 catData
  class mm,sr,bm,bf siData
```

The rest of this document specifies each building block: the table group
({ref}`hep001-table-group`), column datasets ({ref}`hep001-columns`), row
index columns ({ref}`hep001-indexes`), and the four kinds of search-index
dataset ({ref}`hep001-search-indexes`).

(hep001-table-group)=
## The table group

(hep001-class)=
### Identification — the `CLASS` attribute

Every HEP001 table group MUST carry an attribute named `CLASS` with:

* datatype: null-terminated 13-bytes fixed-length ASCII string
  (exactly the length of the value below plus a NUL byte),
* shape: scalar,
* value: `COLUMN_TABLE`.

A consumer MUST identify a group as a HEP001 table group by, and only by, the
presence of a `CLASS` attribute whose string value equals `COLUMN_TABLE`. A
producer MUST NOT write `CLASS="COLUMN_TABLE"` attribute on any group that does
not satisfy the rest of this specification.

```{note}
`CLASS` uses fixed-length ASCII to match the long-standing HDF5 High-Level API
conventions. All other string attributesin this spec are UTF-8 unless explicitly
stated otherwise.

HDF5 fixed-length strings are byte-length-bounded. Producers MUST size
each fixed-length UTF-8 attribute with enough bytes to hold the longest
value they intend to write (a single UTF-8 code point can take up to
four bytes). Values MUST be null-terminated or null-padded per standard
HDF5 string conventions.
```

### The `VERSION` attribute

Every HEP001 table group MUST carry a scalar, fixed-length ASCII attribute
named `VERSION` whose value is the HEP001 revision the table conforms to.
For this revision the value is `"1.0"`. HEP001 consumers MUST interpret these values according to the [SemVer] specification.

[SemVer]: https://semver.org/

### Optional table group attributes

The table group MAY carry the following attributes.

`TITLE`
: Scalar fixed-length UTF-8 string. Human-readable title of the table,
  mirroring HDF5 Table and PyTables. Purely descriptive.

`description`
: Scalar fixed-length UTF-8 string. Free-text description of the table's
  contents. Longer and richer than `TITLE`; intended for documentation
  viewers.

`column-order`
: One-dimensional fixed-length UTF-8 string attribute whose elements are
  the names of the column datasets in their logical order. When present,
  it fully determines the column order presented to users; when absent,
  the logical column order is implementation-defined. Producers SHOULD
  write `column-order` whenever a table has more than one column. The
  attribute name uses a hyphen (not an underscore) to match Anndata.

`INDEX_COLUMNS`
: One-dimensional fixed-length UTF-8 string attribute whose elements are the
  names of the column datasets that serve as row labels for this table, in
  hierarchical order from outermost to innermost level. For example, a table
  indexed by `(donor_id, sample_id, cell_id)` writes `INDEX_COLUMNS =
  ["donor_id", "sample_id", "cell_id"]`. An empty array or absent attribute
  means the table has no row labels and rows are positional only. Every name
  listed MUST resolve to a column dataset that is a direct child of the table
  group. Row index columns apply to the table as a whole — every column in the
  table is labeled by them. See {ref}`hep001-indexes`.

`_index`
: Scalar fixed-length UTF-8 string. The name of the column that supplies
  the primary row labels for this table (for example, `"row_id"`).
  Matches Anndata's `_index` exactly. When both `INDEX_COLUMNS` and
  `_index` are present, `_index` MUST equal `INDEX_COLUMNS[0]`. See
  {ref}`hep001-anndata`.

`units_vocabulary`
: Scalar fixed-length UTF-8 string. Names the vocabulary or authority that
  interprets `units` strings on columns of this table — for example
  `"UDUNITS-2"`, `"UCUM"`, `"QUDT"`, or a URL. When present on the table
  group, it applies as a default to every column whose own `units_vocabulary`
  is absent.

`encoding-type`
: Scalar fixed-length UTF-8 string. Optional. When set to `"dataframe"`
  (and `encoding-version` to `"0.2.0"` or another value the producer
  chooses), it advertises the table group as an Anndata DataFrame so that
  Anndata readers can consume it. See {ref}`hep001-anndata`.

(table-group-placement)=
### Placement of the table group

The table group MAY be located anywhere in an HDF5 file's hierarchy. In the
simplest case the root group of the file is the table group — the file
holds one table and nothing else — and all columns live at the top level.
Tables MAY also be nested under named groups to organize many tables, or
sited beside multidimensional array datasets that they describe.

```{mermaid}
graph TD
  subgraph Flat
    R1[["/ (root = table group)"]]
    R1 --> c1a(["col_a"])
    R1 --> c1b(["col_b"])
    R1 --> SI1[["SEARCH_INDEXES"]]
  end
  subgraph Nested
    R2[["/ (root)"]]
    R2 --> G1[["/experiments/"]]
    G1 --> T1[["run_042 (table group)"]]
    G1 --> T2[["run_043 (table group)"]]
    R2 --> IMG[("/images (3-D dataset)")]
  end

  classDef rootGroup  fill:#F4ECD4,stroke:#B8A468,stroke-width:1.5px,color:#000
  classDef tableGroup fill:#FFF4D4,stroke:#D4B86A,stroke-width:2px,color:#000
  classDef siGroup    fill:#E5DAF5,stroke:#9D85C5,stroke-width:2px,color:#000
  classDef dataCol    fill:#D4EBF8,stroke:#7FA9D0,stroke-width:1px,color:#000
  classDef otherData  fill:#EDE5DC,stroke:#A89B8B,stroke-width:1px,color:#000

  class R1,T1,T2 tableGroup
  class R2,G1 rootGroup
  class SI1 siGroup
  class c1a,c1b dataCol
  class IMG otherData
```

(hep001-self-contained)=
### Self-contained contents

A table group is the complete representation of a single column-oriented
table. Every HDF5 object — group, dataset, named datatype, or link —
that is a descendant of the table group MUST exist solely in service of
the data model defined by this revision of HEP001.

For this revision of HEP001, the only objects permitted anywhere in the
HDF5 hierarchy below a table group are:

* column datasets ({ref}`hep001-columns`), including those designated
  as row index columns by the table group's `INDEX_COLUMNS` attribute
  ({ref}`hep001-indexes`);
* categories datasets backing categorical columns
  ({ref}`hep001-categoricals`);
* the reserved `SEARCH_INDEXES` subgroup, the search-index datasets it
  contains, and their accompanying values datasets
  ({ref}`hep001-search-indexes`).

Any descendant of a table group that does not match one of the
categories above is non-conformant under this revision of HEP001.

This whitelist may be extended by future revisions of HEP001 or by
future HEPs that build on it; any such extension MUST itself exist
solely in service of the data model. A producer following this revision
MUST NOT extend the whitelist at its own discretion.

In particular, a producer MUST NOT place under a table group:

* unrelated tables, including other HEP001 table groups;
* multidimensional array datasets that are not derived from, or
  exclusively bound to, the columns of this table;
* user-defined subgroups for provenance, schema, derived statistics,
  or any other auxiliary content not enumerated above.

Producers that wish to colocate such content with a table MUST do so by
placing it as a sibling of the table group, or under an unrelated parent
group elsewhere in the file. Object references, region references, and
external links MAY be used to associate the table's columns with that
external content without violating this rule.

A consumer that encounters a descendant of a table group not described
by this revision SHOULD emit a diagnostic identifying the offending
HDF5 path. A consumer operating in strict-conformance mode MAY refuse
to process the table; a consumer operating in lenient mode MAY ignore
the offending descendant and continue.

Consumers MAY treat the table group as a closed, self-describing unit:
copying, moving, renaming, deleting, or versioning the table group is
guaranteed to act on the entire table and on nothing else.

```{note}
This restriction applies only *below* the table group. The table group
itself MAY live anywhere in the file, alongside arbitrary other HDF5
content — see {ref}`above <table-group-placement>`. The two rules are
complementary: a HEP001 table travels as a self-contained subtree, but
that subtree can be hosted next to any other HDF5 object the producer
chooses.
```

(hep001-columns)=
## Column datasets

### Required properties

Each column of a HEP001 table MUST be stored as an HDF5 dataset that:

* is a direct child of the table group,
* has rank 1 shape,
* has the same length (number of elements) as every other column dataset
  in the same table group.

The {ref}` name <hep001-dset-name>` of the HDF5 dataset is the column name. Any
name acceptable as an HDF5 link name (UTF-8, excluding `/` and NUL) is
permitted. Producers MUST NOT use any HEP001 reserved name
({ref}`hep001-reserved-names-list`) as a column name. Producers SHOULD also
avoid names that begin with an underscore, which are reserved for
Anndata-aligned attribute names (`_index`) and may be claimed by future HEPs.

### Datatypes

The datatype of a column dataset MAY be any HDF5 datatype. Consumers that
encounter a datatype they do not recognize SHOULD expose the column's raw datatype
instead of quietly ignoring it.

Two caveats apply:

1. Compound datatypes at the column level are permitted, but a table group
   is not a mechanism for storing a nested row-oriented table. When a
   compound-typed column is used, its fields MUST be logically atomic.
2. Variable-length datatypes (e.g. variable-length UTF-8 strings, ragged numeric
   arrays) are permitted but generally discouraged, as their storage
   characteristics are poorly suited to columnar access patterns. Producers
   SHOULD prefer fixed-length equivalents where possible.

### Chunking and filters

Because each column is its own HDF5 dataset, each column MAY independently
select:

* chunk shape (typically `(N,)` for `N` rows per column chunk),
* dataset creation properties such as fill value, allocation time, and
  track-times,
* the filter pipeline.

This per-column flexibility is the core motivation for HEP001 and is
normative: producers MUST NOT require columns to share chunk shape or
filters. Consumers MUST treat each column's storage layout independently.

(fill-vals)=
### Missing values (fill values)

A column dataset's HDF5 fill value identifies rows whose value is missing.
Producers MUST set the dataset's fill value explicitly via the dataset creation
property list (`H5Pset_fill_value`), placing the dataset into the
`H5D_FILL_VALUE_USER_DEFINED` state, and MUST choose a fill value that lies
outside the column's logical value range. A producer MAY declare that range
explicitly with the two attributes `valid_min` and `valid_max` on the column
dataset (each a scalar of the column's element datatype). If present, the chosen
fill value MUST lie strictly outside `[valid_min, valid_max]`. Consumers MUST
retrieve the fill value (`H5Dget_fill_value`) and identify missing rows with the
equality test `value == fill_value`. The same applies even to columns that
contain no missing values — in that case no row will compare equal to the fill
value, and the column simply has no missing rows.

A column's fill value MUST NOT be a `NaN` bit pattern. IEEE 754 specifies that
`NaN != NaN`, so the equality test `value == fill_value` would always return
false against a `NaN` fill and consumers would identify zero rows as missing
regardless of the data. Producers needing a sentinel for a floating-point
column MUST use a non-`NaN`, non-infinite value — for example, the recommended
`9.9692099683868690e+36` from {numref}`§%s <fill-table>` below.

For producers that have no domain-specific constraint forcing a different
choice, the table below lists recommended fill values. These values are
exactly representable in their respective datatypes, far outside any
plausible data range.

(fill-table)=

| HDF5 datatype | Recommended fill value   | Hex (canonical bit pattern) |
|---------------|--------------------------|-----------------------------|
| `int8`        | `-127`                   | `0x81`                      |
| `int16`       | `-32767`                 | `0x8001`                    |
| `int32`       | `-2147483647`            | `0x80000001`                |
| `int64`       | `-9223372036854775807`   | `0x8000000000000001`        |
| `uint8`       | `255`                    | `0xFF`                      |
| `uint16`      | `65535`                  | `0xFFFF`                    |
| `uint32`      | `4294967295`             | `0xFFFFFFFF`                |
| `uint64`      | `18446744073709551615`   | `0xFFFFFFFFFFFFFFFF`        |
| `float32`     | `9.9692099683868690e+36` | `0x7CF00000`                |
| `float64`     | `9.9692099683868690e+36` | `0x479E000000000000`        |
| fixed string  | `""`                     | n/a                         |
| vlen string   | `""`                     | n/a                         |
| enumeration   | `MISSING`                | n/a                         |

The signed-integer sentinels are `INT*_MIN + 1` rather than `INT*_MIN`
itself, leaving a one-value safety margin against operations that land on
the type's minimum (e.g., `abs(INT*_MIN)` is undefined behavior in C).
The unsigned sentinels are the type's maximum.

For datatypes not in the table:

* `float16` is too narrow for a generic sentinel;
* Boolean (1-bit) datatype cannot represent a missing sentinel
  alongside their two valid values. Producers MUST widen such columns to
  `uint8` and use a value greater than `1` (typically `2`).

Producers whose column domain includes any of the recommended sentinels
above MUST set `valid_min` and/or `valid_max` to declare the column's
actual range, and MUST choose a fill value outside that declared range.

```{note}
This is a conscious divergence from Anndata's nullable-integer and
nullable-boolean encodings, which add a companion boolean mask. A future
HEP MAY introduce explicit masks; until then, HEP001 relies on fill
values alone.
```

### Column attributes

A column dataset MAY carry any of the following attributes.

`SEARCH_INDEX_LIST`
: One-dimensional array of HDF5 object references. Each reference points
  to a search-index dataset (see {ref}`hep001-search-indexes`) in the
  `SEARCH_INDEXES` subgroup that accelerates queries on this column.

`CATEGORIES`
: Scalar HDF5 object reference. Used only for categorical columns. Points
  to the dataset that holds the categorical values (see
  {ref}`hep001-categoricals`).

`units`
: Scalar fixed-length UTF-8 string. Physical units of the column's values.
  Absence of `units` implies dimensionless data. Units are interpreted
  according to `units_vocabulary` (on the column, or inherited from the
  table group).

`units_vocabulary`
: Scalar fixed-length UTF-8 string. Identifies the units vocabulary that
  interprets `units`. When present on a column, it overrides the table
  group's `units_vocabulary` for that column. MAY be a short name
  (`"UDUNITS-2"`) or a URL.

`valid_min` / `valid_max`
: Scalar attributes of the same datatype as the column, specifying the minimum
and maximum range of the column's values.

`description`
: Scalar fixed-length UTF-8 string. Plain-text description of the column's
  contents, provenance, or semantics.

(hep001-categoricals)=
### Categorical columns

A categorical column stores integer codes that index into a separate categories
dataset holding the actual values.

A categorical column MUST:

1. Have an integer datatype (any width, signed or unsigned). The column's
   missing value (the dataset's fill value) denotes a missing category.
2. Carry a scalar `CATEGORIES` attribute whose value is an HDF5 object
   reference pointing to the categories dataset.

The categories dataset MUST:

1. Live directly under the same table group.
1. Have rank 1, and any datatype appropriate to the category values.

The categories dataset MAY:

1. Carry a scalar UTF-8 string attribute `encoding-type` with value
   `"categorical"` (matching Anndata).
1. Carry a scalar boolean attribute `ordered` (matching Anndata's ordered
   categoricals). Producers MUST set `ordered` to true exactly when the
   order of entries in the categories dataset is semantically meaningful.

The categories dataset MAY appear in the table group's `column-order`,
but consumers that present the table as a dataframe SHOULD treat it as
metadata of its linked column rather than as a standalone column.

```{mermaid}
graph LR
  col(["label<br/>(int8)<br/>CATEGORIES &rarr; label__CATEGORIES"])
  cat[("label__CATEGORIES<br/>(fixed UTF-8)<br/>encoding‑type=&quot;categorical&quot;<br/>ordered=false")]
  col -- object ref --> cat

  classDef dataCol fill:#D4EBF8,stroke:#7FA9D0,stroke-width:1px,color:#000
  classDef catData fill:#FAE0D4,stroke:#D09F90,stroke-width:1px,color:#000

  class col dataCol
  class cat catData
```

(hep001-indexes)=
## Row index columns

A **row index column** is a column dataset whose name appears in the table
group's `INDEX_COLUMNS` attribute (see {ref}`hep001-table-group`). Row index
columns supply row labels for the table as a whole — every column in the table
is labeled by every row index column. They are ordinary column datasets in every
other respect, and they SHOULD also appear in `column-order` like any other
column.

### Hierarchy

When `INDEX_COLUMNS` contains more than one name, the order is the
row-label hierarchy from outermost to innermost level. For example,
`INDEX_COLUMNS = ["donor_id", "sample_id", "cell_id"]` declares a
three-level row index in which `donor_id` is the outermost grouping
and `cell_id` is the innermost row identifier.

### Typical uses

* **Single row index.** The common case: one column (often a string
  of sample IDs, or an unsigned-integer ordinal) supplies row labels
  for the entire table.
* **Hierarchical row index.** A small number of tables have a
  meaningful row-label hierarchy — for example, donor → sample → cell
  in single-cell genomics, or year → quarter → ticker in financial
  time series. `INDEX_COLUMNS` lists the level columns in order.
* **No row index.** A table whose rows are positional only (e.g., an
  append-only log identified by position) MAY omit `INDEX_COLUMNS` or
  set it to an empty array. Consumers MUST treat the table as having
  positional row identifiers in that case.

```{mermaid}
graph TD
  TG[["table group<br/>INDEX_COLUMNS = [donor_id, sample_id]"]]
  TG --> c1(["donor_id<br/>(column, outer index level)"])
  TG --> c2(["sample_id<br/>(column, inner index level)"])
  TG --> c3(["energy<br/>(data column)"])
  TG --> c4(["label<br/>(data column)"])

  classDef tableGroup fill:#FFF4D4,stroke:#D4B86A,stroke-width:2px,color:#000
  classDef dataCol    fill:#D4EBF8,stroke:#7FA9D0,stroke-width:1px,color:#000
  classDef indexCol   fill:#FAD4E1,stroke:#D08FB0,stroke-width:1px,color:#000

  class TG tableGroup
  class c1,c2 indexCol
  class c3,c4 dataCol
```

(hep001-search-indexes)=
## Search indexes

Search indexes accelerate queries over column values. They do not change the
logical table; they are derivative, recomputable data. A conformant consumer MAY
ignore any or all search indexes and still return correct answers, only more
slowly.

### The `SEARCH_INDEXES` group

A table group MAY contain a direct child group named `SEARCH_INDEXES`. When
present, it MUST hold every search-index dataset for the table, and no other
objects. It carries no required attributes of its own. Search-index datasets in
`SEARCH_INDEXES` MAY have any name.

### Linking columns to search indexes

Each column dataset that benefits from a search index MUST reference
that search index from its own `SEARCH_INDEX_LIST` attribute — a 1-D
array of HDF5 object references to search-index datasets in the
`SEARCH_INDEXES` subgroup. The column-side attribute is the only
linkage; search-index datasets do not carry a back-pointer to the
columns they accelerate. To determine the column that a given
search-index dataset applies to, scan the column datasets of the table
group and identify the one whose `SEARCH_INDEX_LIST` references that
search-index dataset.

```{mermaid}
graph LR
  C1(["ts (column)<br/>SEARCH_INDEX_LIST &rarr; ts__chunk_minmax"])
  C2(["energy (column)<br/>SEARCH_INDEX_LIST &rarr; energy__sorted_rows"])
  SI1[("ts__chunk_minmax<br/>(in SEARCH_INDEXES)")]
  SI2[("energy__sorted_rows<br/>(in SEARCH_INDEXES)")]
  C1 -->|"object ref"| SI1
  C2 -->|"object ref"| SI2

  classDef dataCol fill:#D4EBF8,stroke:#7FA9D0,stroke-width:1px,color:#000
  classDef siData  fill:#D7F0D8,stroke:#7CB78A,stroke-width:1px,color:#000

  class C1,C2 dataCol
  class SI1,SI2 siData
```

(common-idx-attrs)=
### Common per-index attributes

Every search-index dataset MUST carry a scalar fixed-length ASCII
attribute `KIND` whose value is one of the strings defined below:

| `KIND` value         | Purpose                                     | Defined in                   |
|----------------------|---------------------------------------------|------------------------------|
| `CHUNK_MINMAX`       | Per-chunk min and max of a numeric column   | {numref}`§%s <chunk-minmax>` |
| `SORTED_ROWS`        | Row-position permutation of a column        | {numref}`§%s <sorted-row>`   |
| `BITMAP`             | Per-value bitmap of a low-cardinality col.  | {numref}`§%s <bitmap>`       |
| `CHUNK_BLOOM`        | Per-chunk Bloom filter of a column          | {numref}`§%s <bloom>`        |

Future HEPs MAY register additional `KIND` values. Consumers MUST treat
unknown `KIND` values as "ignore this search index".


(chunk-minmax)=
### Chunk min/max search index (`KIND = CHUNK_MINMAX`)

A chunk min/max index accelerates range and equality predicates over a
numeric column by letting the engine skip chunks whose value range does
not overlap the predicate.

**Shape:** The search-index dataset is 1-D with length equal to the number of
chunks in the source column dataset:

* `ceil(column_length / chunk_length)` for a chunked column,
* `1` for a contiguous column.

**Datatype:** An HDF5 compound datatype with the following fields, in
declaration order:

* `min` — same datatype as the source column's element type.
* `max` — same datatype as the source column's element type.
* `nan_count` — `uint64`. The number of NaN values in the chunk. For
  non-floating-point columns this field MUST be present and set to 0.
* `fill_count` — `uint64`. The number of elements in the chunk that equal
  the column's HDF5 fill value (i.e. count of "missing" under {numref}`§%s <fill-vals>`).
* `n` — `uint64`. The number of logical rows covered by this chunk. Because the
  last chunk may be partially full, `n` is strictly less than or equal to the
  column's chunk length.

**Semantics:** For floating-point columns, `min` and `max` MUST ignore
`NaN` values — if every value in the chunk is `NaN`, `min` and `max` MUST be
set to the column's fill value and `nan_count` MUST equal `n`. For
integer and other orderable types, `min` and `max` are the ordinary
ordering extrema over the chunk's values, treating fill-value elements
as missing (and excluded from the ordering) when `fill_count > 0`.

**Applicability:** Each `CHUNK_MINMAX` search-index dataset applies to
exactly one column, identified by the column whose `SEARCH_INDEX_LIST`
references it. A producer MAY build separate `CHUNK_MINMAX` indexes
for several columns, but a single search-index dataset MUST NOT cover
multiple columns because chunks of different columns are independent.

**Additional attributes on the search-index dataset:**

* `KIND` — `"CHUNK_MINMAX"` (see {numref}`§%s <common-idx-attrs>`).

**Query pattern:** To evaluate a predicate `col BETWEEN lo AND hi`:

```{mermaid}
flowchart TD
  A["Predicate: col BETWEEN lo AND hi"]
  A --> B["Open col__chunk_minmax"]
  B --> C{"For each row i of the index"}
  C -->|"max below lo OR min above hi"| D["Skip chunk i"]
  C -->|"otherwise"| E["Read chunk i of col"]
  E --> F["Apply predicate to chunk values"]
  D --> G["Next i"]
  F --> G
  G --> C
```


(sorted-row)=
### Sorted-row permutation index (`KIND = SORTED_ROWS`)

A sorted-row index stores row positions in sorted order of a column's
values, enabling binary search and range scans without reading the full
column.

**Shape:** 1-D, length equal to the column length.

**Datatype:** An unsigned integer wide enough to address every row of the
source column (typically `uint64`).

**Semantics:** Element `i` of the index is the row position `r` such
that, under the ordering defined below, the `i`-th rank of the column's
values lives at row `r`. Ties between rows with identical values MUST
be broken by increasing `r`, so the permutation is total and
deterministic.

**Ordering:** A `SORTED_ROWS` index MUST only be built over a column
whose HDF5 datatype has a sorting order defined below:

* **Signed and unsigned integers** (any width): standard arithmetic
  order.
* **Floating-point values** (`float16`, `float32`, `float64`):
  IEEE 754 numerical order over finite values and the two infinities.
  Negative zero MUST compare equal to positive zero. `NaN` values are
  not ordered; rows whose value is `NaN` MUST be placed at the end of
  the permutation, in increasing `r` order, and the `nan_tail_length`
  attribute MUST record the count. The `NaN` tail and the fill tail
  are disjoint: {numref}`§%s <fill-vals>` forbids `NaN` as a fill value,
  so no row can simultaneously satisfy "value is NaN" and
  "value equals fill."
* **Boolean values:** `false` (`0x00`) sorts before `true` (`0x01`).
* **Fixed- and variable-length strings:** lexicographic comparison over the
  UTF-8 byte sequence — i.e., **byte-wise**, with no byte-order mark and no
  Unicode normalization (no NFC, NFD, NFKC, or NFKD conversion is applied). For
  HDF5 fixed-length strings, the column's storage-padding bytes — NUL bytes
  (`0x00`) for `H5T_STR_NULLTERM` and `H5T_STR_NULLPAD`, space bytes (`0x20`)
  for `H5T_STR_SPACEPAD` — MUST be stripped from the trailing end of each string
  before comparison, so that the same logical string sorts identically whether
  stored fixed- or variable-length. Strings whose declared HDF5 character set is
  `H5T_CSET_ASCII` MUST be ordered as if they were UTF-8 (ASCII is a strict
  subset). HEP001 does **not** specify locale-sensitive collation (e.g., POSIX
  `strcoll`, ICU UCA); byte-wise comparison is the only conformant rule because
  it is deterministic across implementations, platforms, and runtime locales.
* **Opaque (`H5T_OPAQUE`) values:** byte-wise lexicographic comparison
  over the raw bytes of the value; the opaque tag, if any, is not
  part of the comparison.
* **Enum datatypes:** ordered by the underlying integer codes, using
  the integer rule above.

The following datatypes do not have a defined sorting order, so a
`SORTED_ROWS` index MUST NOT be built on them:

* HDF5 object and region references (no canonical order over
  references);
* compound datatypes (a future HEP MAY register a canonical
  multi-field ordering);
* array and variable-length-array datatypes.

Rows whose value equals the column's fill value
({numref}`§%s <fill-vals>`) MUST appear immediately before the NaN
tail (if any), in increasing `r` order, and the `fill_tail_length`
attribute MUST record the count. For datatypes that have no NaN
concept, the NaN tail is empty (`nan_tail_length = 0`) and the
fill-tail rows appear at the very end of the permutation.

**Additional attributes:**

* `KIND` — `"SORTED_ROWS"`.
* `nan_tail_length`, `fill_tail_length` — scalar `uint64`. Both MUST be
  present; either MAY be 0.
* `ordered` — scalar boolean. MUST be true for `SORTED_ROWS`; reserved
  for future indexes that permit partial orderings.

**Applicability:** Each `SORTED_ROWS` search-index dataset applies to
exactly one column, identified by the column whose `SEARCH_INDEX_LIST`
references it. The column's HDF5 datatype MUST have a HEP001-defined
order, as enumerated under *Ordering* above; otherwise no
`SORTED_ROWS` index is permitted.


(bitmap)=
### Bitmap index (`KIND = BITMAP`)

A bitmap index accelerates equality predicates on low-cardinality
columns.

**Shape:** 2-D of shape `(K, ceil(N / 8))`, where `K` is the number of
distinct values (or categories) indexed and `N` is the column length.

**Datatype:** `uint8`. Bit `r % 8` (where bit 0 is the byte's least
significant bit) of byte `r / 8` of row `k` is set if the column's value
at row `r` equals the `k`-th indexed value. Because the storage element
is `uint8`, HDF5 performs no byte swapping on read or write — the bytes
on disk are exactly the bytes the producer wrote.

**Accompanying values dataset:** A sibling 1-D dataset, under `SEARCH_INDEXES`,
holds the `K` indexed values in the same datatype as the source column. Its name
is linked from the bitmap via a scalar `VALUES` object-reference attribute.

**Additional attributes on the bitmap dataset:**

* `KIND` — `"BITMAP"`.
* `VALUES` — scalar HDF5 object reference to the values dataset.
* `ordered` — scalar boolean. When `true`, the entries in the values
  dataset (linked via `VALUES`) are listed in a semantically meaningful
  order, for example, a numerically-sorted set of distinct values or
  an ordinal category sequence such as `["low", "medium", "high"]`.
  The bitmap's `k`-th row corresponds to the `k`-th entry of the values
  dataset under that order. When `false` or absent, the order of the
  values dataset is arbitrary (typically insertion order) and consumers
  MUST NOT infer any semantic ordering from it.

**Applicability:** Each `BITMAP` search-index dataset applies to
exactly one column, identified by the column whose `SEARCH_INDEX_LIST`
references it.


(bloom)=
### Per-chunk Bloom filter index (`KIND = CHUNK_BLOOM`)

A per-chunk Bloom filter accelerates equality predicates on
high-cardinality columns by giving a fast negative answer for chunks
that provably do not contain the queried value.

**Shape:** 2-D of shape `(n_chunks, m_bytes)`, where `n_chunks` is the
number of chunks in the source column and `m_bytes` is the Bloom-filter
byte length per chunk (constant across chunks).

**Datatype:** `uint8`. Each row is the packed bit array of one chunk's
Bloom filter. Because the storage element is `uint8`, HDF5 performs no
byte swapping on read or write — the bytes on disk are exactly the
bytes the producer wrote.

**Bit packing:** Filter bit `g` (where `0 ≤ g < m_bits`) is stored at
bit position `g % 8` (where bit 0 is the byte's least significant bit)
of byte `g / 8` of the chunk's row. Equivalently, setting filter bit
`g` is `row[g >> 3] |= (uint8_t)1 << (g & 7)`. `m_bits` MUST be a
multiple of 8, so that `m_bytes = m_bits / 8` is exact.

**Hash scheme:** HEP001 prescribes — for interoperability — Kirsch–Mitzenmacher
double hashing `h_i(x) = (h_a(x) + i * h_b(x)) mod (8 * m_bytes)` for `i = 0 … k
− 1`. Both `h_a` and `h_b` come from a single invocation of
`MurmurHash3_x64_128` (the 128-bit, 64-bit-tuned variant of [MurmurHash3]) over
the value's *canonical byte representation* (see below), seeded with the value
of the `seed` attribute. The low 64 bits of the 128-bit output become `h_a`; the
high 64 bits become `h_b`. The number of hash functions `k` is stored as an
attribute.

[MurmurHash3]: https://en.wikipedia.org/wiki/MurmurHash#MurmurHash3

**Canonical byte representation:** Bloom-filter hashes MUST be computed
over a column value's *canonical byte representation*, defined here.
The canonical form is independent of how the column is stored on disk:
a column written in big-endian MUST be transcoded to its canonical form
before hashing, and a consumer evaluating a query MUST canonicalize the
query value identically before testing the filter.

* **Signed integers** (`int8`, `int16`, `int32`, `int64`): the value's
  two's-complement representation in the column's storage width, packed
  **little-endian**. An `int8` is one byte; an `int64` is exactly eight.
* **Unsigned integers** (`uint8` … `uint64`): the value's unsigned
  representation in the column's storage width, packed **little-endian**.
* **Floating-point values** (`float16`, `float32`, `float64`): the
  value's IEEE 754 binary encoding at the column's storage width, packed
  **little-endian**. NaN values MUST NOT be inserted into a Bloom filter
  — different NaN bit patterns would yield different filter bits and
  break interoperability — and consumers MUST NOT query for NaN against
  a `CHUNK_BLOOM` index. Producers MUST normalize negative zero to
  positive zero before hashing.
* **Boolean values:** a single byte, `0x00` for false or `0x01` for true.
* **Fixed- and variable-length strings:** the string's **UTF-8** byte sequence,
  with **no byte-order mark** and **no Unicode normalization** (no NFC, NFD,
  NFKC, or NFKD conversion is applied). For HDF5 fixed-length strings, the
  column's storage-padding bytes — NUL bytes (`0x00`) for `H5T_STR_NULLTERM` and
  `H5T_STR_NULLPAD`, space bytes (`0x20`) for `H5T_STR_SPACEPAD` — MUST be
  stripped from the trailing end of each string before hashing, so that the same
  logical string hashes identically regardless of the column's declared padding
  mode. Strings whose declared HDF5 character set is `H5T_CSET_ASCII` MUST be
  hashed as if they were UTF-8 (ASCII is a strict subset).
* **Opaque (`H5T_OPAQUE`) values:** the raw bytes of the value in column
  order; the opaque tag, if any, is not part of the hashed bytes.
* **HDF5 object and region references:** out of scope for this revision.
  Producers MUST NOT build a `CHUNK_BLOOM` index over a reference-typed
  column.
* **Compound, enum, array, and variable-length-array datatypes:** out
  of scope for this revision. Producers MUST NOT build a `CHUNK_BLOOM`
  index over composite-typed columns. A future HEP MAY register
  canonical encodings for these cases.

**Additional attributes:**

* `KIND` — `"CHUNK_BLOOM"`.
* `k` — scalar `uint16`. The number of hash functions.
* `m_bits` — scalar `uint64`. Equal to `8 * m_bytes`; stored explicitly
  for clarity.
* `hash_family` — scalar fixed-length ASCII string; for this revision
  MUST be `"murmur3_x64_128_double"`.
* `seed` — scalar `uint64`. The seed passed to `MurmurHash3_x64_128`.
  Default `0`. Producers MAY change the seed to mitigate adversarial
  inputs; consumers MUST read the seed from this attribute rather than
  assuming `0`.

**Applicability:** Each `CHUNK_BLOOM` search-index dataset applies to
exactly one column, identified by the column whose `SEARCH_INDEX_LIST`
references it. The column's HDF5 datatype MUST belong to one of the
kinds enumerated under *Canonical byte representation*; otherwise no
`CHUNK_BLOOM` index is permitted.

```{note}
Bloom-filter interoperability requires producers and consumers to agree
byte-for-byte on hashing input. HEP001 freezes both the hash family and
the canonical encoding; a future revision MAY register alternative
`hash_family` values or canonical encodings for additional datatypes.
```

## Consistency requirements

A conformant table group satisfies all of the following at all times:

1. Every column dataset (including row index columns) in the same table
   group MUST have the same length.
2. Every dataset in the `SEARCH_INDEXES` subgroup MUST carry a `KIND`
   attribute defined by this HEP (or a future, registered HEP).
3. Every reference in a column's `SEARCH_INDEX_LIST` MUST resolve to a
   search-index dataset under the table group's `SEARCH_INDEXES`
   subgroup.
4. Every categorical column's `CATEGORIES` reference MUST resolve to a
   dataset satisfying {numref}`§%s <hep001-categoricals>`.
5. `column-order`, when present, MUST list every column dataset of the
   table exactly once and MUST NOT list datasets that are not column
   datasets.
6. Every name listed in the table group's `INDEX_COLUMNS` attribute
   (when present) MUST resolve to a column dataset that is a direct
   child of the table group; when `_index` is also present, it MUST
   equal `INDEX_COLUMNS[0]`.

A producer that mutates a table (appends rows, rewrites a column, etc.)
MUST either update the affected search indexes consistently or delete
them before committing the mutation.

(hep001-anndata)=
## Interoperability with Anndata

HEP001 and Anndata's DataFrame layout share the same fundamental idea —
a group with one dataset per column — and HEP001 deliberately uses
Anndata's attribute names (`column-order`, `_index`, `encoding-type`,
`encoding-version`) where the concepts overlap, so that a HEP001 table
group can be made readable by an Anndata consumer with minimal extra
metadata.

A HEP001 producer MAY write a table group that is also a valid Anndata
DataFrame by additionally setting on the table group:

* `encoding-type` = `"dataframe"`,
* `encoding-version` = the Anndata DataFrame version the producer
  targets (for example, `"0.2.0"`).

An Anndata consumer then sees the same group HEP001 sees, down to
categorical columns that share the `"categorical"` encoding. The
attributes specific to HEP001 (`CLASS`, `VERSION`, `INDEX_COLUMNS`,
`SEARCH_INDEX_LIST`, `CATEGORIES`, `VALUES`, `units`,
`units_vocabulary`, `valid_min`, `valid_max`) are ignored by Anndata
and left intact.

`_index` and `INDEX_COLUMNS` carry the same row-label concept in
different shapes: Anndata's `_index` is a scalar string naming one
column; HEP001's `INDEX_COLUMNS` is a 1-D string array supporting
hierarchical row indexes. Producers that target both ecosystems
SHOULD write both, with `_index` set equal to `INDEX_COLUMNS[0]` (the
outermost index level). Anndata consumers see only the primary row
index; HEP001 consumers see the full hierarchy. When a table has no
row labels at all, both attributes are omitted (or `INDEX_COLUMNS` is
an empty array).

Producers targeting both ecosystems SHOULD set the fill value of a
categorical column to `-1` (for signed codes) or the maximum unsigned
value (for unsigned codes), overriding the default sentinel from
{numref}`§%s <fill-table>`, so that Anndata's pandas-derived missingness
convention is honored. The `valid_min` and `valid_max` attributes declare
the actual code range for HEP001 consumers.

Anndata producers commonly use a `NaN` bit pattern as the fill value for
floating-point columns (inherited from pandas' missing-value convention).
HEP001 forbids `NaN` as a fill value (see {numref}`§%s <fill-vals>`)
because IEEE 754 makes `NaN != NaN` and the spec's required equality
test would never detect any missing rows. Producers importing
float columns from Anndata MUST re-fill-value such columns to a
non-`NaN` sentinel (e.g., the recommended `9.9692099683868690e+36`)
before the column is HEP001-conformant.

### Required casing for dual-ecosystem producers

A producer that wants its table groups to be readable by both HEP001 and
Anndata MUST observe the casing rule {ref}`hep001-reserved-names` defines:

* HEP001-owned uppercase reserved names — `CLASS`, `VERSION`, `TITLE`,
  `KIND`, `INDEX_COLUMNS`, `SEARCH_INDEX_LIST`, `CATEGORIES`,
  `SEARCH_INDEXES`, `VALUES`, plus every `KIND` value (`COLUMN_TABLE`,
  `CHUNK_MINMAX`, `SORTED_ROWS`, `BITMAP`, `CHUNK_BLOOM`) — MUST be written
  in fixed-length ASCII, UPPERCASE.
* HEP001-owned lowercase reserved attributes — `valid_min`, `valid_max`
  — MUST be written in lowercase, matching scientific-data convention.
  They are reserved by HEP001 (not by Anndata), and Anndata leaves them
  intact.
* Anndata-shared names — the attribute names `column-order`, `_index`,
  `encoding-type`, `encoding-version`, `ordered`, and the string values
  `"dataframe"` and `"categorical"` — MUST be written in lowercase, exactly
  as Anndata writes them. Uppercasing them would make the group unreadable
  by Anndata.

The two casing regimes coexist on the same group, on the same column, and
on the same categories dataset. Producers MUST NOT case-fold either regime
into the other.

```{note}
The lowercase Anndata names are not "spec-internal" in the same sense as
HEP001's UPPERCASE reserved names — they are part of an external contract
HEP001 chose to honor. A future HEP that updates the Anndata alignment
will, if Anndata ever changes these names, update this list accordingly.
Until then, treat the lowercase form as load-bearing.
```

```{note}
Anndata currently uses a nullable-integer / nullable-boolean encoding
with a sidecar mask. HEP001 {numref}`§%s <fill-vals>` uses fill values instead. Producers
targeting both ecosystems should either avoid nullable columns or write
them in the Anndata form and expose them to HEP001 consumers using a
column that carries a descriptive `description` attribute.
```

(hep001-reserved-names)=
## Reserved names

HEP001 follows the long-standing HDF Group High-Level API practice — established
by the HDF5 Table, Image, and Dimension Scales specifications — of writing
reserved attribute and group names in fixed-length ASCII, UPPERCASE. The intent
is that a reader scanning an HDF5 file can tell at a glance which names belong
to the specification and which were chosen by the producer of the data.

### Naming rules

1. Every attribute or group name introduced as part of the HEP001 specification
   MUST be written in fixed-length uppercase ASCII, with underscores as the only
   word separator (for example, `CLASS`, `INDEX_COLUMNS`, `SEARCH_INDEXES`).

2. Producers MUST NOT use any reserved name listed in
   {ref}`hep001-reserved-names-list` for a column dataset, a search-index
   dataset, a user-supplied attribute, or any other purpose other than the
   one this HEP assigns to it.

3. Names that HEP001 deliberately shares with other ecosystems, currently
   only with Anndata's DataFrame layout (see {ref}`hep001-anndata`), are
   exempt from rule 1 and MUST be written exactly as those ecosystems write
   them. They are listed in {ref}`hep001-shared-names`.

4. Several attributes that align with broader scientific HDF5 community
   practice are exempt from rule 1 and MUST be written in lowercase, so that
   generic metadata harvesters and existing tools can discover them on a
   HEP001 table without case-folding.

   The value-domain attributes `valid_min` and `valid_max`, when present on
   a column dataset, carry contractual meaning: the column's fill value
   MUST lie strictly outside `[valid_min, valid_max]` (see
   {numref}`§%s <fill-vals>`). They appear in the reserved-name catalog
   ({ref}`hep001-reserved-names-list`) despite being lowercase.

5. Per-search-index attributes that are private to a specific search-index
   family — including configuration parameters, declarations of the
   algorithm used, and computed output metrics — are not part of the
   reserved name contract and are written in lowercase `snake_case`. They
   are documented with the index family that defines them
   ({ref}`hep001-search-indexes`).

6. KIND values (the string contents of the `KIND` attribute) are themselves
   reserved tokens and follow the same uppercase rule as reserved names.

(hep001-reserved-names-list)=
### Reserved name catalog

The complete set of HEP001 reserved names is listed below.

#### Group names

`SEARCH_INDEXES`
: The reserved subgroup of a table group that holds every search-index dataset
  for the table. See {ref}`hep001-search-indexes`.

#### Table group attribute names

`CLASS`
: Identifies the group as a HEP001 table group. See {ref}`hep001-class`.

`VERSION`
: HEP001 revision the table conforms to.

`TITLE`
: Human-readable title of the table (optional).

`INDEX_COLUMNS`
: A 1-D array attribute of fixed-length UTF-8 strings, whose elements are names
  of the column datasets that serve as row labels for the table, in hierarchical
  order from outermost to innermost level. See {ref}`hep001-indexes`.

#### Column dataset attribute names

`SEARCH_INDEX_LIST`
: Object references to the search-index datasets that accelerate queries on
  this column.

`CATEGORIES`
: Object reference to the categories dataset that backs a categorical column.

`valid_min`, `valid_max` *(lowercase, by exception)*
: Inclusive lower and upper bounds of the column's logical value range.
  Each is a scalar attribute whose datatype matches the column's element
  datatype. See {numref}`§%s <fill-vals>`.

#### Search-index and categories dataset attribute names

`KIND`
: ASCII enum that identifies the family of a search-index dataset.

`VALUES`
: Object reference, on a `BITMAP` search-index dataset, to its accompanying
  values dataset.

#### `KIND` attribute values

* `CHUNK_MINMAX`
* `SORTED_ROWS`
* `BITMAP`
* `CHUNK_BLOOM`

See {ref}`hep001-search-indexes` for their meaning. Consumers MUST treat unknown
values as "ignore this search index".

(hep001-shared-names)=
### Names shared with Anndata

The following attribute names and string values are written in **lowercase**,
exactly as Anndata writes them, so that a single HEP001 table group can also
be a valid Anndata DataFrame ({ref}`hep001-anndata`). They are reserved for
their Anndata-defined meaning and MUST NOT be repurposed:

* attribute names — `column-order`, `_index`, `encoding-type`,
  `encoding-version`, `ordered`;
* string values — `"dataframe"` and `"categorical"` when written into an
  `encoding-type` attribute.

A producer that does not target Anndata interoperability MAY omit these
names, but if it writes them at all it MUST write them in this exact form
(lowercase, with the hyphens and underscore as shown).

## Worked examples

(min-example-table)=
### A minimal table

A table of four columns — `row_id` (the row index), `ts`, `energy`, `label` —
and no search indexes.

```
/my_table                          (Group)
  CLASS             = "COLUMN_TABLE"   (ASCII, fixed length)
  VERSION           = "1.0"            (ASCII, fixed length)
  TITLE             = "Sample run"     (UTF-8)
  column-order      = ["row_id", "ts", "energy", "label"]  (UTF-8, 1-D)
  INDEX_COLUMNS     = ["row_id"]       (UTF-8, 1-D)
  _index            = "row_id"         (UTF-8; Anndata interop)

/my_table/row_id                   (Dataset, uint64, shape (N,))
  description       = "Globally unique event identifier."

/my_table/ts                       (Dataset, int64, shape (N,))
  units             = "s"
  units_vocabulary  = "UDUNITS-2"
  description       = "Event timestamp."

/my_table/energy                   (Dataset, float32, shape (N,))
  units             = "MeV"

/my_table/label                    (Dataset, int8, shape (N,))
  description       = "Class label."
  CATEGORIES        = ref(label__CATEGORIES)

/my_table/label__CATEGORIES        (Dataset, vlen UTF-8, shape (3,))
  encoding-type     = "categorical"
  ordered           = false
```

### Adding a chunk min/max search index

Extending {numref}`§%s <min-example-table>` with a `CHUNK_MINMAX` index on `ts`:

```
/my_table/ts
  SEARCH_INDEX_LIST = [ref(SEARCH_INDEXES/ts__chunk_minmax)]
  …

/my_table/SEARCH_INDEXES                       (Group)
/my_table/SEARCH_INDEXES/ts__chunk_minmax      (Dataset,
    compound {min: int64, max: int64, nan_count: uint64,
              fill_count: uint64, n: uint64}, shape (n_chunks,))
  KIND          = "CHUNK_MINMAX"
```

### A complete layout

```{mermaid}
graph TD
  TG[["/my_table<br/>CLASS=COLUMN_TABLE<br/>VERSION=1.0<br/>INDEX_COLUMNS=[row_id]"]]
  TG --> ri(["row_id (uint64, row index)"])
  TG --> ts(["ts (int64)"])
  TG --> en(["energy (float32)"])
  TG --> lb(["label (int8, categorical)"])
  TG --> lc[("label__CATEGORIES (vlen UTF-8)")]
  TG --> SI[["SEARCH_INDEXES"]]
  SI --> mm[("ts__chunk_minmax<br/>KIND=CHUNK_MINMAX")]
  SI --> sr[("energy__sorted_rows<br/>KIND=SORTED_ROWS")]
  SI --> bm[("label__bitmap<br/>KIND=BITMAP")]
  SI --> bv[("label__bitmap__VALUES")]
  SI --> bf[("ts__chunk_bloom<br/>KIND=CHUNK_BLOOM")]
  ts -.->|"SEARCH_INDEX_LIST"| mm
  ts -.->|"SEARCH_INDEX_LIST"| bf
  en -.->|"SEARCH_INDEX_LIST"| sr
  lb -.->|"SEARCH_INDEX_LIST"| bm
  bm -.->|"VALUES"| bv

  classDef tableGroup fill:#FFF4D4,stroke:#D4B86A,stroke-width:2px,color:#000
  classDef siGroup    fill:#E5DAF5,stroke:#9D85C5,stroke-width:2px,color:#000
  classDef dataCol    fill:#D4EBF8,stroke:#7FA9D0,stroke-width:1px,color:#000
  classDef indexCol   fill:#FAD4E1,stroke:#D08FB0,stroke-width:1px,color:#000
  classDef catData    fill:#FAE0D4,stroke:#D09F90,stroke-width:1px,color:#000
  classDef siData     fill:#D7F0D8,stroke:#7CB78A,stroke-width:1px,color:#000
  classDef siValues   fill:#E8F3E0,stroke:#A8C088,stroke-width:1px,color:#000

  class TG tableGroup
  class SI siGroup
  class ri indexCol
  class ts,en,lb dataCol
  class lc catData
  class mm,sr,bm,bf siData
  class bv siValues
```

## Security considerations

Search indexes are unsigned, untrusted derivative data. A consumer that
trusts a table's column data MUST NOT, by default, trust the correctness
of a search index found in the same file: a tampered `CHUNK_MINMAX` can
cause the consumer to skip chunks that do in fact satisfy a predicate.
Consumers SHOULD offer a mode that verifies a search index against the
column it covers, or that ignores search indexes entirely. Producers
SHOULD document the provenance of search indexes in the table group's
`description` when that matters to their users.

## Open issues

The preamble of this file lists open design points that will be resolved
during review. They will be moved here as substantive issues arise.

## Future work

The following items are deliberately out of scope for HEP001 v1.0 and
are candidates for future HEPs:

* a sidecar mask mechanism for nullable columns, interoperable with
  Anndata's `nullable-integer` and `nullable-boolean` encodings;
* sparse columns, interoperable with Anndata's CSR / CSC encodings;
* a registry for additional search-index `KIND` values (e.g. zone maps
  beyond min/max, interval trees, kd-trees for multi-column search);
* a standard query-metadata footer that records which indexes a
  consumer used to answer a query (for auditing);
* conventions for append-only tables and for concurrent writers.

## References

* HDF5 Table specification — HDF5 High-Level Library, The HDF Group.
  <https://support.hdfgroup.org/documentation/hdf5/latest/_t_b_l_s_p_e_c.html>
* PyTables File Format — PyTables Users' Guide.
  <https://www.pytables.org/usersguide/file_format.html>
* Anndata on-disk format (DataFrames) — Anndata documentation.
  <https://anndata.readthedocs.io/en/stable/fileformat-prose.html#dataframes>
* Apache Parquet format specification.
  <https://parquet.apache.org/docs/file-format/>
* Apache Arrow columnar format.
  <https://arrow.apache.org/docs/format/Columnar.html>
* RFC 2119 — Key words for use in RFCs to Indicate Requirement Levels.
  <https://datatracker.ietf.org/doc/html/rfc2119>
* RFC 8174 — Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words.
  <https://datatracker.ietf.org/doc/html/rfc8174>
