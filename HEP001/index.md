---
label: HEP001
description: A column-oriented storage layout for tabular data in HDF5, with first-class support for per-column datatypes, chunking, compression, row indexes, and query-accelerating search indexes.
date: 2026-07-06
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
  statement: This material is based upon work supported by the U.S. Department of Energy, Office of Science, Office of Fusion Energy Sciences, under Award Number DE-SC0024442.
  id: DE-SC0024442
numbering:
  heading_2: true
  heading_3: true
---

(hep001-title)=
# H5Col: Column-Oriented Tabular Data in HDF5

```{attention}
This proposal is a **draft**. Comments and pull requests are welcome on the
[HEP repository](https://github.com/HDFAlliance/heps).
```

```{note} Acknowledgments and disclosures
Development of this document was supported by the U.S. Department of Energy,
Office of Science, Office of Fusion Energy Sciences, under Award Number DE-SC0024442.

No contractual or commercial obligations constrain the content of this document.
This work is contributed to the HDF community under the
[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) license.
```

## Introduction

HDF5 has stored tabular data from its earliest days, and several distinct
idioms have grown up around that use case. HEP001 proposes a **column-oriented
storage layout**, in the spirit of Apache Parquet, Apache Arrow, and Feather,
that lives natively as an HDF5 group and combines cleanly with the
multidimensional array datasets that are HDF5's traditional strength.

### A short overview of tabular data in HDF5

The first adopted idiom is the [HDF5 Table specification](https://support.hdfgroup.org/documentation/hdf5/latest/_t_b_l_s_p_e_c.html),
part of the HDF5 High-Level Library and implemented through its `H5TB`
API. A table is a single one-dimensional dataset whose datatype is an
HDF5 compound (record) type,
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
  R --> T[["/events (table group,<br/>CLASS=COLUMN_TABLE)"]]
  R --> I>"/images (3-D dataset,<br/>detector cube)"]
  R --> C>"/calibration (2-D dataset)"]
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
* how columns whose row values are variable-length lists are represented
  as HDF5 groups, to arbitrary nesting depth,
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

### Datatype byte order

Except where this specification pins an exact datatype — the boolean datatype's
`H5T_STD_I8LE` base ({numref}`§%s <hep001-boolean-attributes>`) — integer and
floating-point attributes and datasets MAY be stored in any byte order. HDF5
datatypes are self-describing, and the library converts on read. Datatype names
such as `uint64` in this document therefore constrain width and signedness, not
byte order. The canonical byte representations used for Bloom-filter hashing
({numref}`§%s <bloom>`) are defined independently of storage order.


## Terminology

The following terms are used throughout this specification.

Table group
: An HDF5 group that represents one column-oriented table. Identified by the
`CLASS` attribute (see {numref}`§%s <hep001-class>`).

(hep001-dset-name)=
Dataset name
: The name of the HDF5 hard link that connects an HDF5 dataset with its parent
HDF5 group.

Column dataset
: An HDF5 dataset of rank 1 that is a direct child of a table group and
  represents one column of the table. Datasets inside the reserved
  `CATEGORIES` and `SEARCH_INDEXES` subgroups, and datasets inside list
  column groups, are not column datasets. The dataset's name is the column
  name.

List column
: An HDF5 group that is a direct child of a table group, carries
  `CLASS="LIST_COLUMN"`, and represents one column whose row values are
  variable-length lists. The group's name is the column name. See
  {ref}`hep001-list-columns`.

Column
: A column dataset or a list column group. Where this specification says
  "column" without qualification, the statement applies to both.

Row index column
: A column dataset referenced by the table group's `INDEX_COLUMNS`
  attribute and which therefore supplies row labels for the table. Row index
  columns are otherwise indistinguishable from any other column dataset; the
  designation is made at the table-group level, not on the column itself.

Row
: A position `i` in the half-open range `[0, NROWS)`
  (see {numref}`§%s <hep001-nrows>`) within every column dataset of the
  table group. Every column dataset MUST have the same first-dimension
  extent and that extent MUST be `≥ NROWS`, so the same `i` refers to the
  same logical row everywhere.

`NROWS`
: The number of logical rows currently in the table. A scalar `uint64`
  attribute on the table group, defined in {numref}`§%s <hep001-nrows>`.

Categories dataset
: An HDF5 dataset of rank 1 stored under the `CATEGORIES` child group of a
  table group that holds the label values backing one or more categorical
  columns. See {ref}`hep001-categoricals`.

Search index dataset
: An HDF5 dataset stored under the `SEARCH_INDEXES` child group of a table
  group that accelerates queries over one or more column datasets. Each kind
  of search index is specified in {ref}`hep001-search-indexes`.


## Data model overview

A HEP001 table is an HDF5 group whose direct children are the table's
columns — rank-1 column datasets (one or more of which MAY be designated
as row index columns via the table group's `INDEX_COLUMNS` attribute) and
optionally:

* list column groups holding variable-length list values
({ref}`hep001-list-columns`),
* two reserved subgroups: `CATEGORIES`, with the label datasets that back
categorical columns, and `SEARCH_INDEXES`, with query-acceleration structures.

The table's authoritative row count lives in the table group's `NROWS` attribute
(see {numref}`§%s <hep001-nrows>`); every column dataset has the same
first-dimension extent, and rows `[0, NROWS)` are the table's data.

```{mermaid}
flowchart TD
  subgraph Table_Group [Table Group]
    direction LR
    TG[["/my_table (table group)<br/>CLASS='COLUMN_TABLE',<br/>VERSION='1.0'<br/>NROWS=N,<br/>INDEX_COLUMNS=[ref(row_id)]   "]]

    %% By connecting TG to these in order, and them to each other invisibly,
    %% we force a more structured layout.
    c0(["row_id<br/>(1-D column dataset)"])
    c1(["ts<br>(1-D column dataset)"])
    c2(["energy<br>(1-D column dataset)"])
    c3(["label<br>(1-D column dataset,<br/>categorical)"])

    TG -.->|"INDEX_COLUMNS"| c0

    c0 ~~~ c1 ~~~ c2 ~~~ c3
  end

  subgraph Categories [Categories]
    direction TB
    CG[["/my_table/CATEGORIES (group)"]]
    cc>"label__CATEGORIES<br>(1-D categories dataset)"]
  end

  subgraph Search_Indexes [Search Indexes]
    direction TB
    SI[["/my_table/SEARCH_INDEXES (group)"]]
    mm>"ts__chunk_minmax<br>(1-D compound)"]
    bf>"ts__chunk_bloom<br>(2-D uint8)"]
    sr>"energy__sorted_rows<br>(1-D uint64)"]
    bm>"label__bitmap<br>(2-D uint8)"]
    bv>"label__bitmap__VALUES<br>(1-D)"]

    %% Forcing mm and bf to sit side-by-side under SI
    mm ~~~ bf
  end

  %% Solid Line Connections
  TG --> c0
  TG --> c1
  TG --> c2
  TG --> c3
  TG -->|"subgroup"| CG
  TG -->|"subgroup"| SI

  CG --> cc

  SI --> mm
  SI --> bf
  SI --> sr
  SI --> bm
  SI --> bv

  %% Dotted Line Connections
  c3 -.->|"CATEGORIES"| cc
  bm -.->|"VALUES"| bv
  c1 -.->|"SEARCH_INDEX_LIST"| mm
  c1 -.->|"SEARCH_INDEX_LIST"| bf
  c2 -.->|"SEARCH_INDEX_LIST"| sr
  c3 -.->|"SEARCH_INDEX_LIST"| bm

  %% Styling
  classDef tableGroup fill:#FFF4D4,stroke:#D4B86A,stroke-width:2px,color:#000
  classDef siGroup    fill:#E5DAF5,stroke:#9D85C5,stroke-width:2px,color:#000
  classDef catGroup   fill:#F5E2D0,stroke:#C59D85,stroke-width:2px,color:#000
  classDef dataCol    fill:#D4EBF8,stroke:#7FA9D0,stroke-width:1px,color:#000
  classDef indexCol   fill:#FAD4E1,stroke:#D08FB0,stroke-width:1px,color:#000
  classDef catData    fill:#FAE0D4,stroke:#D09F90,stroke-width:1px,color:#000
  classDef siData     fill:#D7F0D8,stroke:#7CB78A,stroke-width:1px,color:#000
  classDef siValues   fill:#E8F3E0,stroke:#A8C088,stroke-width:1px,color:#000
  style Table_Group fill:transparent,stroke:#999,stroke-width:2px,stroke-dasharray: 5 5
  style Categories fill:transparent,stroke:#999,stroke-width:2px,stroke-dasharray: 5 5
  style Search_Indexes fill:transparent,stroke:#999,stroke-width:2px,stroke-dasharray: 5 5

  class TG tableGroup
  class SI siGroup
  class CG catGroup
  class c0 indexCol
  class c1,c2,c3 dataCol
  class cc catData
  class mm,sr,bm,bf siData
  class bv siValues
```

The rest of this document specifies each building block: the table group
({ref}`hep001-table-group`), column datasets ({ref}`hep001-columns`), list
columns ({ref}`hep001-list-columns`), row index columns
({ref}`hep001-indexes`), and the four kinds of search-index
dataset ({ref}`hep001-search-indexes`).

(hep001-object-references)=
## Object references

Several HEP001 attributes link objects within a table group by HDF5 object
reference: `INDEX_COLUMNS` ({numref}`§%s <hep001-table-group>`), `CATEGORIES`
({numref}`§%s <hep001-categoricals>`), `SEARCH_INDEX_LIST` ({numref}`§%s <hep001-search-indexes>`),
and `VALUES` ({numref}`§%s <bitmap>`). Every such reference MUST use the HDF5
standard reference datatype `H5T_STD_REF` (introduced in HDF5 1.12).
`H5T_STD_REF` is a unified datatype that can carry object, dataset-region, and
attribute references (so future HEP revisions can introduce finer-grained
linkages without a new on-disk reference format).

Producers MUST NOT write the deprecated object-reference datatype
`H5T_STD_REF_OBJ`. A consumer MAY reject, as non-conformant, any HEP001
reference attribute whose datatype is not `H5T_STD_REF`.

(hep001-boolean-attributes)=
## The boolean datatype

HDF5 has no native boolean datatype, and the wider HDF5 ecosystem has not
converged on one encoding. HEP001 fixes a single, self-describing form so
that boolean values are unambiguous on disk wherever they occur: in the
attributes this specification calls *boolean*, in the `MASK` member
datasets of list columns ({numref}`§%s <list-column-members>`), and in
boolean columns ({numref}`§%s <hep001-booleans>`).

The HEP001 boolean datatype is an HDF5 enumerated datatype with:

* base type `H5T_STD_I8LE` (signed 8-bit integer, little-endian), and
* exactly two members, `FALSE` mapped to `0` and `TRUE` mapped to `1`.
  Member names are case-sensitive.

In HDF5 DDL (as emitted by `h5dump`), the datatype is described as:

```
H5T_ENUM {
   H5T_STD_I8LE;
   "FALSE"    0;
   "TRUE"     1;
}
```

Producers MUST write every HEP001 boolean attribute and boolean-valued
dataset with exactly this datatype, and MUST NOT store a HEP001 boolean as
a plain integer, an `H5T_BITFIELD`, or a string. A consumer determines
truth from the enumerated integer value (`0` = false, `1` = true).

Consumers MUST additionally accept, as boolean, any enumeration datatype
whose base type is a one-byte integer of either signedness and whose
members are exactly `FALSE` = 0 and `TRUE` = 1. This tolerance
accommodates writers that use native rather than standard base types
(e.g., h5py); byte order is immaterial for a one-byte type. An
enumeration datatype with any other member set is not a boolean.

(hep001-table-group)=
## The table group

(hep001-class)=
### Identification — the `CLASS` attribute

Every HEP001 table group MUST carry an attribute named `CLASS` with:

* datatype: null-terminated 13-byte fixed-length ASCII string
  (exactly the length of the value below plus a NUL byte),
* shape: scalar,
* value: `COLUMN_TABLE`.

A consumer MUST identify a group as a HEP001 table group by, and only by, the
presence of a `CLASS` attribute whose string value equals `COLUMN_TABLE`. A
producer MUST NOT write `CLASS="COLUMN_TABLE"` attribute on any group that does
not satisfy the rest of this specification.

The identification is strict-write, lenient-read. Producers MUST write
the exact datatype above, but consumers SHOULD accept a `CLASS`
attribute whose *logical string value* equals `COLUMN_TABLE` regardless
of storage details: fixed-length strings of a larger size
(null-terminated or null-padded per standard HDF5 conventions),
variable-length strings, and either declared character set. Experience
with earlier HDF5 conventions shows that third-party writers routinely
vary these details. The same lenient-read rule applies to every HEP001
attribute whose value is a reserved ASCII token — `CLASS` on any group,
`KIND`, and `VERSION`.

```{note}
HDF5 fixed-length strings are byte-length-bounded. Producers MUST size
each fixed-length UTF-8 attribute with enough bytes to hold the longest
value they intend to write (a single UTF-8 code point can take up to
four bytes). Values MUST be null-terminated or null-padded per standard
HDF5 string conventions.
```

### The `VERSION` attribute

Every HEP001 table group MUST carry a scalar, fixed-length ASCII attribute named
`VERSION` whose value is the HEP001 revision the table conforms to. Producers
MUST size the attribute to hold the value being written. For this revision the
value is `"1.0"`.

HEP001 uses a two-component `MAJOR.MINOR` version with the major/minor semantics
of [Semantic Versioning][SemVer]: `MAJOR` increments on a backward-incompatible
change to the data model (one an existing conformant consumer can no longer
read), and `MINOR` on a backward-compatible addition. HEP001 omits SemVer's
third (`PATCH`) component deliberately since it would carry no actionable
meaning. A consumer that prefers a SemVer-style triple MAY read an absent
`PATCH` component as `0` — so `"1.0"` is equivalent to `"1.0.0"`. Consumers MUST
compare `VERSION` values numerically and MUST refuse to process a table whose `MAJOR`
exceeds the highest `MAJOR` they implement.

[SemVer]: https://semver.org/

(hep001-nrows)=
### The `NROWS` attribute

Every HEP001 table group MUST carry a scalar `NROWS` attribute of datatype
`uint64` whose value is the number of logical rows currently in the table.
`NROWS` is the table's authoritative row count: rows `[0, NROWS)` of every
column dataset are part of the table, and rows `[NROWS, extent)`, when
present, are reserved storage.

`NROWS` and a column dataset's first-dimension extent are related but not
equal in general. Every column's extent MUST be `≥ NROWS` (see
{numref}`§%s <hep001-consistency>`), but it MAY exceed `NROWS` when a producer
has preallocated space for future appends. A consumer MUST determine the
table's row count from `NROWS` and MUST NOT infer it from extent — doing so
would silently include preallocated slots or post-crash residue as if they
were table rows.

