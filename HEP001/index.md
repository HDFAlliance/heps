---
label: HEP001
description: A column-oriented storage layout for tabular data in HDF5, with first-class support for per-column datatypes, chunking, compression, row indexes, and query-accelerating search indexes.
date: 2026-04-22
tags:
  - draft
keywords:
  - HDF5
  - tabular data
  - columnar storage
  - dataframe
  - Parquet
  - Anndata
  - search index
authors:
  - name: Aleksandar Jelenak
    affiliation: HDF Group
funding:
  statement: This material is based upon work supported by the U.S. Department of Energy, Office of Science, Office of Fusion Energy Sciences, under Award Number​ DE-SC0024442.​
  id: DE-SC0024442
---

% -----------------------------------------------------------------------------
% Open design points still to resolve:
%   * Does `column-order` become mandatory when the table has more than one
%     column, or always optional?
%   * Should `_index` permit a `/`-delimited path for tables that live under
%     deep hierarchies, or always be a local name?
%   * For the chunk min/max search index of floating-point columns, should
%     NaN handling be normative (e.g. NaNs never update min/max; a separate
%     `nan_count` field records them)?
%   * The Bloom filter appendix pins `k` hash functions using a specific
%     seeded scheme; real adopters may need to negotiate interoperable hashes.
% -----------------------------------------------------------------------------

(hep001-title)=
# HEP001: Column-Oriented Tabular Data in HDF5

```{attention}
This HEP is a **work-in-progress draft**. The data model and attribute names below are intended
to be stable, but normative language (MUST / SHOULD / MAY) will be tightened
during review. Comments and pull requests are welcome on the
[HEP repository](https://github.com/HDFAlliance/heps).
```

```{note} Acknowledgments and disclosures
Development of this document was supported by the U.S. Department of Energy, Office of Science, Office of Fusion Energy Sciences, under Award Number​ DE-SC0024442.

No contractual or commercial obligations constrain the content of this document. This work is contributed to the HDF community under the
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

## 1. Introduction

HDF5 has stored tabular data from its earliest days, and several distinct
idioms have grown up around that use case. HEP001 proposes a **column-oriented
storage layout**, in the spirit of Apache Parquet, Apache Arrow, and Feather,
that lives natively as an HDF5 group and combines cleanly with the
multidimensional array datasets that are HDF5's traditional strength.

### 1.1 A short overview of tabular data in HDF5

The first widely adopted idiom is the [HDF5 Table
specification](https://support.hdfgroup.org/documentation/hdf5/latest/_t_b_l_s_p_e_c.html)
that ships with the HDF5 High-Level (`H5TB`) library. A table is a single
one-dimensional dataset whose datatype is an HDF5 compound (record) type,
decorated with attributes such as `CLASS="TABLE"`, `VERSION`, `TITLE`,
`FIELD_0_NAME … FIELD_N_NAME`, `FIELD_0_FILL … FIELD_N_FILL`, and `NROWS`.
Rows of the logical table become elements of the dataset; columns become
fields of the compound datatype. This layout is simple and portable, but it
is fundamentally **row-oriented**: every row occupies contiguous bytes, every
column shares the same chunking, and changing a single column's datatype,
chunk shape, or compression filter requires rewriting the entire dataset.

The second influential idiom is [PyTables](https://www.pytables.org), a Python
package that has layered a rich query engine on top of HDF5. PyTables likewise
stores a table as a single one-dimensional dataset of a compound type —
decorated with its own `CLASS="TABLE"`, `VERSION`, `TITLE`, `FIELD_N_FILL`,
`NROWS`, and `PYTABLES_FORMAT_VERSION` attributes — and adds a rich family of
companion structures for indexing (Completely Sorted Indexes, or CSIs),
in-kernel queries, and compression with Blosc. PyTables popularized the idea
that a useful table format needs more than bytes on disk: it needs indexes,
metadata, and conventions that let tools reason about the data.

The third and most recent influence is [Anndata](https://anndata.readthedocs.io/),
which treats a dataframe as an HDF5 **group** rather than a single
compound dataset. Each column of the dataframe is stored as its own
one-dimensional dataset inside that group. The group carries attributes that
tell Anndata how to reassemble the columns into a dataframe —
`encoding-type="dataframe"`, `encoding-version`, `column-order`
(a UTF-8 string array giving the column order), and `_index` (the name of
the column that supplies row labels). Anndata extends this convention with
dedicated encodings for nullable integers, nullable booleans, categoricals,
sparse matrices, and more. This layout is **column-oriented**, and it is the
closest existing HDF5 practice to what modern analytical engines expect.

### 1.2 Why columnar, and why now

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
data does not have to live alone. An HDF5 file can — in a single self-
describing container — hold

* a HEP001 table group with the experiment's per-event observations,
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
  F --> R["/ (root group)"]
  R --> T["/events (table group, CLASS=COLUMN_TABLE)"]
  R --> I["/images (3-D dataset, detector cube)"]
  R --> C["/calibration (2-D dataset)"]
  T --> c1["ts (int64)"]
  T --> c2["energy (float32)"]
  T --> c3["pixel_ref (object reference)"]
  c3 -. "dereferences to slabs of" .-> I
```

