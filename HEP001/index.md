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
stores a table as a single one-dimensional dataset of a compound type decorated
with its own `CLASS="TABLE"`, `VERSION`, `TITLE`, `FIELD_N_FILL`, `NROWS`, and
`PYTABLES_FORMAT_VERSION` attributes. This adds a rich family of companion
structures for indexing, in-kernel queries, and compression with Blosc. PyTables
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
matrices, and more. This layout is **column-oriented**, and it is the closest
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
  style R fill:orange
  R --> T["/events (table group, CLASS=COLUMN_TABLE)"]
  style T fill:orange
  R --> I["/images (3-D dataset, detector cube)"]
  R --> C["/calibration (2-D dataset)"]
  T --> c1["ts (int64)"]
  T --> c2["energy (float32)"]
  T --> c3["pixel_ref (object reference)"]
  c3 -. "dereferences to slabs of" .-> I
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
* a particular on-the-wire protocol for distributing tables,
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

(hep001-reserved-names)=
## Reserved names

HEP001 follows the long-standing HDF Group High-Level convention — established
by the HDF5 Table specification, the HDF5 Image specification, and Dimension
Scales — of writing specification-owned attribute and group names in
fixed-length ASCII, UPPERCASE. The intent is that a reader scanning an HDF5
file can tell at a glance which names belong to the specification and which were
chosen by the producer of the data.

### Naming rules

1. Every attribute or group name introduced as part of the HEP001 specification
   MUST be written in fixed-length uppercase ASCII, with underscores as the only
   word separator (for example, `CLASS`, `COLUMN_LIST`, `SEARCH_INDEXES`).
2. Producers MUST NOT use any reserved name listed in
   {ref}`hep001-reserved-names-list` for a column dataset, an index dataset, a
   user-supplied attribute, or any other purpose other than the one this HEP
   assigns to it.
3. Names that HEP001 deliberately shares with other ecosystems, currently
   only with Anndata's DataFrame layout (see {ref}`hep001-anndata`), are
   exempt from rule 1 and MUST be written exactly as those ecosystems write
   them. They are listed in {ref}`hep001-shared-names`.
4. Descriptive annotation attributes `units`, `units_vocabulary`, and
   `description` are also exempt from rule 1 and MUST be written in lowercase.
   They carry no contractual meaning: their presence, absence, or value does not
   change how a HEP001 consumer interprets the table or any of its objects. The
   lowercase form aligns with broader HDF5 community practice (notably netCDF,
   the CF metadata conventions, and Dublin Core descriptive metadata), so that
   generic metadata harvesters and netCDF-aware tools can discover these
   attributes on a HEP001 table without case-folding. Any future descriptive
   annotation attribute introduced by this HEP or a successor MUST follow the
   same lowercase convention. These attributes are defined alongside the objects
   that may carry them ({ref}`hep001-table-group`, {ref}`hep001-columns`); they
   do not appear in the reserved-name catalog because they are not part of the
   HEP001 spec.
5. Per-search-index *tunable* parameters that configure a particular index
   family are not part of the reserved-name contract and are written in
   lowercase `snake_case`. They are documented with the index family that
   defines them ({ref}`hep001-search-indexes`).
6. KIND values (the string contents of the `KIND` attribute) are themselves
   reserved tokens and follow the same uppercase rule as reserved names.

(hep001-reserved-names-list)=
### Reserved name catalog

The complete set of HEP001 reserved names is:

**Group names**

`SEARCH_INDEXES`
: The reserved subgroup of a table group that holds every search-index dataset
  for the table. See {ref}`hep001-search-indexes`.

**Attribute names on the table group**

`CLASS`
: Identifies the group as a HEP001 table group. See {ref}`hep001-class`.

`VERSION`
: HEP001 revision the table conforms to.

`TITLE`
: Human-readable title of the table (optional).

**Attribute names on column datasets**

`INDEX_LIST`
: Object references to the index datasets that label this column.

`SEARCH_INDEX_LIST`
: Object references to the search-index datasets that accelerate queries on
  this column.

`CATEGORIES`
: Object reference to the categories dataset that backs a categorical column.