The attribute is borrowed from the long-established HDF5 Table and PyTables
conventions, where it plays the same role. Centralizing the row count in
one place — independent of any column's storage state — gives producers a
single-attribute commit point for atomic appends (see
{numref}`§%s <write-workflow>`) and gives consumers a single place to look
for the table's size.

For a freshly created, empty table, `NROWS = 0`. Every column dataset's
first-dimension extent MUST be `≥ 0` (a zero-length column is permitted
and is the natural form of a brand-new table).

### Optional table group attributes

The table group MAY carry the following attributes.

`GENERATION`
: Scalar `uint64`. Identifies the current state of the table's column
  data for the index validity check. REQUIRED whenever the table group
  contains any search-index dataset; MAY be omitted otherwise. See
  {numref}`§%s <validity-tokens>`.

`TITLE`
: Scalar fixed-length UTF-8 string. Human-readable title of the table,
  mirroring HDF5 Table and PyTables. Purely descriptive.

`INDEX_COLUMNS`
: One-dimensional attribute of HDF5 object references whose elements point
  to the column datasets that serve as row labels for this table, in
  hierarchical order from outermost to innermost level. For example, a table
  indexed by `(donor_id, sample_id, cell_id)` writes `INDEX_COLUMNS =
  [ref(donor_id), ref(sample_id), ref(cell_id)]`. An empty array or absent
  attribute means the table has no row labels and rows are positional only.
  Every reference MUST resolve to a column dataset that is a direct child of
  the table group and MUST NOT be a null reference; a list column
  ({ref}`hep001-list-columns`) MUST NOT be referenced, because lists have
  no HEP001-defined order. Row index columns apply
  to the table as a whole — every column in the table is labeled by them.
  See {ref}`hep001-indexes`.

`column-order`
: One-dimensional fixed-length UTF-8 string attribute whose elements are
  the names of the table's columns — column datasets and list columns
  alike — in their logical order. When present,
  it fully determines the column order presented to users; when absent,
  the logical column order is implementation-defined. Producers SHOULD
  write `column-order` whenever a table has more than one column. The
  attribute name uses a hyphen (not an underscore) to match Anndata.

`_index`
: Scalar fixed-length UTF-8 string. The dataset name of the column that
  supplies the primary row labels for this table (for example, `"row_id"`).
  The name is borrowed from Anndata's `_index`. When both `INDEX_COLUMNS` and
  `_index` are present, `_index` MUST equal the dataset name of the
  column referenced by `INDEX_COLUMNS[0]`. `INDEX_COLUMNS` is the
  authoritative row-label declaration; `_index` is a derived convenience
  written for Anndata affinity, and consumers MUST NOT prefer it over
  `INDEX_COLUMNS` when both are present.

`encoding-type` / `encoding-version`
: Scalar fixed-length UTF-8 strings, both optional. These names are borrowed
  from Anndata's DataFrame convention ({numref}`§%s <hep001-shared-names>`). A
  producer MAY set `encoding-type="dataframe"` and `encoding-version="0.2.0"`
  if it separately wants Anndata software to attempt reading the group, but
  HEP001 neither requires nor interprets them and makes no guarantee that the
  result is a usable Anndata DataFrame
  ({numref}`§%s <hep001-anndata-relationship>`).

`description`
: Scalar fixed-length UTF-8 string. Free-text description of the table's
  contents. Longer and richer than `TITLE`; intended for documentation
  viewers.

`units_vocabulary`
: Scalar fixed-length UTF-8 string. Names the vocabulary or authority that
  interprets `units` strings on columns of this table — for example
  `"UDUNITS-2"`, `"UCUM"`, `"QUDT"`, or a URL. When present on the table
  group, it applies as a default to every column whose own `units_vocabulary`
  is absent.

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
    R2 --> IMG>"/images (3-D dataset)"]
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
the data model defined here.

The only objects permitted anywhere in the HDF5 hierarchy below a table group
are:

* the column datasets of the table ({ref}`hep001-columns`), which MAY include
  row index columns designated by the table group's `INDEX_COLUMNS` attribute
  ({ref}`hep001-indexes`);
* list column groups and everything they contain
  ({ref}`hep001-list-columns`);
* the reserved `CATEGORIES` subgroup and everything it contains
  ({ref}`hep001-categoricals`);
* the reserved `SEARCH_INDEXES` subgroup and everything it contains
  ({ref}`hep001-search-indexes`).

Any descendant of a table group that does not match one of the categories above
is non-conformant. A producer MUST NOT place under a table group:

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

Each column of a HEP001 table — except list columns, which are stored as
groups per {ref}`hep001-list-columns` — MUST be stored as an HDF5 dataset
that:

* is a direct child of the table group,
* has rank 1 shape,
* has the same first-dimension extent as every other column dataset
  in the same table group, and that extent is `≥ NROWS`
  (see {numref}`§%s <hep001-nrows>` and {numref}`§%s <hep001-consistency>`).

The {ref}`name <hep001-dset-name>` of the HDF5 dataset is the column name. Any
name acceptable as an HDF5 link name (UTF-8, excluding `/` and NUL) is
permitted. Producers MUST NOT use any HEP001 reserved name
({ref}`hep001-reserved-names-list`) as a column name. Producers SHOULD also
avoid names that begin with an underscore, which are reserved for
Anndata-aligned attribute names (`_index`) and may be claimed by future HEPs.

(hep001-column-discovery)=
### Column discovery

The only HDF5 datasets that are direct children of a table group are its column
datasets: categories datasets live one level down inside the `CATEGORIES`
subgroup ({numref}`§%s <hep001-categoricals>`), search-index datasets inside
the `SEARCH_INDEXES` subgroup ({numref}`§%s <hep001-search-indexes>`), and a
list column's member datasets inside its list column group
({numref}`§%s <hep001-list-columns>`). A consumer therefore enumerates a
table's columns as: the rank-1 datasets that are direct children of the
table group, plus the direct-child groups whose `CLASS` attribute equals
`LIST_COLUMN`. The `CATEGORIES` and `SEARCH_INDEXES` children carry no
`CLASS` attribute and are skipped automatically.

When `column-order` ({numref}`§%s <hep001-table-group>`) is present it lists
exactly these columns and fixes their order; when absent, the set of
columns is still fully determined by this rule, and only their order is
implementation-defined.

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
   SHOULD prefer fixed-length equivalents where possible. For ragged
   per-row sequences in particular, producers SHOULD use a list column
   ({ref}`hep001-list-columns`) instead of a variable-length-typed column
   dataset; within list columns, variable-length datatypes are prohibited
   outright ({numref}`§%s <list-no-vlen>`).

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
outside the column's logical value range. Boolean columns are the one
exception: they have no out-of-domain value and MUST NOT declare a fill
value ({numref}`§%s <hep001-booleans>`). A producer MAY declare that range
explicitly with the two attributes `valid_min` and `valid_max` on the column
dataset (each a scalar of the column's element datatype). If present, the chosen
fill value MUST lie strictly outside `[valid_min, valid_max]`.

(canonical-missing-test)=
Consumers MUST retrieve the fill value — via `H5Dget_create_plist`
followed by `H5Pget_fill_value`, using `H5Pfill_value_defined` to
confirm the `H5D_FILL_VALUE_USER_DEFINED` state — and identify
missing rows with the canonical missing-value test:

```
missing(value, fill_value) = isnan(fill_value) ? isnan(value) : value == fill_value
```

For every column whose fill value is not a NaN bit pattern — including
every column whose datatype is not floating-point, and every column whose
producer chose any of the recommended fill values in
{numref}`Table %s <fill-table>` — the `isnan(fill_value)` branch is always
false and the test reduces to ordinary bit-equality `value == fill_value`.
The same applies even to columns that contain no missing values — no row
satisfies the test, and the column simply has no missing rows.

For floating-point columns with a NaN bit pattern as the
fill value, the test reduces to `isnan(value)`. IEEE 754 makes
`NaN != NaN`, so a literal `value == fill_value` test would miss every
fill-marked row in such columns; consumers MUST use the `isnan(value)`
branch instead. HEP001 does not recommend NaN as a fill value — it
conflates "the producer marked this row missing" with "the result of a
floating-point computation was indeterminate" — but it permits NaN for
producers who need a zero-cost round-trip with NaN-native ecosystems.

For producers that have no domain-specific constraint forcing a different
choice, the table below lists recommended fill values for each datatype
family.

```{table} Recommended fill values
:label: fill-table

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
```

The recommended sentinels above are chosen as follows.

**Integers.** Signed-integer sentinels are `INT*_MIN + 1` rather than
`INT*_MIN` itself, leaving a one-value safety margin against operations
that land on the type's minimum (e.g., `abs(INT*_MIN)` is undefined
behavior in C). Unsigned sentinels are the type's maximum.