### 1.3 Scope and non-goals

HEP001 specifies:

* the structure of a group that holds a column-oriented table,
* the identifying attributes that let software recognize such a group,
* how individual columns are represented as HDF5 datasets,
* how row-label **indexes** relate columns and the table group,
* how **search indexes** accelerate queries over column values, including a
  normative chunk-min/max index and framework definitions for sorted-row,
  bitmap, and per-chunk Bloom-filter indexes.

HEP001 does *not* specify:

* a query language or execution engine,
* a particular on-the-wire protocol for distributing tables,
* semantic conventions for domain-specific column units (outside the
  `units` / `units_vocabulary` attribute pair, which is descriptive only),
* how tables are committed, versioned, or synchronized — those are the
  province of the storage layer and of other HEPs.

## 2. Conformance

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY**
in this document are to be interpreted as described in [RFC 2119] and
[RFC 8174] when, and only when, they appear in all capitals.

[RFC 2119]: https://datatracker.ietf.org/doc/html/rfc2119
[RFC 8174]: https://datatracker.ietf.org/doc/html/rfc8174

A file, group, or dataset is **HEP001-conformant** when it satisfies every
MUST in the section that applies to it. A producer is HEP001-conformant
when every table group it writes is conformant; a consumer is conformant
when it can read any conformant table group without data loss.

## 3. Terminology

The following terms are used throughout this specification.

Table group
: An HDF5 group that represents one column-oriented table. Identified by the
  `CLASS` attribute (see {ref}`hep001-class`).

Column dataset
: An HDF5 dataset of rank 1 that lives directly under a table group and
  represents one column of the table. The dataset's name is the column name.

Index dataset
: An HDF5 dataset of rank 1 that lives directly under a table group and
  supplies row labels for one or more column datasets. Distinguished by the
  presence of the `_columns_list` attribute.

Search index dataset
: An HDF5 dataset stored under the `_search_indexes` child group of a table
  group that accelerates queries over one or more column datasets. Each kind
  of search index is specified in {ref}`hep001-search-indexes`.

Row
: A position `i` in a column dataset. Every column dataset and every index
  dataset in the same table group MUST have the same length, so the same
  `i` refers to the same logical row everywhere.

## 4. Data model overview

A HEP001 table is an HDF5 **group** whose direct children are the table's
columns (and, optionally, row indexes), plus a reserved `_search_indexes`
subgroup that holds query-acceleration structures.