**Attribute names on index, search-index, and categories datasets**

`COLUMN_LIST`
: Object references to the column datasets that an index dataset or
  search-index dataset applies to.

`KIND`
: ASCII enum that identifies the family of a search-index dataset.

`VALUES`
: Object reference, on a `BITMAP` search-index dataset, to its accompanying
  values dataset.

**KIND values**

`COLUMN_TABLE`, `CHUNK_MINMAX`, `SORTED_ROWS`, `BITMAP`, `CHUNK_BLOOM` — see
{ref}`hep001-class` and {ref}`hep001-search-indexes`. Future HEPs MAY register
additional KIND values; consumers MUST treat unknown values as "ignore this
search index".

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

Index dataset
: An HDF5 dataset of rank 1 that lives directly under a table group and
  supplies row labels for one or more column datasets. Distinguished by the
  presence of the `COLUMN_LIST` attribute.

Search index dataset
: An HDF5 dataset stored under the `SEARCH_INDEXES` child group of a table
  group that accelerates queries over one or more column datasets. Each kind
  of search index is specified in {ref}`hep001-search-indexes`.

Row
: A position `i` in a column dataset. Every column dataset and every index
  dataset in the same table group MUST have the same length, so the same
  `i` refers to the same logical row everywhere.

## Data model overview

A HEP001 table is an HDF5 **group** whose direct children are the table's
columns (and, optionally, row indexes), plus a reserved `SEARCH_INDEXES`
subgroup that holds query-acceleration structures.

```{mermaid}
graph TD
  TG["/my_table (table group)<br/>CLASS=COLUMN_TABLE, VERSION=1.0"]
  TG --> c1["ts (1-D column dataset)"]
  TG --> c2["energy (1-D column dataset)"]
  TG --> c3["label (1-D column dataset,<br/>categorical)"]
  TG --> c4["labelsCATEGORIES (1-D dataset)"]
  TG --> ix["row_id (1-D index dataset,<br/>COLUMN_LIST &rarr; ts, energy, label)"]
  TG --> SI["SEARCH_INDEXES (group)"]
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
## The table group

(hep001-class)=
### Identification — the `CLASS` attribute

Every HEP001 table group MUST carry an attribute named `CLASS` with:

* datatype: fixed-length ASCII string, 12 bytes (exactly the length of the
  value below), null-terminated or null-padded per the HDF5 string conventions,
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

HDF5 fixed-length strings are byte-length-bounded. Producers MUST size
each fixed-length UTF-8 attribute with enough bytes to hold the longest
value they intend to write (a single UTF-8 code point can take up to
four bytes). Values MUST be null-terminated or null-padded per standard
HDF5 string conventions.
```

### The `VERSION` attribute

Every HEP001 table group MUST carry a scalar, fixed-length ASCII attribute
named `VERSION` whose value is the HEP001 revision the table conforms to.
For this revision the value is `1.0`.

Future revisions of HEP001 MUST either preserve backward compatibility
with `1.0` consumers or bump the major version.

### Optional table-level attributes

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
    R1(["/ (root = table group)"])
    R1 --> c1a["col_a"]
    R1 --> c1b["col_b"]
    R1 --> SI1(["SEARCH_INDEXES"])
  end
  subgraph Nested
    R2(["/ (root)"])
    R2 --> G1["/experiments/"]
    G1 --> T1(["run_042 (table group)"])
    G1 --> T2(["run_043 (table group)"])
    R2 --> IMG["/images (3-D dataset)"]
  end