**Floating point.** Two choices are acceptable:

1. The recommended non-NaN sentinel `9.9692099683868690e+36`. Exactly
   representable in both `float32` and `float64`, and preserves bit-identity
   under width casts, so the equality branch of the canonical
   missing-value test works without rounding-tolerance gymnastics. This
   is the recommended choice for new tables that do not need byte-level
   round-tripping with NaN-native ecosystems.
2. Any NaN bit pattern (quiet, signaling, signed, with arbitrary
   payload). The canonical missing-value test then takes the
   `isnan(value)` branch. This choice is permitted but not recommended
   on its own merits; producers should use it when zero-cost
   interoperability with pandas, Anndata, NumPy, R, or other NaN-as-missing
   ecosystems matters more than disambiguating "missing" from "result
   of an indeterminate computation".

**Strings.** The empty string is the recommended sentinel because it is
rarely a meaningful value in practice.

**Enumerations.** The recommended sentinel is a designated enum member
named `MISSING`. Producers using an enum column with missing values MUST
include such a member in the enum type definition; the integer code
backing the `MISSING` member then serves as the column's fill value.

For datatypes not in the table:

* `float16` is too narrow for a generic sentinel.
* Boolean columns cannot represent a missing sentinel alongside their
  two valid values and declare no fill value at all. For nullable
  boolean data, use one of the two forms defined in
  {numref}`§%s <hep001-booleans>` (a three-member enumeration with
  `MISSING`, RECOMMENDED, or a `uint8` column with fill `255`).

Producers whose column domain includes any of the recommended sentinels
above MUST choose a different fill value via `H5Pset_fill_value`. For
columns with a natural numeric range (integer and floating-point
columns), producers MUST also declare that range via `valid_min` and
`valid_max` and choose the fill outside it. For columns without a
natural numeric range (strings, opaque), the alternative fill value
alone constitutes the override; no additional attribute is required.

```{note}
This is a conscious divergence from Anndata's nullable-integer and
nullable-boolean encodings, which add a companion boolean mask. For
column datasets, HEP001 relies on fill values alone. List columns are
the one exception: a list column's optional `MASK` member dataset marks
null lists ({numref}`§%s <list-column-members>`), because a `uint64`
offsets encoding has no spare value domain for a fill sentinel that
would distinguish a null list from an empty one.
```

### Column attributes

A column dataset MAY carry any of the following attributes.

`SEARCH_INDEX_LIST`
: One-dimensional array of HDF5 object references. Each reference points
  to a search-index dataset (see {numref}`§%s <hep001-search-indexes>`) in the
  `SEARCH_INDEXES` subgroup that accelerates queries on this column.

`CATEGORIES`
: Scalar HDF5 object reference. Used only for categorical columns. Points
  to the dataset that holds the categorical values (see
  {numref}`§%s <hep001-categoricals>`).

`valid_min` / `valid_max`
: Scalar attributes of the same datatype as the column, specifying the minimum
  and maximum range of the column's values. See {numref}`§%s <fill-vals>`.

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

(hep001-categoricals)=
### Categorical columns

A categorical column stores integer codes that index into a separate
*categories dataset* holding the actual label values. The code at row `i`
is the zero-based position of that row's label in the categories dataset.

#### The `CATEGORIES` group

A table group MAY contain a direct child group named `CATEGORIES`. When
present, it MUST hold every categories dataset for the table, and no other
objects. It carries no required attributes of its own. Categories datasets
in `CATEGORIES` MAY have any name; the binding between a column and its
categories dataset is the column's `CATEGORIES` object reference
({numref}`§%s <hep001-object-references>`), never a parsed name.

#### Categorical column requirements

A categorical column MUST:

1. Have an integer datatype (any width, signed or unsigned). The column's
   missing value (the dataset's fill value) denotes a missing category.
2. Carry a scalar `CATEGORIES` attribute whose value is an HDF5 object
   reference resolving to a categories dataset in the table group's
   `CATEGORIES` subgroup.

A categories dataset MUST:

1. Live directly under the table group's `CATEGORIES` subgroup.
2. Have rank 1, with any datatype appropriate to the label values.

A categories dataset MAY:

1. Carry a scalar boolean attribute `ordered` (matching Anndata's ordered
   categoricals), encoded per {numref}`§%s <hep001-boolean-attributes>`.
   Producers MUST set `ordered` to true exactly when the order of entries
   in the categories dataset is semantically meaningful.
2. Back more than one categorical column. Several categorical columns that
   share a common label set MAY reference the same categories dataset
   through their `CATEGORIES` attributes; producers need not duplicate a
   shared code book.

A categories dataset is not a column dataset: it does not count toward the
table's columns, and it MUST NOT appear in the table group's `column-order`
({numref}`§%s <hep001-consistency>`).

```{mermaid}
graph LR
  col(["/my_table/label<br/>(int8 column)<br/>CATEGORIES &rarr; CATEGORIES/label__CATEGORIES"])
  cg[["/my_table/CATEGORIES<br/>(group)"]]
  cat>"label__CATEGORIES<br/>(fixed UTF-8)<br/>ordered=false"]
  cg --> cat
  col -- object ref --> cat

  classDef dataCol  fill:#D4EBF8,stroke:#7FA9D0,stroke-width:1px,color:#000
  classDef catGroup fill:#F5E2D0,stroke:#C59D85,stroke-width:2px,color:#000
  classDef catData  fill:#FAE0D4,stroke:#D09F90,stroke-width:1px,color:#000

  class col dataCol
  class cg catGroup
  class cat catData
```

(hep001-booleans)=
### Boolean columns

A *boolean column* is a column dataset whose datatype is the HEP001
boolean datatype {numref}`§%s <hep001-boolean-attributes>`. The
producer-strict, consumer-lenient rules defined there apply: producers
MUST write exactly that datatype; consumers MUST accept the tolerated
one-byte variants. An enumeration datatype with any other member set is
not a boolean column and is governed by the ordinary rules for
enumeration columns.

#### Values

Every element of a boolean column at a row position in `[0, NROWS)` MUST
be `0` or `1`. A boolean column holding any other value at those
positions is non-conformant, and consumers MUST NOT interpret such a
value as either `FALSE` or `TRUE`: silent coercion would let two
conformant consumers return different results for the same query and
would make the canonical byte representation of
{numref}`§%s <bloom>` ambiguous. Rows in `[NROWS, extent)` are reserved
storage and exempt ({numref}`§%s <hep001-nrows>`).

#### Missing values

A boolean column cannot represent missing values: both values of its
domain are meaningful, so no in-domain sentinel exists. A boolean column
MUST NOT declare a user-defined fill value, and the requirements of
{numref}`§%s <fill-vals>` do not apply to it — boolean columns are the
one column datatype exempt from that section's mandatory fill value.

A producer that needs a *nullable* boolean MUST use one of the following
two forms instead. Neither is a boolean column.

1. RECOMMENDED: an enumeration column with exactly three members —
   `FALSE` = 0, `TRUE` = 1, `MISSING` = 2 — declaring the integer code
   of `MISSING` as the fill value, per the enumeration convention of
   {numref}`§%s <fill-vals>`. This form is self-describing: the member
   names preserve the column's boolean semantics for any consumer.
2. An integer column of datatype `uint8` with values `0` and `1` and a
   declared fill value outside `{0, 1}` (`255` RECOMMENDED, matching the
   recommended `uint8` fill of {numref}`Table %s <fill-table>`), for
   pipelines that prefer plain integers at the cost of self-description.

#### Interaction with search indexes

For the purposes of {numref}`§%s <hep001-search-indexes>`:

* **Ordering** (`SORTED_ROWS`, `CHUNK_MINMAX`): boolean values order by
  their integer codes; `FALSE` sorts before `TRUE`.
* **`CHUNK_MINMAX`**: the `min` and `max` fields use the column's
  datatype. Min/max pruning over a two-value domain is rarely useful;
  producers SHOULD prefer `BITMAP` for boolean columns.
* **`CHUNK_BLOOM`**: the canonical byte representation of a boolean
  value is a single byte, `0x00` for `FALSE` and `0x01` for `TRUE`
  ({numref}`§%s <bloom>`). Boolean columns are the one enumeration
  datatype permitted to carry a `CHUNK_BLOOM` index.
* **`BITMAP`**: permitted; the accompanying values dataset holds at most
  two entries.

(hep001-list-columns)=
## List columns

A *list column* stores a variable-length list of values in each row —
`list<float32>`, `list<utf8>`, `list<list<int16>>`, and so on. Unlike every
other column in this specification, a list column is not a single rank-1
dataset: it is an HDF5 group that is a direct child of the table group.
The group's link name is the column name, and the member datasets inside
the group together encode the rows' lists.

The encoding is the offsets layout of the Apache Arrow columnar format.
All elements of all rows are stored back-to-back in a flattened `VALUES`
member, in row order, and a monotonic `OFFSETS` dataset records each row's
slice: row `i`'s list occupies elements `[OFFSETS[i], OFFSETS[i+1])` of the
`VALUES` member. Access is positional and `O(1)` per row, and a `VALUES`
dataset is an ordinary HDF5 dataset with its own datatype, chunk shape, and
filter pipeline. The same per-column storage independence that motivates HEP001
extends to list elements. Because the `VALUES` member MAY itself be another
list encoding, lists nest to arbitrary depth: JSON-style arrays of arrays
of any depth fit naturally, provided the innermost elements share one
datatype.

(list-column-identification)=
### Identification

Every list column group MUST carry the following two attributes:

`CLASS`
: Scalar, null-terminated fixed-length ASCII string attribute with value
  `LIST_COLUMN`. A consumer MUST identify a list column as a direct child
  group of a table group whose `CLASS` attribute equals `LIST_COLUMN`.
  Groups deeper inside a list column MAY also carry `CLASS="LIST_COLUMN"`
  (nested lists, {numref}`§%s <list-column-members>`); they are part of
  their outermost list column's encoding and are not themselves columns
  of the table.

`KIND`
: Scalar, null-terminated fixed-length ASCII string attribute naming the
  storage method used inside the group. The only value defined by this
  revision is `OFFSETS`, specified in this section. Future revisions MAY
  register additional values. A consumer that encounters a `KIND` it does
  not implement MUST NOT silently drop the column; it SHOULD surface the
  column as present but unreadable.

A producer MUST NOT write `CLASS="LIST_COLUMN"` on any group that does not
satisfy the rest of this section.

(list-column-members)=
### Members of a list column group

A `KIND="OFFSETS"` list column group contains the following members, and
no other objects. The member dataset names are reserved names
({ref}`hep001-reserved-names-list`).

`OFFSETS`
: REQUIRED. A rank-1 dataset of datatype `uint64`. `OFFSETS[0]` MUST be
  `0`, and values MUST be monotonically non-decreasing over positions
  `[0, L]`, where `L` is the number of entries at this level
  ({numref}`§%s <list-level-length>`). Entry `i` occupies elements
  `[OFFSETS[i], OFFSETS[i+1])` of the `VALUES` member; an empty list has
  `OFFSETS[i+1] = OFFSETS[i]`. The first-dimension extent MUST be
  `≥ L + 1`; positions beyond `L` are reserved storage
  ({numref}`§%s <write-workflow>`).

`VALUES`
: REQUIRED. The flattened elements of all entries, in entry order.
  Exactly one of:

  * a rank-1 HDF5 dataset — the *leaf* case. The element datatype MAY be
    any datatype permitted for a column dataset except variable-length
    datatypes ({numref}`§%s <list-no-vlen>`);
  * a group with `CLASS="LIST_COLUMN"` and `KIND="OFFSETS"` — the
    elements are themselves lists (nested lists);
  * a group with `CLASS="STRING_VALUES"` — the elements are
    variable-length UTF-8 strings ({numref}`§%s <string-values>`).

`MASK`
: OPTIONAL. A rank-1 dataset of the boolean datatype of
  {numref}`§%s <hep001-boolean-attributes>`, with first-dimension extent
  `≥ L`. `MASK[i] = FALSE` declares entry `i` *null* (missing);
  `MASK[i] = TRUE` declares it present. When `MASK` is absent, every
  entry at this level is present. For a null entry the producer MUST
  write `OFFSETS[i+1] = OFFSETS[i]`, so that null and empty entries both
  occupy zero elements of `VALUES` and only `MASK` distinguishes them.

Each member dataset MAY independently select its chunk shape, dataset
creation properties, and filter pipeline, per the same rule that governs
column datasets. Producers SHOULD NOT rely on fill values of `OFFSETS`
or `MASK` datasets; missingness at each level is expressed only as
specified here.

(list-level-length)=
### Levels, entries, and nesting

The entries of the *top level* of a list column are the table's rows:
`L = NROWS`, and the same `[0, NROWS)` / reserved-tail semantics that
govern column datasets ({numref}`§%s <hep001-nrows>`) govern the top-level
`OFFSETS` and `MASK`.

When a `VALUES` member is itself a list encoding (a nested `LIST_COLUMN`
or a `STRING_VALUES` group), the entries of that nested level are the
*elements* of its parent level: its entry count is
`L_nested = OFFSETS_parent[L_parent]`. Its own `OFFSETS` extent MUST be
`≥ L_nested + 1` and its `MASK` extent, when present, `≥ L_nested`. This
rule applies recursively to any depth. A nested `MASK` distinguishes a
null inner list from an empty inner list at that level.

Missing values at the three structural positions are therefore expressed
as: a *null list* by the `MASK` of the level above the list; a *null
string element* by the `MASK` of the `STRING_VALUES` group; and a
*missing leaf element* by the fill value of the leaf `VALUES` dataset,
using the canonical missing-value test of {numref}`§%s <fill-vals>`
unchanged. The leaf `VALUES` dataset MAY carry `valid_min` / `valid_max`
per {numref}`§%s <fill-vals>`.

(string-values)=
### String elements — the `STRING_VALUES` group

Variable-length string elements are stored without HDF5 variable-length
datatypes, as a second offsets level over a byte buffer. A group with
`CLASS="STRING_VALUES"` (scalar, null-terminated fixed-length ASCII
string attribute) contains exactly:

`OFFSETS`
: REQUIRED. As in {numref}`§%s <list-column-members>`: rank-1 `uint64`,
  `OFFSETS[0] = 0`, monotonic, extent `≥ L + 1` for `L` entries. Entry
  `j` is the byte range `[OFFSETS[j], OFFSETS[j+1])` of `CHARS`.

`CHARS`
: REQUIRED. A rank-1 dataset of datatype `uint8` holding the UTF-8
  encoding of all entries, back to back — with no byte-order mark and no
  Unicode normalization (no NFC, NFD, NFKC, or NFKD conversion), matching
  the string rules used elsewhere in this specification. Because the
  element type is `uint8`, the buffer passes through the filter pipeline
  and compresses like any other dataset.

`MASK`
: OPTIONAL. As in {numref}`§%s <list-column-members>`; distinguishes a
  null string entry from an empty one.

Producers whose string elements have a known maximum byte length SHOULD
consider the simpler leaf alternative: a `VALUES` dataset of a
fixed-length UTF-8 string datatype sized to that bound
(one UTF-8 code point can occupy up to four bytes). Fixed-length string
elements pass through the filter pipeline directly and need no
`STRING_VALUES` group; the empty-string missing-value convention of
{numref}`§%s <fill-vals>` applies to them unchanged.

(list-no-vlen)=
### No variable-length datatypes

A producer MUST NOT use any HDF5 variable-length datatype — variable-length
strings or variable-length sequences of any base type — anywhere below a
list column group. Every conformant list column subtree is therefore free
of global-heap storage.

This rule is stricter than the SHOULD-level discouragement that applies to
column datasets, and it is deliberate: list columns exist to make
variable-length data fast, and variable-length datatypes are not. Their
element bytes live on the HDF5 global heap, outside the dataset's chunks,
so the filter pipeline compresses only the heap references while the
data itself is stored uncompressed with per-object heap overhead. Heap
access defeats direct chunk I/O and cloud-optimized readers. Parallel
HDF5 collective I/O cannot write variable-length data. Heap space
freed by rewrites is not reclaimed short of `h5repack`. The `OFFSETS` +
`CHARS` encoding of {numref}`§%s <string-values>` and fixed-length
element datatypes together cover the same use cases without any of
these costs.

### List column attributes

A list column group MAY carry the descriptive attributes `description`,
`units`, and `units_vocabulary` with the same meanings they have on column
datasets; `units` describes the leaf elements. A list column group MUST NOT
carry `SEARCH_INDEX_LIST` — this revision defines no search indexes over
list columns; a future revision MAY. A list column MUST NOT be referenced
by the table group's `INDEX_COLUMNS` attribute — lists have no
HEP001-defined order, so they cannot serve as row labels. A list column
SHOULD appear in `column-order` by name, like any other column.

### Reading a list column

To read row `i` of a `KIND="OFFSETS"` list column, a consumer:

1. reads `MASK[i]` if `MASK` is present `FALSE` yields a null list;
1. reads `OFFSETS[i]` and `OFFSETS[i+1]`; equal values with a present (or
absent) mask yield an empty list;
1. reads elements `[OFFSETS[i], OFFSETS[i+1])` of `VALUES`, recursing per level
when `VALUES` is a group.

Since step 1 matters only when the slice is empty, consumers MAY defer it and
typical rows cost one `OFFSETS` read plus one `VALUES` read — the same two
dependent reads that any variable-length encoding requires, with the first
falling on a small, cache-friendly dataset.

```{mermaid}
graph TD
  TG[["table group"]]
  LC[["readings (list column group)<br/>CLASS='LIST_COLUMN'<br/>KIND='OFFSETS'"]]
  OF>"OFFSETS (uint64, N+1)"]
  MK>"MASK (bool enum, N, optional)"]
  VL>"VALUES (float32, M)"]
  LC2[["tags (list column group)<br/>CLASS='LIST_COLUMN'<br/>KIND='OFFSETS'"]]
  OF2>"OFFSETS (uint64, N+1)"]
  SV[["VALUES (group)<br/>CLASS='STRING_VALUES'"]]
  OF3>"OFFSETS (uint64, M2+1)"]
  CH>"CHARS (uint8, B)"]

  TG --> LC
  TG --> LC2
  LC --> OF
  LC --> MK
  LC --> VL
  LC2 --> OF2
  LC2 --> SV
  SV --> OF3
  SV --> CH

  classDef tableGroup fill:#FFF4D4,stroke:#D4B86A,stroke-width:2px,color:#000
  classDef listGroup  fill:#DDEBDD,stroke:#7CB78A,stroke-width:2px,color:#000
  classDef dataCol    fill:#D4EBF8,stroke:#7FA9D0,stroke-width:1px,color:#000

  class TG tableGroup
  class LC,LC2,SV listGroup
  class OF,MK,VL,OF2,OF3,CH dataCol
```

(hep001-indexes)=
## Row index columns

A row index column is a column dataset referenced by the table group's
`INDEX_COLUMNS` attribute (see {numref}`§%s <hep001-table-group>`). Row index
columns supply row labels for the table as a whole — every column in the table
is labeled by every row index column. They are ordinary column datasets in every
other respect, and they SHOULD also appear in `column-order` like any other
column.

### Hierarchy

When `INDEX_COLUMNS` contains more than one reference, the order is the
row-label hierarchy from outermost to innermost level. For example,
`INDEX_COLUMNS = [ref(donor_id), ref(sample_id), ref(cell_id)]` declares
a three-level row index in which `donor_id` is the outermost grouping
and `cell_id` is the innermost row identifier.

### Typical uses

* **Single row index.** The common case: one column (often a string
  of sample IDs, or an unsigned-integer ordinal) supplies row labels
  for the entire table.
* **Hierarchical row index.** A small number of tables have a
  meaningful row-label hierarchy — for example, donor → sample → cell
  in single-cell genomics, or year → quarter → ticker in financial
  time series. `INDEX_COLUMNS` lists the level columns in order.
* **No row index.** A table whose rows are identified solely by their position
  (the *N*-th row is "row *N*") SHOULD omit `INDEX_COLUMNS` entirely. Producers
  MAY equivalently write `INDEX_COLUMNS` as an empty 1-D object-reference
  array; consumers MUST treat the two forms as semantically identical.

```{mermaid}
graph TD
  TG[["table group<br/>INDEX_COLUMNS =<br>[ref(donor_id),<br>ref(sample_id)]"]]
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
present, it MUST hold every search-index dataset for the table, together with
any *accompanying datasets* those search indexes require and no other objects.
It carries no required attributes of its own. Datasets in `SEARCH_INDEXES` MAY
have any name.

A search-index dataset is distinguished from an accompanying dataset by the
`KIND` attribute ({numref}`§%s <common-idx-attrs>`): every search-index
dataset MUST carry `KIND`, and an accompanying dataset MUST NOT carry `KIND`.

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

```{note}
The dataset names used throughout this specification's examples and
diagrams — `ts__chunk_minmax`, `energy__sorted_rows`, `label__CATEGORIES`,
`label__bitmap__VALUES`, and similar `<column>__<role>` forms — are an
illustrative convention only. HEP001 assigns them no meaning: a
search-index dataset MAY have any name (see above), and a categories
dataset's name is likewise unconstrained ({ref}`hep001-categoricals`).
The authoritative linkage between a column and its search indexes,
categories dataset, or values dataset is always the relevant HDF5 object
reference (`SEARCH_INDEX_LIST`, `CATEGORIES`, `VALUES`), never a parsed
name. Because column names are arbitrary UTF-8 and MAY themselves contain
`__`, consumers MUST NOT infer any association from dataset names and MUST
follow the object references instead.
```

```{mermaid}
graph LR
  C1(["ts (column)<br/>SEARCH_INDEX_LIST &rarr; ts__chunk_minmax"])
  C2(["energy (column)<br/>SEARCH_INDEX_LIST &rarr; energy__sorted_rows"])
  SI1>"ts__chunk_minmax<br/>(in SEARCH_INDEXES)"]
  SI2>"energy__sorted_rows<br/>(in SEARCH_INDEXES)"]
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
| `CHUNK_MINMAX`       | Per-chunk min and max of an orderable column | {numref}`§%s <chunk-minmax>` |
| `SORTED_ROWS`        | Row-position permutation of a column        | {numref}`§%s <sorted-row>`   |
| `BITMAP`             | Per-value bitmap of a low-cardinality col.  | {numref}`§%s <bitmap>`       |
| `CHUNK_BLOOM`        | Per-chunk Bloom filter of a column          | {numref}`§%s <bloom>`        |

Future HEPs MAY register additional `KIND` values. Consumers MUST treat
unknown `KIND` values as "ignore this search index".