```{mermaid}
graph TD
  TG["/my_table (table group)<br/>CLASS=COLUMN_TABLE, VERSION=1.0"]
  TG --> c1["ts (1-D column dataset)"]
  TG --> c2["energy (1-D column dataset)"]
  TG --> c3["label (1-D column dataset,<br/>categorical)"]
  TG --> c4["labels_categories (1-D dataset)"]
  TG --> ix["row_id (1-D index dataset,<br/>_columns_list &rarr; ts, energy, label)"]
  TG --> SI["_search_indexes (group)"]
  SI --> mm["ts__chunk_minmax (1-D compound)"]
  SI --> sr["energy__sorted_rows (1-D uint64)"]
  SI --> bm["label__bitmap (1-D compound)"]
  SI --> bf["ts__chunk_bloom (2-D uint8)"]
```

The rest of this document specifies each building block: the table group
({ref}`hep001-table-group`), column datasets ({ref}`hep001-columns`), index
datasets ({ref}`hep001-indexes`), and the four kinds of search index dataset
({ref}`hep001-search-indexes`).

(hep001-table-group)=
## 5. The table group

(hep001-class)=
### 5.1 Identification — the `CLASS` attribute

Every HEP001 table group MUST carry an attribute named `CLASS` with:

* datatype: **fixed-length ASCII string**, 12 bytes (exactly the length of the
  value below, including the trailing byte available for null termination),
  null-terminated or null-padded per the HDF5 string conventions,
* shape: scalar,
* value: `COLUMN_TABLE`.

A consumer MUST identify a group as a HEP001 table group by, and only by,
the presence of a scalar `CLASS` attribute whose string value equals
`COLUMN_TABLE`. A producer MUST NOT write `CLASS="COLUMN_TABLE"` on any
group that does not satisfy the rest of this specification.

```{note}
`CLASS` uses fixed-length ASCII to match long-standing HDF5 High-Level
convention (HDF5 Table, PyTables, Image spec). All other string attributes
in this spec are UTF-8 unless explicitly stated otherwise.
```

### 5.2 The `VERSION` attribute

Every HEP001 table group MUST carry a scalar, fixed-length ASCII attribute
named `VERSION` whose value is the HEP001 revision the table conforms to.
For this revision the value is `1.0`.

Future revisions of HEP001 MUST either preserve backward compatibility
with `1.0` consumers or bump the major version.

### 5.3 Optional table-level attributes

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

`_index`
: Scalar fixed-length UTF-8 string. The name of the column dataset, or
  index dataset, that supplies the canonical row labels for this table
  (for example, `"row_id"`). Matches Anndata's `_index`. Consumers treat
  `_index` as a hint that a particular dataset under the group is "the"
  row index.

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

```{note}
HDF5 fixed-length strings are byte-length-bounded. Producers MUST size
each fixed-length UTF-8 attribute with enough bytes to hold the longest
value they intend to write (a single UTF-8 code point can take up to
four bytes). Values MUST be null-terminated or null-padded per standard
HDF5 string conventions.
```

### 5.4 Placement of the table group

The table group MAY be located anywhere in an HDF5 file's hierarchy. In the
simplest case the root group of the file **is** the table group — the file
holds one table and nothing else — and all columns live at the top level.
Tables MAY also be nested under named groups to organize many tables, or
sited beside multidimensional array datasets that they describe.

```{mermaid}
graph TD
  subgraph Flat
    R1["/ (root = table group)"]
    R1 --> c1a["col_a"]
    R1 --> c1b["col_b"]
    R1 --> SI1["_search_indexes"]
  end
  subgraph Nested
    R2["/ (root)"]
    R2 --> G1["/experiments/"]
    G1 --> T1["run_042 (table group)"]
    G1 --> T2["run_043 (table group)"]
    R2 --> IMG["/images (3-D dataset)"]
  end
```

(hep001-columns)=
## 6. Column datasets

### 6.1 Required properties

Each column of a HEP001 table MUST be stored as an HDF5 dataset that:

* is a direct child of the table group,
* has rank 1,
* has the same length as every other column dataset and index dataset in
  the same table group.