```

(hep001-self-contained)=
### Self-contained contents

A table group is the complete representation of a single column-oriented
table. Every HDF5 object — group, dataset, named datatype, or link —
that is a descendant of the table group MUST exist solely in service of
the table that group defines.

For this revision of HEP001, the only objects permitted anywhere in the
HDF5 hierarchy below a table group are:

* column datasets ({ref}`hep001-columns`);
* index datasets ({ref}`hep001-indexes`);
* categories datasets backing categorical columns
  ({ref}`hep001-categoricals`);
* the reserved `SEARCH_INDEXES` subgroup, the search-index datasets it
  contains, and their accompanying values datasets
  ({ref}`hep001-search-indexes`).

Future revisions of HEP001, and future HEPs that build on it, MAY
introduce additional kinds of group or dataset below a table group.
Every such addition MUST exist exclusively to describe, qualify,
accelerate, or otherwise support the table defined by the enclosing
table group. A descendant whose semantics are independent of that table
is not conformant.

In particular, a producer MUST NOT place under a table group:

* unrelated tables, including other HEP001 table groups;
* multidimensional array datasets that are not derived from, or
  exclusively bound to, the columns of this table;
* user data, application state, or arbitrary metadata that is not
  defined by this HEP or a future, registered HEP.

Producers that wish to colocate such content with a table MUST do so by
placing it as a sibling of the table group, or under an unrelated parent
group elsewhere in the file. Object references, region references, and
external links MAY be used to associate the table's columns with that
external content without violating this rule.

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
* has rank 1,
* has the same length as every other column dataset and index dataset in
  the same table group.

The {ref}` name <hep001-dset-name>` of the HDF5 dataset is the column name. Any
name acceptable as an HDF5 link name (UTF-8, excluding `/` and NUL) is
permitted. Producers MUST NOT use any HEP001 reserved name
({ref}`hep001-reserved-names-list`) — which includes the group name
`SEARCH_INDEXES` — as a column name. Producers SHOULD also avoid names that
begin with an underscore, which are reserved for Anndata-aligned attribute
names (`_index`) and may be claimed by future HEPs.

### Datatypes

The datatype of a column dataset MAY be any HDF5 datatype. Consumers that
encounter a datatype they cannot map to their in-memory model SHOULD surface the
column's raw datatype rather than dropping it silently.

Two caveats apply:

1. Compound datatypes at the column level are permitted, but a table group
   is not a mechanism for storing a nested row-oriented table. When a
   compound-typed column is used, its fields SHOULD be logically atomic.
2. Variable-length datatypes are permitted and often desirable (e.g.
   variable-length UTF-8 strings, ragged numeric arrays). Producers SHOULD
   apply chunking and filters consistent with the datatype's storage
   characteristics.

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

### Missing values (fill values)

HEP001 does not define a sidecar mask dataset for missing values. A column
dataset's HDF5 fill value identifies rows whose value is missing. Producers MUST
set the dataset's fill value explicitly via the dataset creation property list
(`H5Pset_fill_value`) to a value outside of the column values range. Consumers
MUST retrieve this value (`H5Dget_fill_value`) to properly identify column's
missing values.

```{note}
This is a conscious divergence from Anndata's nullable-integer and
nullable-boolean encodings, which add a companion boolean mask. A future
HEP MAY introduce explicit masks; until then, HEP001 relies on fill
values alone.
```

### Column attributes

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

`INDEX_LIST`
: One-dimensional array of HDF5 object references. Each reference points
  to an index dataset (see {ref}`hep001-indexes`) that labels the rows of
  this column. The order of references is meaningful — consumers MAY
  interpret the first reference as the primary label source and subsequent
  references as alternative labels.

`SEARCH_INDEX_LIST`
: One-dimensional array of HDF5 object references. Each reference points
  to a search-index dataset (see {ref}`hep001-search-indexes`) in the
  `SEARCH_INDEXES` subgroup that accelerates queries on this column.

`CATEGORIES`
: Scalar HDF5 object reference. Used only for categorical columns. Points
  to the dataset that holds the categorical values (see
  {ref}`hep001-categoricals`).

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
  col["label (int8)<br/>CATEGORIES &rarr; label__CATEGORIES"]
  cat["label__CATEGORIES (vlen UTF-8)<br/>encoding-type=&quot;categorical&quot;<br/>ordered=false"]
  col -- object ref --> cat
```

(hep001-indexes)=
## Index datasets

An index dataset supplies row labels for one or more columns. A table
group MAY carry zero or more index datasets as direct children.

### Required properties

An index dataset MUST:

* be a direct child of the table group,
* have rank 1,
* have the same length as every column dataset and every other index
  dataset in the same table group,
* carry an attribute named `COLUMN_LIST` of rank 1 and datatype HDF5
  object reference, whose elements are references to the column datasets
  it labels.