Every search-index dataset MUST also carry the two validity-token attributes
`SOURCE_GENERATION` and `SOURCE_NROWS` ({numref}`§%s <validity-tokens>`). A
consumer MUST evaluate the validity check defined there before using any search
index.

The shapes of search-index datasets grow with the table: `CHUNK_MINMAX`
and `CHUNK_BLOOM` track the source column's data-bearing chunk count,
`SORTED_ROWS` and `BITMAP` track `NROWS`. Producers SHOULD therefore
create search-index datasets chunked, with the growing dimension
unlimited, so appends can extend them in place; a producer MUST NOT instead
delete and recreate an index dataset on each update.

A search-index dataset MAY also carry a `description` attribute (scalar
fixed-length UTF-8 string), per the descriptive-annotation convention
used elsewhere in this spec (see {ref}`hep001-reserved-names`, rule 4).
Producers SHOULD use `description` to record provenance — the timestamp
of index construction, the producer software and version, and any
hyperparameters not captured by the index family's own attributes.
Consumers MAY ignore `description` for query-execution purposes; it is
purely informational.


(chunk-minmax)=
### Chunk min/max search index (`KIND = CHUNK_MINMAX`)

A chunk min/max index accelerates range and equality predicates over an
orderable column by letting the engine skip chunks whose value range does
not overlap the predicate. The column's datatype MUST have a HEP001-defined
order — the same orders enumerated for `SORTED_ROWS` under *Ordering* in
{numref}`§%s <sorted-row>` (integers, floating-point, boolean, strings,
opaque, and enumerations) — and `min` and `max` are computed under that
order. The datatypes that `SORTED_ROWS` excludes for lack of a defined
order (object and region references, compound, array, and
variable-length-array datatypes) likewise MUST NOT carry a `CHUNK_MINMAX`
index.

**Shape:** The search-index dataset is 1-D with length equal to the number of
chunks of the source column dataset that contain logical-table rows:

* `ceil(NROWS / chunk_length)` for a chunked column,
* `1` for a contiguous column (or `0` when `NROWS = 0`).

Chunks lying entirely in the column's preallocated tail
(`[NROWS, extent)`, see {numref}`§%s <hep001-nrows>`) are not indexed.

**Datatype:** An HDF5 compound datatype with the following fields, in
declaration order:

* `min` — same datatype as the source column's element type.
* `max` — same datatype as the source column's element type.
* `nan_count` — `uint64`. The number of IEEE 754 NaN values in the chunk,
  regardless of whether NaN is the column's fill value. For
  non-floating-point columns this field MUST be present and set to 0.
* `fill_count` — `uint64`. The number of elements in the chunk that the
  canonical missing-value test ({numref}`§%s <fill-vals>`) classifies as
  missing. When the column's fill value is itself a NaN bit pattern, every
  NaN element is missing by definition, and `fill_count` equals `nan_count`
  for that chunk; this overlap is intentional and consumers MAY rely on
  `fill_count` alone as the chunk's missing-row count regardless of the
  column's fill choice.
* `n` — `uint64`. The number of logical rows (i.e., rows in
  `[0, NROWS)`; see {numref}`§%s <hep001-nrows>`) covered by this chunk.
  For every chunk other than the last, `n` equals the column's chunk
  length; for the last data-bearing chunk, `n` MAY be strictly less than
  the chunk length when `NROWS` is not a multiple of it.

**Semantics:** `min` and `max` are computed over only those elements of
the chunk that the canonical missing-value test ({numref}`§%s <fill-vals>`)
classifies as non-missing, and that are themselves orderable. For
floating-point columns this means: NaN values are excluded (because IEEE 754
does not order them, regardless of whether NaN is the column's fill value),
and elements that match the column's non-NaN fill value (if any) are
excluded. For integer and other orderable types, only elements equal to
the column's fill value are excluded. When the chunk has no non-missing,
orderable elements (`fill_count + nan_count == n` for floating-point
columns, or `fill_count == n` for other orderable types), `min` and `max`
MUST be set to the column's fill value. In that case `min` and `max` are
placeholders with no semantic meaning: consumers MUST recognize such
chunks from the `fill_count`, `nan_count`, and `n` fields and MUST NOT
use the placeholder values as bounds in pruning decisions.

**Applicability:** Each `CHUNK_MINMAX` search-index dataset applies to
exactly one column, identified by the column whose `SEARCH_INDEX_LIST`
references it. The column's HDF5 datatype MUST have a HEP001-defined
order, as enumerated for `SORTED_ROWS` under *Ordering*
({numref}`§%s <sorted-row>`); otherwise no `CHUNK_MINMAX` index is
permitted. A producer MAY build separate `CHUNK_MINMAX` indexes
for several columns, but a single search-index dataset MUST NOT cover
multiple columns because chunks of different columns are independent.

**Additional attributes on the search-index dataset:**

* `KIND` — `"CHUNK_MINMAX"` (see {numref}`§%s <common-idx-attrs>`).

All other per-index data lives in the compound datatype's fields
(`min`, `max`, `nan_count`, `fill_count`, `n`); these are HDF5 datatype
fields, not HDF5 attributes.

(sorted-row)=
### Sorted-row permutation index (`KIND = SORTED_ROWS`)

A sorted-row index stores row positions in sorted order of a column's
values, enabling binary search and range scans without reading the full
column.

**Shape:** 1-D, length equal to `NROWS` (see {numref}`§%s <hep001-nrows>`).
Rows of the source column lying in the preallocated tail
`[NROWS, extent)` are not part of the permutation.

**Datatype:** An unsigned integer wide enough to address every row of the
source column (typically `uint64`).

**Semantics:** Element `i` of the index is the row position `r` such
that, under the ordering defined below, the `i`-th rank of the column's
values lives at row `r`. Ties between rows with identical values MUST
be broken by increasing `r`, so the permutation is total and
deterministic.

**Ordering:** A `SORTED_ROWS` index MUST only be built over a column
whose HDF5 datatype has a HEP001-defined order, as enumerated below:

* **Signed and unsigned integers** (any width): standard arithmetic
  order.
* **Floating-point values** (`float16`, `float32`, `float64`):
  IEEE 754 numerical order over finite values and the two infinities.
  Negative zero MUST compare equal to positive zero. `NaN` values are
  not ordered; rows whose value is `NaN` MUST be placed at the end of
  the permutation, in increasing `r` order, and the `nan_tail_length`
  attribute MUST record the count. The NaN tail and the fill tail are
  always disjoint by construction: a row is placed in the NaN tail if
  its value is `NaN`, and otherwise in the fill tail if its value equals
  a non-`NaN` fill. When the column's fill value is itself `NaN`, every
  missing row is a `NaN` row and is placed in the `NaN` tail;
  `fill_tail_length` is then `0`, and `nan_tail_length` equals the
  column's total missing-row count.
* **Boolean values** (boolean columns, {numref}`§%s <hep001-booleans>`):
  ordered by integer code; `FALSE` (`0`) sorts before `TRUE` (`1`).
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

Rows whose value matches a non-NaN column fill value under the
canonical missing-value test ({numref}`§%s <fill-vals>`) MUST appear
immediately before the NaN tail (if any), in increasing `r` order,
and the `fill_tail_length` attribute MUST record the count. For
datatypes that have no NaN concept, the NaN tail is empty
(`nan_tail_length = 0`) and the fill-tail rows appear at the very end
of the permutation. When the column's fill value is itself NaN, all
missing rows are NaN rows and are placed in the NaN tail (see the
floating-point ordering rule above); the fill tail is then empty
(`fill_tail_length = 0`).

**Additional attributes:**

* `KIND` — `"SORTED_ROWS"`.
* `nan_tail_length`, `fill_tail_length` — scalar `uint64`. Both MUST be
  present; either MAY be 0.
* `ordered` — scalar boolean ({numref}`§%s <hep001-boolean-attributes>`).
  MUST be true for `SORTED_ROWS`; reserved for future indexes that permit
  partial orderings.

**Applicability:** Each `SORTED_ROWS` search-index dataset applies to
exactly one column, identified by the column whose `SEARCH_INDEX_LIST`
references it. The column's HDF5 datatype MUST have a HEP001-defined
order, as enumerated under *Ordering* above; otherwise no
`SORTED_ROWS` index is permitted.


(bitmap)=
### Bitmap index (`KIND = BITMAP`)

A bitmap index accelerates equality predicates on low-cardinality
columns.

**Shape:** 2-D of shape `(K, ceil(NROWS / 8))`, where `K` is the number of
distinct values (or categories) indexed and `NROWS` is the table's row
count (see {numref}`§%s <hep001-nrows>`). Rows of the source column
lying in the preallocated tail `[NROWS, extent)` are not indexed.

**Datatype:** `uint8`. Bit `r % 8` (where bit 0 is the byte's least
significant bit) of byte `r / 8` of row `k` is set if the column's value
at row `r` equals the `k`-th indexed value. Because the storage element
is `uint8`, HDF5 performs no byte swapping on read or write — the bytes
on disk are exactly the bytes the producer wrote. When `NROWS` is not a
multiple of 8, the bits at positions `≥ NROWS` in each row's final byte
are padding: producers MUST write them as `0`, and consumers MUST
ignore them regardless of their value.

**Accompanying values dataset:** A sibling 1-D dataset, under `SEARCH_INDEXES`,
holds the `K` indexed values in the same datatype as the source column. Its name
is linked from the bitmap via a scalar `VALUES` object-reference attribute. This
values dataset is an *accompanying dataset*, not a search-index dataset: it MUST
NOT carry a `KIND` attribute (see {numref}`§%s <hep001-search-indexes>`).

**Additional attributes on the bitmap dataset:**

* `KIND` — `"BITMAP"`.
* `VALUES` — scalar HDF5 object reference to the values dataset.
* `ordered` — scalar boolean ({numref}`§%s <hep001-boolean-attributes>`).
  When `true`, the entries in the values dataset (linked via `VALUES`) are
  listed in a semantically meaningful
  order, for example, a numerically-sorted set of distinct values or
  an ordinal category sequence such as `["low", "medium", "high"]`.
  The bitmap's `k`-th row corresponds to the `k`-th entry of the values
  dataset under that order. When `false` or absent, the order of the
  values dataset is arbitrary (typically insertion order) and consumers
  MUST NOT infer any semantic ordering from it.
* `exhaustive` — scalar boolean ({numref}`§%s <hep001-boolean-attributes>`).
  When `true`, the values dataset enumerates every distinct non-missing
  value that occurs in rows `[0, NROWS)` of the source column, so a
  query value absent from the values dataset provably matches zero
  rows. When `false` or absent, the enumeration MAY be partial — for
  example, only the most frequent values — and consumers MUST treat a
  query value absent from the values dataset as unindexed (falling back
  to a scan), not as proof of absence.

**Applicability:** Each `BITMAP` search-index dataset applies to
exactly one column, identified by the column whose `SEARCH_INDEX_LIST`
references it.


(bloom)=
### Per-chunk Bloom filter index (`KIND = CHUNK_BLOOM`)

A per-chunk Bloom filter accelerates equality predicates on
high-cardinality columns by giving a fast negative answer for chunks
that provably do not contain the queried value.

**Shape:** 2-D of shape `(n_chunks, m_bytes)`, where `n_chunks` is the
number of chunks of the source column that contain logical-table rows
(`ceil(NROWS / chunk_length)` for a chunked column, `1` for a contiguous
column, or `0` when `NROWS = 0`; see {numref}`§%s <hep001-nrows>`) and
`m_bytes` is the Bloom-filter byte length per chunk (constant across
chunks). Chunks lying entirely in the column's preallocated tail
`[NROWS, extent)` are not indexed.

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
  **little-endian**. `NaN` values MUST NOT be inserted into a Bloom filter
  — different `NaN` bit patterns would yield different filter bits and
  break interoperability — and consumers MUST NOT query for `NaN` against
  a `CHUNK_BLOOM` index. The rule applies whether `NaN` is a real-data
  value in the column or the column's chosen fill value: in the latter
  case, `NaN` elements are missing values, and missing values are by
  convention excluded from Bloom-filter membership testing anyway.
  Producers MUST normalize negative zero to positive zero before hashing.
* **Boolean values** (boolean columns, {numref}`§%s <hep001-booleans>`):
  a single byte, `0x00` for `FALSE` or `0x01` for `TRUE`.
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
* **Compound, array, and variable-length-array datatypes, and enumerations other
  than the boolean datatype ({numref}`§%s <hep001-boolean-attributes>`):** out
  of scope for this revision. Producers MUST NOT build a `CHUNK_BLOOM` index
  over such columns. A future HEP MAY register canonical encodings for these
  cases.

**Additional attributes:**

* `KIND` — `"CHUNK_BLOOM"`.
* `k` — scalar `uint16`. The number of hash functions.
* `m_bits` — scalar `uint64`. Equal to `8 * m_bytes`; stored explicitly
  for clarity.
* `hash_family` — scalar fixed-length ASCII string; for this revision
  MUST be `"murmur3_x64_128_double"`.
* `seed` — scalar `uint32`. The seed passed to `MurmurHash3_x64_128`,
  whose reference implementation accepts a 32-bit seed. Default `0`.
  Producers MAY change the seed to mitigate adversarial inputs;
  consumers MUST read the seed from this attribute rather than
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

(write-workflow)=
## Writing and appending data

This section specifies how producers add, remove, or rewrite rows of a
HEP001 table and how consumers interpret the table during and after
those operations. The contract is anchored on the table group's `NROWS`
attribute ({numref}`§%s <hep001-nrows>`).

### How consumers interpret `NROWS`

`NROWS` is the authoritative count of rows currently in the table.
A consumer MUST treat rows `[0, NROWS)` of every column dataset as
the table's data and MUST ignore rows `[NROWS, extent)` of any
column dataset, even when those rows hold values that are not the
column's fill value. The tail is reserved storage, not data; its
contents have no semantic meaning under HEP001 and MAY contain
arbitrary bytes left behind by a previous write, a previous
truncation, or HDF5's own fill-value mechanism.

The same cutoff applies to search indexes. A consumer MUST consult
only those index entries that describe rows or chunks within
`[0, NROWS)`. Tail entries — for example, a `CHUNK_MINMAX` row
describing a chunk that lies entirely in `[NROWS, extent)` — MAY be
present as residue from preallocation or from a previous, larger table
state, and MUST be ignored.

For list columns the cutoff applies through the top-level `OFFSETS`:
rows `[0, NROWS)` define the valid slices, positions of `OFFSETS`
beyond `NROWS` (and of `MASK` beyond `NROWS`) are reserved storage, and
elements of any `VALUES` member beyond `OFFSETS[NROWS]` (applied
recursively through nested levels) have no semantic meaning and MUST be
ignored.

(validity-tokens)=
### Index validity tokens

Search indexes are derivative data, and this section places the burden
of keeping them consistent on producers. Validity tokens give consumers
a cheap, constant-time way to detect when that burden was not met — a
crashed producer, an interrupted rebuild, or a tool that modified column
data without maintaining indexes — without validating an index against
the column it describes.

#### The `GENERATION` attribute

A table group that contains one or more search-index datasets MUST carry
a scalar attribute named `GENERATION` of datatype `uint64`. A table
group with no search-index datasets MAY omit it.

`GENERATION` identifies the current state of the table's column data. A
producer MUST increment `GENERATION` by one as part of any operation
that:

1. modifies the value of any element of any column at a row position in
   `[0, NROWS)` — including the member datasets of a list column — or
2. commits a change to `NROWS` (append, {numref}`§%s <appending-rows>`;
   truncation, {numref}`§%s <hep001-truncation>`).

`GENERATION` is compared by equality only. Producers MUST NOT assign
meaning to its magnitude beyond distinguishing successive states, and
consumers MUST NOT order table states by comparing `GENERATION` values.
The initial value is unspecified; `0` is RECOMMENDED.

Schema operations — adding, removing, or renaming columns — do not
require an increment. Index linkage is per column and by object
reference ({numref}`§%s <hep001-object-references>`): removing a column
removes the `SEARCH_INDEX_LIST` attribute through which its indexes were
reachable, adding a column affects no existing index, and renaming
changes nothing an object reference resolves through. A producer MAY
increment `GENERATION` on schema operations anyway; a spurious increment
is safe — it can only disable indexes, never validate stale ones.

#### Per-index source tokens

Every search-index dataset MUST carry two scalar attributes of datatype
`uint64`:

`SOURCE_GENERATION`
: The value that `GENERATION` holds — or will hold, once the table state
  this index describes is committed — for the state the index content
  was built against.

`SOURCE_NROWS`
: The value that `NROWS` holds, or will hold, for that same state.

Building a new search index over an unchanged table is not a mutation:
the producer writes `SOURCE_GENERATION` and `SOURCE_NROWS` equal to the
table's current `GENERATION` and `NROWS` (writing `GENERATION` first if
the table did not previously carry it) and does not increment
`GENERATION`.