The **name** of the HDF5 dataset is the column name. Any name acceptable as
an HDF5 link name (UTF-8, excluding `/` and NUL) is permitted. Producers
SHOULD avoid names that begin with an underscore, which are reserved for
internal index and metadata datasets defined by this or future HEPs, and
MUST NOT use the reserved name `_search_indexes`.

### 6.2 Datatypes

The datatype of a column dataset MAY be any HDF5 datatype the producer
chooses: fixed- or variable-length integer, floating point, boolean,
fixed- or variable-length string, compound, enum, array, opaque, or
object/region reference. Consumers that encounter a datatype they cannot
map to their in-memory model SHOULD surface the column's raw datatype
rather than dropping it silently.

Two caveats apply:

1. Compound datatypes at the column level are permitted, but a table group
   is not a mechanism for storing a nested row-oriented table. When a
   compound-typed column is used, its fields SHOULD be logically atomic
   (for example, a `(re, im)` complex number).
2. Variable-length datatypes are permitted and often desirable (e.g.
   variable-length UTF-8 strings, ragged numeric arrays). Producers SHOULD
   apply chunking and filters consistent with the datatype's storage
   characteristics.

### 6.3 Chunking and filters

Because each column is its own HDF5 dataset, each column MAY independently
select:

* chunk shape (typically `(N,)` for some chunk length `N` per column),
* dataset creation properties such as fill value, allocation time,
  and track-times,
* the filter pipeline, including shuffle, bit-shuffle, n-bit, scale-offset,
  Deflate, Zstandard, Blosc, Blosc2, SZ, and any other registered HDF5
  filter supported by the producer and consumers.

This per-column flexibility is the core motivation for HEP001 and is
normative: producers MUST NOT require columns to share chunk shape or
filters. Consumers MUST treat each column's storage layout independently.

### 6.4 Missing values (fill values)

HEP001 does not define a sidecar mask dataset for missing values. A column
dataset's HDF5 fill value identifies rows whose value is missing. Producers
that need to represent missing values MUST set the dataset's fill value
explicitly via the dataset creation property list (`H5Pset_fill_value`) and
SHOULD document the chosen fill value in the column's `description`
attribute. Consumers MAY rely on `H5Dget_fill_value` to recover that
sentinel.

```{note}
This is a conscious divergence from Anndata's nullable-integer and
nullable-boolean encodings, which add a companion boolean mask. A future
HEP MAY introduce explicit masks; until then, HEP001 relies on fill
values alone.
```

### 6.5 Column attributes

A column dataset MAY carry any of the following attributes.

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

`description`
: Scalar fixed-length UTF-8 string. Plain-text description of the column's
  contents, provenance, or semantics.

`_indexes`
: One-dimensional array of HDF5 object references. Each reference points
  to an index dataset (see {ref}`hep001-indexes`) that labels the rows of
  this column. The order of references is meaningful — consumers MAY
  interpret the first reference as the primary label source and subsequent
  references as alternative labels.

`_search_indexes`
: One-dimensional array of HDF5 object references. Each reference points
  to a search-index dataset (see {ref}`hep001-search-indexes`) in the
  `_search_indexes` subgroup that accelerates queries on this column.

`_categories`
: Scalar HDF5 object reference. Used only for categorical columns. Points
  to the dataset that holds the categorical values (see
  {ref}`hep001-categoricals`).

(hep001-categoricals)=
### 6.6 Categorical columns

A **categorical** column stores integer codes that index into a separate
**categories** dataset holding the actual values. This is the same pattern
Anndata and Apache Arrow use.

A categorical column MUST:

1. Have an integer datatype (any width, signed or unsigned). The code
   value `-1` (or, for unsigned codes, the dataset's documented fill
   value) denotes a missing category.
2. Carry a scalar `_categories` attribute whose value is an HDF5 object
   reference pointing to the categories dataset.

The **categories dataset** MUST:

1. Live directly under the same table group.
2. Have rank 1, and any datatype appropriate to the category values
   (typically UTF-8 variable-length string).