A dataset MAY simultaneously be a column dataset and an index dataset:
that is, it MAY appear in the `column-order` attribute and also carry
`COLUMN_LIST`. This matches Anndata, where the dataset named by
`_index` is typically also a regular column.

### Linking columns to their indexes

Each column dataset that wishes to be labeled by an index MUST carry an
`INDEX_LIST` attribute — a 1-D array of object references pointing to its
index datasets. The relationship is bidirectional and MUST be consistent:
if index `I` references column `C` via its `COLUMN_LIST`, then `C` MUST
reference `I` via its `INDEX_LIST`, and vice versa.

```{mermaid}
graph LR
  IX["row_id (index dataset)<br/>COLUMN_LIST &rarr; ts, energy"]
  C1["ts (column)<br/>INDEX_LIST &rarr; row_id"]
  C2["energy (column)<br/>INDEX_LIST &rarr; row_id"]
  IX -->|"COLUMN_LIST"| C1
  IX -->|"COLUMN_LIST"| C2
  C1 -->|"INDEX_LIST"| IX
  C2 -->|"INDEX_LIST"| IX
```

### Typical uses

* **Row labels.** A string index dataset supplies user-meaningful names
  (e.g. sample IDs) for each row.
* **Ordinal indexes.** An integer index dataset supplies a globally unique
  row number, useful when joining tables.
* **Multi-indexes.** Multiple index datasets may co-exist; their order in
  the column's `INDEX_LIST` attribute expresses their precedence.

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

### Bidirectional linking

Each search-index dataset MUST carry an attribute `COLUMN_LIST` — a 1-D
array of HDF5 object references to every column dataset it accelerates.
Conversely, each column dataset that benefits from a search index MUST
reference it from its own `SEARCH_INDEX_LIST` attribute. The two sides MUST
stay consistent in the same sense as for index datasets
({ref}`hep001-indexes`).

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

**Shape:** The index dataset is 1-D with length equal to the number of chunks in
the source column dataset (`ceil(column_length / chunk_length)` for a chunked
column). A contiguous column (dataset) is treated as having one chunk.

**Datatype:** An HDF5 compound datatype with the following fields, in
declaration order:

* `min` — same datatype as the source column's element type.
* `max` — same datatype as the source column's element type.
* `nan_count` — `uint64`. The number of NaN values in the chunk. For
  non-floating-point columns this field MUST be present and set to 0.
* `fill_count` — `uint64`. The number of elements in the chunk that equal
  the column's HDF5 fill value (i.e. count of "missing" under §7.4).
* `n` — `uint64`. The number of logical rows covered by this chunk. The
  last chunk MAY be partially full, in which case `n` is strictly less
  than the column's chunk length.

**Semantics:** For floating-point columns, `min` and `max` MUST ignore
NaN values — if every value in the chunk is NaN, `min` and `max` MUST be
set to the column's fill value and `nan_count` MUST equal `n`. For
integer and other orderable types, `min` and `max` are the ordinary
ordering extrema over the chunk's values, treating fill-value elements
as missing (and excluded from the ordering) when `fill_count > 0`.

**Applicability:** A `CHUNK_MINMAX` index MUST link to exactly one column
dataset via `COLUMN_LIST`. A producer MAY build separate
`CHUNK_MINMAX` indexes for several columns, but a single dataset MUST
NOT cover multiple columns because chunks of different columns are
independent.

**Additional attributes on the index dataset:**

* `chunk_shape` — 1-D uint64 array. The source column's chunk shape at
  the time the index was built. (Not derivable from HDF5 metadata for
  contiguous columns; always redundant and checkable for chunked ones.)
* `KIND` — `"CHUNK_MINMAX"` (see §9.3).

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
that the `i`-th smallest value of the column lives at row `r`. Ties MUST
be broken by increasing `r`, so the permutation is total and
deterministic. Handling of NaN and fill-value rows:

* Floating-point columns MUST place NaN-valued rows at the end of the
  permutation, in increasing `r` order, and MUST set the attribute
  `nan_tail_length` to the count of such rows.