Because `GENERATION` is table-wide, a mutation of one column also
invalidates the indexes of columns that were not modified. Those indexes
remain factually correct; a producer restores them by rewriting their
`SOURCE_GENERATION` with the new value — an attribute-only write, no
rebuild required.

#### Consumer validity check

Before using a search-index dataset, a consumer MUST evaluate:

```
SOURCE_GENERATION == GENERATION  AND  SOURCE_NROWS == NROWS
```

If either comparison fails, or if any of the four attributes is absent
or of the wrong datatype, the consumer MUST behave as if the
search-index dataset were not present. A failed check is not an error
and MUST NOT cause the table to be rejected; it only disables
index-accelerated access. Consumers MUST NOT partially trust a stale
index — for example, by consulting a `SORTED_ROWS` permutation over the
first `SOURCE_NROWS` rows of a longer table — because the token mismatch
may equally stem from in-place modifications within those rows.

#### Write ordering

One invariant makes every crash state detectable: **at every instant, a
search-index dataset whose content does not correctly describe the
committed table state MUST carry tokens that fail the validity check.**
The invariant dictates opposite update orders for the two kinds of
mutation:

* **Mutations gated by the `NROWS` commit** (append, truncation): the
  rebuilt index describes a *future* state, so the producer MUST write
  the future-valued tokens **before** modifying the index content. The
  new `SOURCE_GENERATION` does not yet exist on the table group, so the
  index fails the check throughout its own rebuild; a crash mid-rebuild
  leaves it detectably stale rather than plausibly valid.
* **Immediately visible mutations** (in-place updates,
  {numref}`§%s <in-place-updates>`): the producer MUST commit the
  `GENERATION` increment (and flush) **before** writing any column data,
  disabling every index up front, and MUST rewrite each index's tokens
  only **after** that index's content is again correct. Writing
  current-valued tokens before the content would let a mid-rebuild index
  pass the check.

According to the append protocol ({numref}`§%s <appending-rows>`), a
crash before the `GENERATION` write leaves the old table state fully
usable with its old, still-valid indexes — appended data is invisible. A
crash between the `GENERATION` and `NROWS` writes leaves the table
readable at the old `NROWS` with no valid indexes: rebuilt indexes fail
the `SOURCE_NROWS` comparison, untouched ones the `SOURCE_GENERATION`
comparison. `SOURCE_NROWS` exists precisely for that window — with
`SOURCE_GENERATION` alone, a rebuilt index would appear valid in it. No
write ordering between the `GENERATION` and `NROWS` attributes
themselves is required: if `NROWS` happens to reach storage first and
the producer crashes, updated indexes fail on `SOURCE_GENERATION`
instead. Both orders are detectably stale.

These guarantees are conditional on the integrity of the HDF5 file
itself: HDF5 does not journal its own metadata, and the
crash-consistency statements of this section describe the HEP001-level
state of a file that survives the crash structurally intact.

```{note}
The validity check defends against accidental staleness, not against an
adversarial writer, who can trivially forge matching tokens. Consumers
processing untrusted files remain subject to the requirements of
{numref}`§%s <hep001-security>`.
```

(appending-rows)=
### Appending rows

A producer that appends `K` new rows to a table MUST perform the
operation in the following order, treating step 6 as the single commit
point that publishes the new rows to readers:

1. **Read the current `NROWS` and `GENERATION`** (call them `N_old` and
   `g_old`). A table with no search-index datasets carries no
   `GENERATION` and skips the token-related parts of steps 4–5.
2. **Extend every column dataset** so that its first-dimension extent is
   `≥ N_old + K`. A column that already has spare capacity from a prior
   preallocation needs no extension; the requirement is the
   post-condition `extent ≥ N_old + K` on every column. For every list
   column, extend the member datasets to their own post-conditions:
   top-level `OFFSETS` extent `≥ N_old + K + 1`, `MASK` (when present)
   extent `≥ N_old + K`, and each `VALUES` member (recursively) large
   enough for the elements being appended.
3. **Write the new row values** into rows `[N_old, N_old + K)` of each
   column. Writes MAY proceed in any order across columns. Within a
   list column, write leaf-first: element values at the deepest level
   first, then each enclosing level's `OFFSETS` (and `MASK`), ending
   with the top-level `OFFSETS` entries `[N_old + 1, N_old + K]` —
   so that at every moment the already-committed rows `[0, N_old)`
   remain fully described.
4. **Update the search indexes** the producer chooses to maintain. For
   each such index: first write its `SOURCE_GENERATION = g_old + 1` and
   `SOURCE_NROWS = N_old + K` — the tokens-before-content rule of
   {numref}`§%s <validity-tokens>` — then rebuild or update its content
   so that, after step 6 commits, it correctly describes rows
   `[0, N_old + K)`. For index families that support efficient
   incremental updates (`CHUNK_MINMAX`, `CHUNK_BLOOM`), a producer MAY
   simply append new entries; for families that do not (`SORTED_ROWS`,
   `BITMAP`), the producer typically rebuilds the index from scratch.
   Indexes the producer does not maintain MAY be left untouched — the
   validity check disables them — or deleted together with the
   corresponding references in each column's `SEARCH_INDEX_LIST`.
5. **Write `GENERATION = g_old + 1`** (when the table carries
   `GENERATION`).
6. **Commit by writing `NROWS = N_old + K`** as the final step. The
   attribute update SHOULD be followed by an `H5Fflush` (or equivalent)
   before the producer reports the append as complete.

The ordering is critical for crash recovery. Until step 6 commits,
every consumer that opens the file still sees `NROWS = N_old` and
therefore the table exactly as it was before the append. A producer
that crashes anywhere in steps 1–4 leaves a file whose observable
state is identical to the pre-append state: the unused tail in
`[N_old, extent)` is reserved storage, any index residue beyond
`N_old` is ignored, and indexes rewritten in step 4 fail the validity
check. A crash after step 5 but before step 6 additionally disables
all of the table's search indexes until a producer refreshes or
rebuilds them ({numref}`§%s <validity-tokens>`). The table data remains
fully readable. No cleanup is required for readers to use the
file correctly. A subsequent producer that wants to retry the append
SHOULD either overwrite the unused tail or extend further.