3. Carry a scalar UTF-8 string attribute `encoding-type` with value
   `"categorical"` (matching Anndata).
4. Carry a scalar boolean attribute `ordered` (matching Anndata's ordered
   categoricals). Producers MUST set `ordered` to true exactly when the
   order of entries in the categories dataset is semantically meaningful.

The categories dataset MAY appear in the table group's `column-order`,
but consumers that present the table as a dataframe SHOULD treat it as
metadata of its linked column rather than as a standalone column.

```{mermaid}
graph LR
  col["label (int8)<br/>_categories &rarr; label_categories"]
  cat["label_categories (vlen UTF-8)<br/>encoding-type=&quot;categorical&quot;<br/>ordered=false"]
  col -- object ref --> cat
```

(hep001-indexes)=
## 7. Index datasets

An **index dataset** supplies row labels for one or more columns. A table
group MAY carry zero or more index datasets as direct children.

### 7.1 Required properties

An index dataset MUST:

* be a direct child of the table group,
* have rank 1,
* have the same length as every column dataset and every other index
  dataset in the same table group,
* carry an attribute named `_columns_list` of rank 1 and datatype HDF5
  object reference, whose elements are references to the column datasets
  it labels.

A dataset MAY simultaneously be a column dataset and an index dataset:
that is, it MAY appear in the `column-order` attribute and also carry
`_columns_list`. This matches Anndata, where the dataset named by
`_index` is typically also a regular column.

### 7.2 Linking columns to their indexes

Each column dataset that wishes to be labeled by an index MUST carry an
`_indexes` attribute — a 1-D array of object references pointing to its
index datasets. The relationship is bidirectional and MUST be consistent:
if index `I` references column `C` via its `_columns_list`, then `C` MUST
reference `I` via its `_indexes`, and vice versa.

```{mermaid}
graph LR
  IX["row_id (index dataset)<br/>_columns_list &rarr; ts, energy"]
  C1["ts (column)<br/>_indexes &rarr; row_id"]
  C2["energy (column)<br/>_indexes &rarr; row_id"]
  IX -->|"_columns_list"| C1
  IX -->|"_columns_list"| C2
  C1 -->|"_indexes"| IX
  C2 -->|"_indexes"| IX
```

### 7.3 Typical uses

* **Row labels.** A string index dataset supplies user-meaningful names
  (e.g. sample IDs) for each row.
* **Ordinal indexes.** An integer index dataset supplies a globally unique
  row number, useful when joining tables.
* **Multi-indexes.** Multiple index datasets may co-exist; their order in
  the column's `_indexes` attribute expresses their precedence.

(hep001-search-indexes)=
## 8. Search indexes

**Search indexes** accelerate queries over column values. They do not
change the logical table; they are derivative, recomputable data. A
conformant consumer MAY ignore any or all search indexes and still
return correct answers, only more slowly.

### 8.1 The `_search_indexes` group

A table group MAY contain a direct child group named `_search_indexes`.
When present, it MUST hold every search-index dataset for the table, and
no other objects. It carries no required attributes of its own.

Search-index datasets in `_search_indexes` MAY have any name. The
convention recommended by this HEP is `<column>__<kind>`, with a double
underscore separating the source column name from the index kind (for
example, `ts__chunk_minmax`, `energy__sorted_rows`,
`label__bitmap`, `ts__chunk_bloom`).

### 8.2 Bidirectional linking

Each search-index dataset MUST carry an attribute `_columns_list` — a 1-D
array of HDF5 object references to every column dataset it accelerates.
Conversely, each column dataset that benefits from a search index MUST
reference it from its own `_search_indexes` attribute. The two sides MUST
stay consistent in the same sense as for index datasets
({ref}`hep001-indexes`).

### 8.3 Common per-index attributes

Every search-index dataset MUST carry a scalar fixed-length ASCII
attribute `KIND` whose value is one of the strings defined below:

| `KIND` value         | Purpose                                     | Defined in    |
|----------------------|---------------------------------------------|---------------|
| `CHUNK_MINMAX`       | Per-chunk min and max of a numeric column   | §8.4          |
| `SORTED_ROWS`        | Row-position permutation of a column        | §8.5          |
| `BITMAP`             | Per-value bitmap of a low-cardinality col.  | §8.6          |
| `CHUNK_BLOOM`        | Per-chunk Bloom filter of a column          | §8.7          |

Future HEPs MAY register additional `KIND` values. Consumers MUST treat
unknown `KIND` values as "ignore this search index".

### 8.4 Chunk min/max search index (`KIND = CHUNK_MINMAX`)

A chunk min/max index accelerates range and equality predicates over a
numeric column by letting the engine skip chunks whose value range does
not overlap the predicate.

**Shape.** The index dataset is 1-D with length equal to the number of
chunks in the source column dataset (`ceil(column_length / chunk_length)`
for a contiguously-chunked column).

**Datatype.** An HDF5 compound datatype with the following fields, in
declaration order:

* `min` — same datatype as the source column's element type.
* `max` — same datatype as the source column's element type.
* `nan_count` — `uint64`. The number of NaN values in the chunk. For
  non-floating-point columns this field MUST be present and set to 0.
* `fill_count` — `uint64`. The number of elements in the chunk that equal
  the column's HDF5 fill value (i.e. count of "missing" under §6.4).
* `n` — `uint64`. The number of logical rows covered by this chunk. The
  last chunk MAY be partially full, in which case `n` is strictly less
  than the column's chunk length.

**Semantics.** For floating-point columns, `min` and `max` MUST ignore
NaN values — if every value in the chunk is NaN, `min` and `max` MUST be
set to the column's fill value and `nan_count` MUST equal `n`. For
integer and other orderable types, `min` and `max` are the ordinary
ordering extrema over the chunk's values, treating fill-value elements
as missing (and excluded from the ordering) when `fill_count > 0`.

**Applicability.** A `CHUNK_MINMAX` index MUST link to exactly one column
dataset via `_columns_list`. A producer MAY build separate
`CHUNK_MINMAX` indexes for several columns, but a single dataset MUST
NOT cover multiple columns because chunks of different columns are
independent.

**Additional attributes on the index dataset.**

* `chunk_shape` — 1-D uint64 array. The source column's chunk shape at
  the time the index was built. (Not derivable from HDF5 metadata for
  contiguous columns; always redundant and checkable for chunked ones.)
* `KIND` — `"CHUNK_MINMAX"` (see §8.3).

**Query pattern.** To evaluate a predicate `col BETWEEN lo AND hi`:

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

### 8.5 Sorted-row permutation index (`KIND = SORTED_ROWS`)

A sorted-row index stores row positions in sorted order of a column's
values, enabling binary search and range scans without reading the full
column.

**Shape.** 1-D, length equal to the column length.

**Datatype.** An unsigned integer wide enough to address every row of the
source column (typically `uint64`).

**Semantics.** Element `i` of the index is the row position `r` such
that the `i`-th smallest value of the column lives at row `r`. Ties MUST
be broken by increasing `r`, so the permutation is total and
deterministic. Handling of NaN and fill-value rows:

* Floating-point columns MUST place NaN-valued rows at the end of the
  permutation, in increasing `r` order, and MUST set the attribute
  `nan_tail_length` to the count of such rows.
* Rows whose value equals the column's fill value (§6.4) MUST appear
  immediately before the NaN tail, in increasing `r` order, and the
  attribute `fill_tail_length` MUST record the count.

**Additional attributes.**

* `KIND` — `"SORTED_ROWS"`.
* `nan_tail_length`, `fill_tail_length` — scalar `uint64`. Both MUST be
  present; either MAY be 0.
* `ordered` — scalar boolean. MUST be true for `SORTED_ROWS`; reserved
  for future indexes that permit partial orderings.

**Applicability.** Exactly one column via `_columns_list`.