* Rows whose value equals the column's fill value (§7.4) MUST appear
  immediately before the NaN tail, in increasing `r` order, and the
  attribute `fill_tail_length` MUST record the count.

**Additional attributes:**

* `KIND` — `"SORTED_ROWS"`.
* `nan_tail_length`, `fill_tail_length` — scalar `uint64`. Both MUST be
  present; either MAY be 0.
* `ordered` — scalar boolean. MUST be true for `SORTED_ROWS`; reserved
  for future indexes that permit partial orderings.

**Applicability:** Exactly one column via `COLUMN_LIST`.


(bitmap)=
### Bitmap index (`KIND = BITMAP`)

A bitmap index accelerates equality predicates on low-cardinality
columns.

**Shape:** 2-D of shape `(K, ceil(N / 8))`, where `K` is the number of
distinct values (or categories) indexed and `N` is the column length.

**Datatype:** `uint8`. Bit `r % 8` of byte `r / 8` of row `k` is set iff
the column's value at row `r` equals the `k`-th indexed value.

**Accompanying values dataset:** A sibling 1-D dataset named
`<bitmap_name>__VALUES`, under `SEARCH_INDEXES`, holds the `K` indexed
values in the same datatype as the source column. Its name is linked
from the bitmap via a scalar `VALUES` object-reference attribute.

**Additional attributes on the bitmap dataset:**

* `KIND` — `"BITMAP"`.
* `VALUES` — scalar HDF5 object reference to the values dataset.
* `ordered` — scalar boolean; true iff the rows of the bitmap follow the
  ordering of the values.

**Applicability:** Exactly one column via `COLUMN_LIST`.


(bloom)=
### Per-chunk Bloom filter index (`KIND = CHUNK_BLOOM`)

A per-chunk Bloom filter accelerates equality predicates on
high-cardinality columns by giving a fast negative answer for chunks
that provably do not contain the queried value.

**Shape:** 2-D of shape `(n_chunks, m_bytes)`, where `n_chunks` is the
number of chunks in the source column and `m_bytes` is the Bloom-filter
byte length per chunk (constant across chunks).

**Datatype:** `uint8`. Each row is the packed bit array of one chunk's
Bloom filter.

**Hash scheme:** HEP001 prescribes — for interoperability — the double
hashing `h_i(x) = h_a(x) + i · h_b(x) mod (8 · m_bytes)`, with `h_a` and
`h_b` taken from the 64-bit halves of a 128-bit MurmurHash3 of the
value's canonical byte representation, seeded with `0` and `0x9E3779B9`
respectively. The number of hash functions `k` is stored as an
attribute.

**Additional attributes:**

* `KIND` — `"CHUNK_BLOOM"`.
* `k` — scalar `uint16`. The number of hash functions.
* `m_bits` — scalar `uint64`. Equal to `8 * m_bytes`; stored explicitly
  for clarity.
* `hash_family` — scalar fixed-length ASCII string; for this revision
  MUST be `"murmur3_128_double"`.
* `chunk_shape` — 1-D `uint64`, as in §9.4.

**Applicability:** Exactly one column via `COLUMN_LIST`.

```{note}
Bloom filter interoperability requires producers and consumers to agree
on hashing and canonical byte representation. HEP001 freezes both; a
future revision MAY register alternative `hash_family` values.
```

## Consistency requirements

A conformant table group satisfies all of the following at all times:

1. Every column dataset, every index dataset, and every sub-dataset whose
   semantics require row alignment MUST have the same length.
2. For every `INDEX_LIST` reference on a column, the target index dataset
   MUST reference that column in `COLUMN_LIST`, and vice versa.
3. For every `SEARCH_INDEX_LIST` reference on a column, the target
   search-index dataset MUST reference that column in `COLUMN_LIST`,
   and vice versa.
4. Every dataset in the `SEARCH_INDEXES` subgroup MUST carry a `KIND`
   attribute defined by this HEP (or a future, registered HEP).
5. Every categorical column's `CATEGORIES` reference MUST resolve to a
   dataset satisfying §7.7.