As with every crash-consistency statement in this document, the above
describes the HEP001-level state of a file that survives the crash
structurally intact ({numref}`§%s <validity-tokens>`). HDF5 does not
journal its own metadata: a crash during any metadata write — a dataset
extension, an attribute update, a B-tree modification — MAY corrupt the
HDF5 file itself, beyond what this specification's ordering rules can
protect. Producers that require durability against file-level
corruption MUST arrange it outside HDF5 (for example, journaled or
copy-on-write storage, filesystem snapshots, or write-ahead copies of
the file).

HDF5 itself does not guarantee atomic ordering between the data writes
of step 3, the index writes of step 4, and the attribute updates of
steps 5–6 unless the producer issues explicit `H5Fflush` calls between
them. A producer that requires strong durability
across a crash MUST issue an `H5Fflush` after step 4 and again after
step 6, so that the on-disk state cannot show `NROWS = N_old + K`
together with unwritten column data or stale indexes.

```{note}
The commit point of this workflow is a modification of the table group's
`NROWS` *attribute*. The original HDF5 SWMR (Single-Writer/Multiple-Reader)
feature cannot be used to publish an append this way: an SWMR writer may
only append raw data to existing datasets along an unlimited dimension and
MUST NOT create or modify attributes — or any other metadata — while SWMR
writing is active. Producers that need live concurrent readers therefore
cannot rely on classic SWMR for the `NROWS` commit. The prototype VFD SWMR
feature does permit attribute creation and modification while writing,
but as of this revision it has not shipped in any stable HDF5 release —
it exists on a development feature branch only. Until it ships, live
concurrent reading of a table under active append requires coordination
outside HDF5.
```

### Preallocation

A producer MAY extend column datasets past the current `NROWS` to
amortize the cost of `H5Dset_extent` across many small appends — for
example, extending by one chunk's worth of rows at a time and filling
that chunk over several append batches. The extended tail is reserved
storage; its contents have no semantic meaning under HEP001 until a
subsequent commit (step 6 above) increases `NROWS` to cover them.

A producer that preallocates SHOULD ensure that all columns in the
same table group remain at equal first-dimension extents after each
operation (see {numref}`§%s <hep001-consistency>`). The simplest and
recommended discipline is to preallocate every column by the same
number of rows at the same time.

(hep001-truncation)=
### Truncation

A producer MAY shrink the logical table by writing a smaller `NROWS`
value. The truncation is logical: the column datasets MAY retain their
old extents, with rows `[new_NROWS, old_NROWS)` becoming reserved
storage. A producer that wants to reclaim physical space MUST rewrite
each column dataset to its new extent.

The same logical truncation applies to list columns with no additional
writes: the smaller `NROWS` shrinks the valid top-level `OFFSETS` range
to `[0, new_NROWS]`, and `OFFSETS[new_NROWS]` becomes the bound on
valid elements at every nested level
({numref}`§%s <list-level-length>`). Positions and elements beyond
those bounds become reserved storage. Reclaiming their physical space
likewise requires rewriting the affected member datasets to their new
logical sizes.

The same index rules that govern appends apply on truncation, with
`new_NROWS` in place of `N_old + K`: the producer either maintains an
index (tokens first, then content; steps 4–6 of
{numref}`§%s <appending-rows>`), deletes it (removing the corresponding
`SEARCH_INDEX_LIST` entry on the column), or leaves it untouched to be
disabled by the validity check ({numref}`§%s <validity-tokens>`).

(in-place-updates)=
### In-place updates

A producer MAY rewrite individual cells of the table — change the
value at one or more row positions `< NROWS` — without altering
`NROWS`. Unlike appended rows, in-place writes are visible to consumers
the moment they reach the file, so the mutation MUST follow the
invalidate-first ordering of {numref}`§%s <validity-tokens>`:

1. **Write `GENERATION = g_old + 1`** and flush (`H5Fflush`). From this point,
   every search index in the table fails the validity check. (A table with no
   search indexes carries no `GENERATION` and skips steps 1 and 3.)
2. **Write the new cell values.**
3. **Restore search indexes.** For each index over a modified column: rebuild or
   update its content first, then write its `SOURCE_GENERATION = g_old + 1`
   (`SOURCE_NROWS` is unchanged) content before tokens, because the new
   generation is already committed. For each index over an unmodified column,
   rewriting `SOURCE_GENERATION` alone suffices. Indexes the producer does not
   restore MAY be left disabled or deleted. Flush.

A crash before step 2 leaves the data unchanged and every index
disabled but factually correct — a subsequent producer MAY restore them
by rewriting tokens alone. A crash during step 2 leaves possibly
partial data with every index disabled. In-place updates of the data
itself do not benefit from the single-attribute commit point that
`NROWS` provides; producers that require atomic semantics for
in-place edits MUST arrange them externally (for example, by writing
to a fresh column dataset and swapping it in under a future revision
of this HEP, or by using application-level coordination outside HDF5).

(hep001-consistency)=
## Consistency requirements

A conformant table group satisfies all of the following at all times:

1. The table group MUST carry a scalar `NROWS` attribute of datatype
   `uint64` (see {numref}`§%s <hep001-nrows>`).
2. Every column dataset (including row index columns) in the same table
   group MUST have the same first-dimension extent, and that extent
   MUST be `≥ NROWS`.
3. Every search-index dataset in the `SEARCH_INDEXES` group MUST carry a `KIND`
   attribute. The only other datasets permitted in the `SEARCH_INDEXES` group
   are the accompanying datasets those search indexes require, and an
   accompanying dataset MUST NOT carry a `KIND` attribute.
4. Every reference in a column's `SEARCH_INDEX_LIST` MUST resolve to a
   search-index dataset under the table group's `SEARCH_INDEXES`
   subgroup.
5. Every categorical column's `CATEGORIES` reference MUST resolve to a
   categories dataset in the table group's `CATEGORIES` subgroup
   ({numref}`§%s <hep001-categoricals>`). The `CATEGORIES` subgroup, when
   present, MUST contain only categories datasets, and every categories
   dataset MUST be referenced by at least one categorical column's
   `CATEGORIES` attribute.
6. `column-order`, when present, MUST list every column of the
   table (column datasets and list columns) exactly once and MUST NOT
   list any object that is not a column (in particular, not a categories
   dataset, a search-index dataset, or a member of a list column group).
7. Every reference in the table group's `INDEX_COLUMNS` attribute
   (when present) MUST resolve to a column dataset that is a direct
   child of the table group, MUST NOT be a null reference, and MUST NOT
   resolve to a list column group; when
   `_index` is also present, it MUST equal the dataset name of the
   column referenced by `INDEX_COLUMNS[0]`.
8. Every categorical column's fill value (see {numref}`§%s <fill-vals>`)
   MUST NOT collide with a valid integer code in the linked categories
   dataset's index range, so that the canonical missing-value test
   ({numref}`§%s <fill-vals>`) unambiguously denotes "missing category"
   rather than "valid value at category index *fill*."
9. Every search-index dataset whose validity check
   ({numref}`§%s <validity-tokens>`) passes MUST correctly describe its
   source column for row positions in `[0, NROWS)`. Tail entries that
   describe positions `≥ NROWS` — for example, residue from
   preallocation or from a prior truncation — MAY be present and
   MUST be ignored by consumers. A search-index dataset whose validity
   check fails is exempt from this requirement; consumers treat it as
   absent.
10. Every list column group MUST satisfy the structural rules of
    {ref}`hep001-list-columns` at every level: `CLASS` and `KIND`
    attributes present; `OFFSETS[0] = 0` and monotonically
    non-decreasing over `[0, L]` with extent `≥ L + 1` for that level's
    entry count `L`; `MASK`, when present, of extent `≥ L`, with
    `OFFSETS[i+1] = OFFSETS[i]` for every entry whose `MASK[i]` is
    `FALSE`; and a `VALUES` member that is a rank-1 dataset, a
    conformant nested list column group, or a conformant
    `STRING_VALUES` group.
11. No variable-length datatype occurs anywhere below a list column
    group ({numref}`§%s <list-no-vlen>`).
12. When a table group contains one or more search-index datasets, the
    table group MUST carry a scalar `uint64` `GENERATION` attribute, and
    every search-index dataset MUST carry scalar `uint64`
    `SOURCE_GENERATION` and `SOURCE_NROWS` attributes
    ({numref}`§%s <validity-tokens>`).
13. Every boolean column ({numref}`§%s <hep001-booleans>`) carries the
    HEP001 boolean datatype, holds only the values `0` and `1` at row
    positions in `[0, NROWS)`, and declares no user-defined fill value.

A producer that mutates a table (appends rows, truncates, rewrites a
column, etc.) MUST follow {numref}`§%s <write-workflow>` — for each
affected search index, either updating it consistently, deleting it, or
leaving it to be disabled by the validity check.

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

3. Names that HEP001 deliberately borrows from other ecosystems, currently
   only from Anndata's DataFrame layout
   ({numref}`§%s <hep001-anndata-relationship>`), are exempt from rule 1 and
   MUST be written exactly as those ecosystems write them. They are listed in
   {ref}`hep001-shared-names`.

4. Several attributes that align with broader scientific HDF5 community
   practice are exempt from rule 1 and MUST be written in lowercase, so that
   generic metadata harvesters and existing tools can discover them on a
   HEP001 table without case-folding.

   The descriptive annotation attributes `units`, `units_vocabulary`, and
   `description` are lowercase and carry no contractual meaning: their
   presence, absence, or value does not change how a HEP001 consumer
   interprets the table or any of its objects. They are defined alongside
   the objects that may carry them ({ref}`hep001-table-group`,
   {ref}`hep001-columns`) and do not appear in the reserved-name catalog.
   Any future descriptive annotation attribute introduced by this HEP or
   a successor MUST follow the same lowercase convention.

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

`CATEGORIES`
: The reserved subgroup of a table group that holds every categories dataset
  for the table. See {ref}`hep001-categoricals`. The token `CATEGORIES` names
  both this group and the column attribute of the same name below; the two are
  unambiguous because one is an HDF5 link name and the other an attribute name,
  but producers should be aware of the overload.

`SEARCH_INDEXES`
: The reserved subgroup of a table group that holds every search-index dataset
  for the table. See {ref}`hep001-search-indexes`.

#### List column member names

The following dataset names are reserved inside list column groups and
`STRING_VALUES` groups ({ref}`hep001-list-columns`). A list column group
itself is named by the producer (its name is the column name); only its
members carry reserved names.

`OFFSETS`
: The offsets dataset of a list column or `STRING_VALUES` group.

`VALUES`
: The flattened-elements member of a list column group: a rank-1 leaf
  dataset, a nested list column group, or a `STRING_VALUES` group. Shares
  its token with the `VALUES` attribute of `BITMAP` search indexes below;
  the two are unambiguous because one is an HDF5 link name and the other
  an attribute name.

`MASK`
: The optional null-entry mask dataset of a list column or
  `STRING_VALUES` group.

`CHARS`
: The UTF-8 byte-buffer dataset of a `STRING_VALUES` group.

#### Table group attribute names

`CLASS`
: Identifies the role of a HEP001 group. On a table group its value is
  `COLUMN_TABLE` ({ref}`hep001-class`); on a list column group,
  `LIST_COLUMN`; on the string-elements group of a list column,
  `STRING_VALUES` ({ref}`hep001-list-columns`). The value strings are
  reserved tokens.

`VERSION`
: HEP001 revision the table conforms to.

`NROWS`
: Scalar `uint64`. Number of logical rows currently in the table. See
  {numref}`§%s <hep001-nrows>`.

`TITLE`
: Human-readable title of the table (optional).

`INDEX_COLUMNS`
: A 1-D array attribute of HDF5 object references, whose elements point to
  the column datasets that serve as row labels for the table, in hierarchical
  order from outermost to innermost level. See {ref}`hep001-indexes`.