### 8.6 Bitmap index (`KIND = BITMAP`)

A bitmap index accelerates equality predicates on low-cardinality
columns.

**Shape.** 2-D of shape `(K, ceil(N / 8))`, where `K` is the number of
distinct values (or categories) indexed and `N` is the column length.

**Datatype.** `uint8`. Bit `r % 8` of byte `r / 8` of row `k` is set iff
the column's value at row `r` equals the `k`-th indexed value.

**Accompanying values dataset.** A sibling 1-D dataset named
`<bitmap_name>__values`, under `_search_indexes`, holds the `K` indexed
values in the same datatype as the source column. Its name is linked
from the bitmap via a scalar `_values` object-reference attribute.

**Additional attributes on the bitmap dataset.**

* `KIND` — `"BITMAP"`.
* `_values` — scalar HDF5 object reference to the values dataset.
* `ordered` — scalar boolean; true iff the rows of the bitmap follow the
  ordering of the values.

**Applicability.** Exactly one column via `_columns_list`.

### 8.7 Per-chunk Bloom filter index (`KIND = CHUNK_BLOOM`)

A per-chunk Bloom filter accelerates equality predicates on
high-cardinality columns by giving a fast negative answer for chunks
that provably do not contain the queried value.

**Shape.** 2-D of shape `(n_chunks, m_bytes)`, where `n_chunks` is the
number of chunks in the source column and `m_bytes` is the Bloom-filter
byte length per chunk (constant across chunks).

**Datatype.** `uint8`. Each row is the packed bit array of one chunk's
Bloom filter.

**Hash scheme.** HEP001 prescribes — for interoperability — the double
hashing `h_i(x) = h_a(x) + i · h_b(x) mod (8 · m_bytes)`, with `h_a` and
`h_b` taken from the 64-bit halves of a 128-bit MurmurHash3 of the
value's canonical byte representation, seeded with `0` and `0x9E3779B9`
respectively. The number of hash functions `k` is stored as an
attribute.

**Additional attributes.**

* `KIND` — `"CHUNK_BLOOM"`.
* `k` — scalar `uint16`. The number of hash functions.
* `m_bits` — scalar `uint64`. Equal to `8 * m_bytes`; stored explicitly
  for clarity.
* `hash_family` — scalar fixed-length ASCII string; for this revision
  MUST be `"murmur3_128_double"`.
* `chunk_shape` — 1-D `uint64`, as in §8.4.

**Applicability.** Exactly one column via `_columns_list`.

```{note}
Bloom filter interoperability requires producers and consumers to agree
on hashing and canonical byte representation. HEP001 freezes both; a
future revision MAY register alternative `hash_family` values.
```

## 9. Consistency requirements

A conformant table group satisfies all of the following at all times:

1. Every column dataset, every index dataset, and every sub-dataset whose
   semantics require row alignment MUST have the same length.
2. For every `_indexes` reference on a column, the target index dataset
   MUST reference that column in `_columns_list`, and vice versa.
3. For every `_search_indexes` reference on a column, the target
   search-index dataset MUST reference that column in `_columns_list`,
   and vice versa.
4. Every dataset in the `_search_indexes` subgroup MUST carry a `KIND`
   attribute defined by this HEP (or a future, registered HEP).
5. Every categorical column's `_categories` reference MUST resolve to a
   dataset satisfying §6.7.
6. `column-order`, when present, MUST list every column dataset of the
   table exactly once and MUST NOT list datasets that are not column
   datasets.

A producer that mutates a table (appends rows, rewrites a column, etc.)
MUST either update the affected search indexes consistently or delete
them before committing the mutation.

(hep001-anndata)=
## 10. Interoperability with Anndata

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
attributes specific to HEP001 (`CLASS`, `VERSION`, `_indexes`,
`_search_indexes`, `_columns_list`, `units`, `units_vocabulary`) are
ignored by Anndata and left intact.