6. `column-order`, when present, MUST list every column dataset of the
   table exactly once and MUST NOT list datasets that are not column
   datasets.

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
attributes specific to HEP001 (`CLASS`, `VERSION`, `INDEX_LIST`,
`SEARCH_INDEX_LIST`, `COLUMN_LIST`, `CATEGORIES`, `VALUES`, `units`,
`units_vocabulary`) are ignored by Anndata and left intact.

### Required casing for dual-ecosystem producers

A producer that wants its table groups to be readable by both HEP001 and
Anndata MUST observe the casing rule {ref}`hep001-reserved-names` defines:

* HEP001-owned reserved names — `CLASS`, `VERSION`, `TITLE`, `KIND`,
  `INDEX_LIST`, `SEARCH_INDEX_LIST`, `COLUMN_LIST`, `CATEGORIES`,
  `SEARCH_INDEXES`, `VALUES`, plus every `KIND` value (`COLUMN_TABLE`,
  `CHUNK_MINMAX`, `SORTED_ROWS`, `BITMAP`, `CHUNK_BLOOM`) — MUST be written
  in fixed-length ASCII, UPPERCASE.
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
with a sidecar mask. HEP001 §7.4 uses fill values instead. Producers
targeting both ecosystems should either avoid nullable columns or write
them in the Anndata form and expose them to HEP001 consumers using a
column that carries a descriptive `description` attribute.
```

## Worked examples

### A minimal table

A table of three columns (`ts`, `energy`, `label`), a row index
(`row_id`), and no search indexes.

```
/my_table                          (Group)
  CLASS             = "COLUMN_TABLE"   (ASCII, fixed length)
  VERSION           = "1.0"            (ASCII, fixed length)
  TITLE             = "Sample run"     (UTF-8)
  column-order      = ["ts", "energy", "label"]  (UTF-8, 1-D)
  _index            = "row_id"         (UTF-8)

/my_table/ts                       (Dataset, int64, shape (N,))
  units             = "s"
  units_vocabulary  = "UDUNITS-2"
  description       = "Event timestamp."
  INDEX_LIST        = [ref(row_id)]

/my_table/energy                   (Dataset, float32, shape (N,))
  units             = "MeV"
  INDEX_LIST        = [ref(row_id)]

/my_table/label                    (Dataset, int8, shape (N,))
  description       = "Class label."
  CATEGORIES        = ref(label__CATEGORIES)
  INDEX_LIST        = [ref(row_id)]

/my_table/label__CATEGORIES        (Dataset, vlen UTF-8, shape (3,))
  encoding-type     = "categorical"
  ordered           = false

/my_table/row_id                   (Dataset, uint64, shape (N,))
  COLUMN_LIST       = [ref(ts), ref(energy), ref(label)]
```

### Adding a chunk min/max search index

Extending §12.1 with a `CHUNK_MINMAX` index on `ts`:

```
/my_table/ts
  SEARCH_INDEX_LIST = [ref(SEARCH_INDEXES/ts__chunk_minmax)]
  …

/my_table/SEARCH_INDEXES                       (Group)
/my_table/SEARCH_INDEXES/ts__chunk_minmax      (Dataset,
    compound {min: int64, max: int64, nan_count: uint64,
              fill_count: uint64, n: uint64}, shape (n_chunks,))
  KIND          = "CHUNK_MINMAX"
  chunk_shape   = [chunk_length]
  COLUMN_LIST   = [ref(/my_table/ts)]
```

### A complete layout

```{mermaid}
graph TD
  TG["/my_table<br/>CLASS=COLUMN_TABLE<br/>VERSION=1.0"]
  TG --> ts["ts (int64)"]
  TG --> en["energy (float32)"]
  TG --> lb["label (int8, categorical)"]
  TG --> lc["label__CATEGORIES (vlen UTF-8)"]
  TG --> ri["row_id (uint64, index)"]
  TG --> SI["SEARCH_INDEXES"]
  SI --> mm["ts__chunk_minmax<br/>KIND=CHUNK_MINMAX"]
  SI --> sr["energy__sorted_rows<br/>KIND=SORTED_ROWS"]
  SI --> bm["label__bitmap<br/>KIND=BITMAP"]
  SI --> bv["label__bitmap__VALUES"]
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