`GENERATION`
: Scalar `uint64`. State identifier of the table's column data, used by
  the index validity check. See {numref}`§%s <validity-tokens>`.

#### Column dataset attribute names

`SEARCH_INDEX_LIST`
: Object references to the search-index datasets that accelerate queries on
  this column.

`CATEGORIES`
: Object reference to the categories dataset, in the table group's
  `CATEGORIES` subgroup, that backs a categorical column. Shares its token
  with the `CATEGORIES` group above.

`valid_min`, `valid_max` *(lowercase, by exception)*
: Inclusive lower and upper bounds of the column's logical value range.
  Each is a scalar attribute whose datatype matches the column's element
  datatype. See {numref}`§%s <fill-vals>`.

#### Search-index and categories dataset attribute names

`KIND`
: ASCII string that identifies the family of a search-index dataset, or
  the storage method of a list column group
  ({numref}`§%s <list-column-identification>`). The two value sets are
  disjoint and are interpreted according to the object that carries the
  attribute.

`VALUES`
: Object reference, on a `BITMAP` search-index dataset, to its accompanying
  values dataset. Shares its token with the `VALUES` member name of list
  column groups above.

`SOURCE_GENERATION`, `SOURCE_NROWS`
: Scalar `uint64` validity tokens carried by every search-index dataset,
  recording the table state the index content was built against. See
  {numref}`§%s <validity-tokens>`.

#### `KIND` attribute values

On search-index datasets:

* `CHUNK_MINMAX`
* `SORTED_ROWS`
* `BITMAP`
* `CHUNK_BLOOM`

See {ref}`hep001-search-indexes` for their meaning. Consumers MUST treat unknown
values as "ignore this search index".

On list column groups:

* `OFFSETS`

See {ref}`hep001-list-columns` for its meaning. A consumer that does not
implement a list column's `KIND` MUST NOT silently drop the column
({numref}`§%s <list-column-identification>`).

(hep001-shared-names)=
### Names shared with Anndata

The following attribute names and string values are borrowed from Anndata's
DataFrame layout ({numref}`§%s <hep001-anndata-relationship>`) and are written
in **lowercase**, exactly as Anndata writes them, so that a producer targeting
an Anndata converter does not have to case-fold. They are reserved for their
Anndata-defined meaning and MUST NOT be repurposed:

* attribute names — `column-order`, `_index`, `encoding-type`,
  `encoding-version`, `ordered`;
* string values — `"dataframe"` and `"categorical"` when written into an
  `encoding-type` attribute.

Of these, HEP001 itself uses `column-order`, `_index`, and `ordered`
({numref}`§%s <hep001-table-group>`, {numref}`§%s <hep001-boolean-attributes>`);
`encoding-type`, `encoding-version`, and the string values are optional
pass-through names that HEP001 does not interpret. A producer MAY omit any of
these names, but if it writes one at all it MUST use this exact form
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
  NROWS             = N                (uint64, scalar)
  TITLE             = "Sample run"     (UTF-8)
  column-order      = ["row_id", "ts", "energy", "label"]  (UTF-8, 1-D)
  INDEX_COLUMNS     = [ref(row_id)]    (1-D object references)
  _index            = "row_id"         (UTF-8; primary row-label name)

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
  CATEGORIES        = ref(CATEGORIES/label__CATEGORIES)

/my_table/CATEGORIES               (Group)

/my_table/CATEGORIES/label__CATEGORIES   (Dataset, vlen UTF-8, shape (3,))
  ordered           = false
```

### A table with list columns

Extending {numref}`§%s <min-example-table>` with two list columns:
`readings` (`list<float32>`, with null lists possible) and `tags`
(`list<utf8>`, unbounded string elements, no null lists):

```
/my_table
  column-order      = ["row_id", "ts", "energy", "label",
                       "readings", "tags"]
  …

/my_table/readings                 (Group)
  CLASS             = "LIST_COLUMN"    (ASCII, fixed length)
  KIND              = "OFFSETS"        (ASCII, fixed length)
  description       = "Per-event auxiliary sensor readings."

/my_table/readings/OFFSETS         (Dataset, uint64, shape (N+1,))
/my_table/readings/MASK            (Dataset, bool enum, shape (N,))
/my_table/readings/VALUES          (Dataset, float32, shape (M,))
  units             = "V"

/my_table/tags                     (Group)
  CLASS             = "LIST_COLUMN"
  KIND              = "OFFSETS"

/my_table/tags/OFFSETS             (Dataset, uint64, shape (N+1,))
/my_table/tags/VALUES              (Group)
  CLASS             = "STRING_VALUES"

/my_table/tags/VALUES/OFFSETS      (Dataset, uint64, shape (M2+1,))
/my_table/tags/VALUES/CHARS        (Dataset, uint8, shape (B,))
```

Row `i` of `readings` is `VALUES[OFFSETS[i] : OFFSETS[i+1]]`, unless
`MASK[i]` is `FALSE`, in which case the list is null. Row `i` of `tags`
resolves through two offsets levels: entry `j` of the row's slice is the
UTF-8 decoding of `CHARS[VALUES/OFFSETS[j] : VALUES/OFFSETS[j+1]]`.

### Adding a chunk min/max search index

Extending {numref}`§%s <min-example-table>` with a `CHUNK_MINMAX` index on `ts`:

```
/my_table
  GENERATION        = g                (uint64; index validity token)
  …

/my_table/ts
  SEARCH_INDEX_LIST = [ref(SEARCH_INDEXES/ts__chunk_minmax)]
  …

/my_table/SEARCH_INDEXES                       (Group)
/my_table/SEARCH_INDEXES/ts__chunk_minmax      (Dataset,
    compound {min: int64, max: int64, nan_count: uint64,
              fill_count: uint64, n: uint64}, shape (n_chunks,))
  KIND              = "CHUNK_MINMAX"
  SOURCE_GENERATION = g                (uint64; matches GENERATION)
  SOURCE_NROWS      = N                (uint64; matches NROWS)
```

### A complete layout

```{mermaid}
graph TD
  TG[["/my_table<br/>CLASS=COLUMN_TABLE<br/>VERSION=1.0<br/>NROWS=N<br/>INDEX_COLUMNS=[ref(row_id)]"]]
  TG --> ri(["row_id (uint64, row index)"])
  TG --> ts(["ts (int64)"])
  TG --> en(["energy (float32)"])
  TG --> lb(["label (int8, categorical)"])
  TG --> CG[["CATEGORIES"]]
  TG --> SI[["SEARCH_INDEXES"]]
  CG --> lc>"label__CATEGORIES (vlen UTF-8)"]
  SI --> mm>"ts__chunk_minmax<br/>KIND=CHUNK_MINMAX"]
  SI --> sr>"energy__sorted_rows<br/>KIND=SORTED_ROWS"]
  SI --> bm>"label__bitmap<br/>KIND=BITMAP"]
  SI --> bv>"label__bitmap__VALUES"]
  SI --> bf>"ts__chunk_bloom<br/>KIND=CHUNK_BLOOM"]
  lb -.->|"CATEGORIES"| lc
  ts -.->|"SEARCH_INDEX_LIST"| mm
  ts -.->|"SEARCH_INDEX_LIST"| bf
  en -.->|"SEARCH_INDEX_LIST"| sr
  lb -.->|"SEARCH_INDEX_LIST"| bm
  bm -.->|"VALUES"| bv

  classDef tableGroup fill:#FFF4D4,stroke:#D4B86A,stroke-width:2px,color:#000
  classDef siGroup    fill:#E5DAF5,stroke:#9D85C5,stroke-width:2px,color:#000
  classDef catGroup   fill:#F5E2D0,stroke:#C59D85,stroke-width:2px,color:#000
  classDef dataCol    fill:#D4EBF8,stroke:#7FA9D0,stroke-width:1px,color:#000
  classDef indexCol   fill:#FAD4E1,stroke:#D08FB0,stroke-width:1px,color:#000
  classDef catData    fill:#FAE0D4,stroke:#D09F90,stroke-width:1px,color:#000
  classDef siData     fill:#D7F0D8,stroke:#7CB78A,stroke-width:1px,color:#000
  classDef siValues   fill:#E8F3E0,stroke:#A8C088,stroke-width:1px,color:#000

  class TG tableGroup
  class SI siGroup
  class CG catGroup
  class ri indexCol
  class ts,en,lb dataCol
  class lc catData
  class mm,sr,bm,bf siData
  class bv siValues
```

(hep001-anndata-relationship)=
## Relationship to Anndata

Anndata is the most direct inspiration for HEP001: HEP001 adopts the same
group-of-one-dataset-per-column shape and deliberately reuses several of
Anndata's attribute names (`column-order`, `_index`, `ordered`) so that
converters between the two are straightforward. HEP001 is **not**, however, a
drop-in Anndata DataFrame format, and this document does not specify
Anndata read/write conformance. The two layouts diverge on points that
matter for typical tabular data — Anndata encodes categoricals and nullable
columns as *subgroups* (with `codes`/`categories` or `values`/`mask`
children) and tags every column array with an `encoding-type`, whereas
HEP001 keeps each column as a single rank-1 dataset and records missing
values through fill values. Because an HDF5 link resolves to one object,
a column that Anndata stores as a subgroup cannot simultaneously be a HEP001
column dataset. A single group can therefore satisfy both specifications
only for the restricted case of dense numeric and variable-length string
columns with no categorical or nullable columns; anything richer requires an
explicit import/export step. HEP001 reserves the Anndata-derived attribute
names ({numref}`§%s <hep001-shared-names>`) so producers may write them when
targeting such a converter, but assigns them no HEP001 meaning.

(hep001-security)=
## Security considerations

Search indexes are unsigned, untrusted derivative data. A consumer that trusts a
table's column data MUST NOT, by default, trust the correctness of a search
index found in the same file: a tampered `CHUNK_MINMAX` can cause the consumer
to skip chunks that do in fact satisfy a predicate. The validity tokens of
{numref}`§%s <validity-tokens>` detect accidental staleness only and add nothing
against tampering — an adversarial writer can trivially forge matching tokens.
Consumers SHOULD offer a mode that verifies a search index against the column it
covers, or that ignores search indexes entirely. Producers SHOULD document the
provenance of search indexes in the table group's `description` or in each
search-index dataset's `description` attribute when that matters to their users.


## References

* HDF5 file format specification — The HDF Group.
  <https://docs.hdfgroup.org/hdf5/develop/_f_m_t3.html>
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
* RFC 2119 — Key words for use in RFCs to Indicate Requirement Levels. S. Bradner, 1997.
  <https://datatracker.ietf.org/doc/html/rfc2119>
* RFC 8174 — Ambiguity of Uppercase vs Lowercase in RFC 2119 Key Words. B. Leiba, 2017.
  <https://datatracker.ietf.org/doc/html/rfc8174>
* IEEE Std 754-2019 — IEEE Standard for Floating-Point Arithmetic. IEEE, 2019.
  DOI: 10.1109/IEEESTD.2019.8766229
* Semantic Versioning 2.0.0 — T. Preston-Werner.
  <https://semver.org/spec/v2.0.0.html>
* Unicode Standard Annex #15 — Unicode Normalization Forms. The Unicode Consortium.
  <https://www.unicode.org/reports/tr15/>
* MurmurHash3 — A. Appleby, SMHasher project.
  <https://github.com/aappleby/smhasher/wiki/MurmurHash3>
* A. Kirsch and M. Mitzenmacher, "Less Hashing, Same Performance: Building a Better Bloom Filter,"
  in *Algorithms — ESA 2006* (LNCS 4168), pp. 456–467, 2006.
  DOI: 10.1007/11841036_42