```{note}
Anndata currently uses a nullable-integer / nullable-boolean encoding
with a sidecar mask. HEP001 §6.4 uses fill values instead. Producers
targeting both ecosystems should either avoid nullable columns or write
them in the Anndata form and expose them to HEP001 consumers using a
column that carries a descriptive `description` attribute.
```

## 11. Worked examples

### 11.1 A minimal table

A table of three columns (`ts`, `energy`, `label`), a row index
(`row_id`), and no search indexes.

```
/my_table                       (Group)
  CLASS            = "COLUMN_TABLE"   (ASCII, fixed length)
  VERSION          = "1.0"            (ASCII, fixed length)
  TITLE            = "Sample run"     (UTF-8)
  column-order     = ["ts", "energy", "label"]  (UTF-8, 1-D)
  _index           = "row_id"         (UTF-8)

/my_table/ts                    (Dataset, int64, shape (N,))
  units            = "s"
  units_vocabulary = "UDUNITS-2"
  description      = "Event timestamp."
  _indexes         = [ref(row_id)]

/my_table/energy                (Dataset, float32, shape (N,))
  units            = "MeV"
  _indexes         = [ref(row_id)]

/my_table/label                 (Dataset, int8, shape (N,))
  description      = "Class label."
  _categories      = ref(label_categories)
  _indexes         = [ref(row_id)]

/my_table/label_categories      (Dataset, vlen UTF-8, shape (3,))
  encoding-type    = "categorical"
  ordered          = false

/my_table/row_id                (Dataset, uint64, shape (N,))
  _columns_list    = [ref(ts), ref(energy), ref(label)]
```

### 11.2 Adding a chunk min/max search index

Extending §11.1 with a `CHUNK_MINMAX` index on `ts`:

```
/my_table/ts
  _search_indexes  = [ref(_search_indexes/ts__chunk_minmax)]
  …

/my_table/_search_indexes                       (Group)
/my_table/_search_indexes/ts__chunk_minmax      (Dataset,
    compound {min: int64, max: int64, nan_count: uint64,
              fill_count: uint64, n: uint64}, shape (n_chunks,))
  KIND           = "CHUNK_MINMAX"
  chunk_shape    = [chunk_length]
  _columns_list  = [ref(/my_table/ts)]
```

### 11.3 A complete layout

```{mermaid}
graph TD
  TG["/my_table<br/>CLASS=COLUMN_TABLE<br/>VERSION=1.0"]
  TG --> ts["ts (int64)"]
  TG --> en["energy (float32)"]
  TG --> lb["label (int8, categorical)"]
  TG --> lc["label_categories (vlen UTF-8)"]
  TG --> ri["row_id (uint64, index)"]
  TG --> SI["_search_indexes"]
  SI --> mm["ts__chunk_minmax<br/>KIND=CHUNK_MINMAX"]
  SI --> sr["energy__sorted_rows<br/>KIND=SORTED_ROWS"]
  SI --> bm["label__bitmap<br/>KIND=BITMAP"]
  SI --> bv["label__bitmap__values"]
  SI --> bf["ts__chunk_bloom<br/>KIND=CHUNK_BLOOM"]
  ts -.-> mm
  ts -.-> bf
  en -.-> sr
  lb -.-> bm
  bm -.-> bv
  ri -.-> ts
  ri -.-> en
  ri -.-> lb
```

## 12. Security considerations

Search indexes are unsigned, untrusted derivative data. A consumer that
trusts a table's column data MUST NOT, by default, trust the correctness
of a search index found in the same file: a tampered `CHUNK_MINMAX` can
cause the consumer to skip chunks that do in fact satisfy a predicate.
Consumers SHOULD offer a mode that verifies a search index against the
column it covers, or that ignores search indexes entirely. Producers
SHOULD document the provenance of search indexes in the table group's
`description` when that matters to their users.

## 13. Open issues

The preamble of this file lists open design points that will be resolved
during review. They will be moved here as substantive issues arise.

## 14. Future work

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

## 15. References

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
