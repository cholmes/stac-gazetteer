# Gazetteer Extension Specification

- **Title:** Gazetteer
- **Identifier:** <https://cholmes.github.io/stac-gazetteer/gazetteer/v1.0.0/schema.json>
- **Field Name Prefix:** gazetteer
- **Scope:** Collection, Catalog
- **Extension [Maturity Classification](https://github.com/radiantearth/stac-spec/tree/master/extensions/README.md#extension-maturity):** Proposal
- **Owner**: @cholmes

This document explains the Gazetteer Extension to the [SpatioTemporal Asset
Catalog](https://github.com/radiantearth/stac-spec) (STAC) specification.

A **gazetteer** is a names index: a sorted parquet file that lets a client do
place-name search ("liechtenstein" -> Liechtenstein, a bbox) over plain HTTP
range requests, with no server and no database. This extension is what
`table` is to tabular data: putting its URI in `stac_extensions[]` is the
declaration "this Collection is searchable by name."

This extension describes **search to a bounding box**, nothing more. Request a
polygon outline for a result instead of a bbox? That's the companion
[`gazetteer-boundaries`](../../gazetteer-boundaries/v1.0.0/README.md)
extension, which is additive on top of this one. A publisher who only wants
"search box that flies the map to a bbox" needs nothing from that second
extension at all.

This is a **Collection-level extension**. A client should never expect
`gazetteer:*` fields in Item properties, and this schema forbids them there.

- Examples:
  - [geoBoundaries CGAZ Collection](../../examples/geoboundaries-cgaz-collection.json):
    both extensions together, every value real
  - [FieldMaps index-only Collection](../../examples/fieldmaps-index-only-collection.json):
    `gazetteer` alone, no boundary assets tagged
- [JSON Schema](schema.json)
- [Changelog](../../CHANGELOG.md)

## What this schema cannot check

**Read this before treating a passing validator as proof a catalog works.**

The requirement that actually makes a gazetteer usable is that its index
asset's rows ascend by `name_norm` (or whatever column `gazetteer:sort`
names), compared **byte-wise on its UTF-8 encoding** — not locale collation —
and that `name_norm` was produced by exactly the normalization named in
`gazetteer:normalization`:

```
NFD -> strip combining marks (\p{M}) -> lowercase
    -> keep only [\p{L}\p{N} '-] -> collapse whitespace -> trim
```

**No JSON Schema can express "this column is sorted a particular way" or "this
column's values were produced by this specific function," and no STAC
validator will check it.** A Collection can satisfy every requirement in this
document — the right fields, the right types, a `gazetteer-index` asset — and
still be unusable, because the parquet behind the asset is not actually sorted
that way, or `name_norm` was produced by a different normalization than it
claims.

**Schema conformance here is necessary but not sufficient.** A client
implementing a binary search against `gazetteer:sort` is trusting the
publisher's claim about the sort, not something this extension verified. Treat
a passing validator as "the metadata is well-formed," not as "this gazetteer
works." A separate conformance checker — one that actually opens the parquet
file and verifies the declared sort order and normalization against the
stored bytes — is required to make that second, stronger claim, and is tracked
as follow-on work alongside this extension (see
[cholmes/cng-map-widgets#1](https://github.com/cholmes/cng-map-widgets/issues/1)).

## Fields

The fields in the table below can be used in these parts of STAC documents:

- [x] Catalogs
- [x] Collections
- [ ] Item Properties (incl. Summaries in Collections) — **explicitly forbidden**, see above
- [x] Assets (for Collections only, specifically the `gazetteer-index` asset)
- [ ] Links
- [ ] Bands

### Collection fields

| Field Name | Type | Description |
| --- | --- | --- |
| gazetteer:levels | \[integer] | **REQUIRED.** The admin levels present in the gazetteer, e.g. `[0, 1, 2]`. |
| gazetteer:normalization | string | **REQUIRED.** Identifier for the normalization algorithm that produced the sort column(s), e.g. `"nfd-strip-marks-lower/1"`. The most load-bearing field in this extension — see [What this schema cannot check](#what-this-schema-cannot-check). |
| gazetteer:sort | string \| \[string] | **REQUIRED.** The index column, or ordered list of columns, the index asset's rows ascend by. The spec mandates byte-wise UTF-8 ordering on the first column, never locale collation. |
| gazetteer:stats_truncated | boolean | Honesty field: whether published row-count/level statistics are known to be incomplete. |
| gazetteer:dropped_unnamed | integer | Honesty field: count of unnamed rows the publisher dropped from the index (e.g. geoBoundaries drops 37 unnamed ADM2 rows and says so). |

### Asset fields

Apply to the asset(s) with `"roles": ["gazetteer-index"]`. **At least one such
asset is required** on any Collection declaring this extension.

| Field Name | Type | Description |
| --- | --- | --- |
| gazetteer:row_groups | integer | **REQUIRED** on a `gazetteer-index` asset. Number of row groups in the index parquet, for range-read planning. |
| gazetteer:feature_id_column | string | Name of the index's stable per-row id column. Defaults to `feature_id` when absent (geoBoundaries publishes `shape_id`). |
| gazetteer:footer_metadata_bytes | integer | Size in bytes of the thrift footer alone (not footer + page index) — what a range-reading client should request. |
| gazetteer:row_group_rows | integer | Rows per row group, when uniform. Required for the `geometry_ref` fast path used by `gazetteer-boundaries`. |

An optional second index asset may carry `"roles": ["gazetteer-index",
"prominent"]`: a small, eager-loaded strict subset of the index, same schema
and sort, enabling a client to do a fast initial search wave before the full
(or range-read) index has loaded. This extension places no additional
requirements on it beyond what applies to any `gazetteer-index` asset above.

Other fields commonly present on a `gazetteer-index` asset — `table:columns`,
`table:row_count` (from the [table
extension](https://github.com/stac-extensions/table)), `file:size`,
`file:checksum` (from the [file
extension](https://github.com/stac-extensions/file)) — are reused from those
extensions rather than reinvented here. Declare them alongside
`gazetteer:row_groups`, not as substitutes for it.

## Index column contract

The fields above describe the asset. The table below describes the **columns
inside that parquet file** — not expressible in JSON Schema at all, since
schema.json only sees STAC JSON, never the parquet's own schema. Document
these via `table:columns` (from the table extension) on the asset, and treat
this list as the contract a conformance checker would verify.

| Column | Status | Notes |
| --- | --- | --- |
| `name` | **Required** | Display name, as published. |
| `name_norm` | **Required** | The sort key. See `gazetteer:normalization`. |
| `level` | **Required** | Admin level, matching one entry of `gazetteer:levels`. |
| `feature_id` | **Required** | Stable per-row id. Actual column name given by `gazetteer:feature_id_column` when it differs from `feature_id`. |
| `bbox` | **Required** | The result bounding box. |
| `country` | Recommended | ISO country code, for ranking and filtering. |
| `country_name` | Recommended | Country display name, for dropdown context. |
| `parent_name` | Recommended | Immediate parent's display name, for dropdown context. |
| `area_km2` | Recommended | Used to rank larger places first. |
| `source` | Recommended | Per-row publisher/source attribution, when it varies by row. |
| `name_lang` | Optional | Language/script tag for `name`, when the catalog is multilingual. |
| `alt_of` | Optional | Marks a row as an alternate name of another row. |
| `class` | Optional | A publisher-defined feature class. |
| `population` | Optional | Used as a secondary ranking signal. |
| `parent_id` | Optional | Stable id of the immediate parent row. |

Requiring all of these to claim conformance is deliberately avoided: a client
can degrade gracefully on every recommended or optional column, and demoting
them is what makes adoption realistic.

## Relation types

This extension defines no `rel` types.

## Contributing

All contributions are subject to the
[STAC Specification Code of Conduct](https://github.com/radiantearth/stac-spec/blob/master/CODE_OF_CONDUCT.md).
For contributions, please follow the
[STAC specification contributing guide](https://github.com/radiantearth/stac-spec/blob/master/CONTRIBUTING.md).
