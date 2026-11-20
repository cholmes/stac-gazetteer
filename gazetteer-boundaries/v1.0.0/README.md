# Gazetteer Boundaries Extension Specification

- **Title:** Gazetteer Boundaries
- **Identifier:** <https://cholmes.github.io/stac-gazetteer/gazetteer-boundaries/v1.0.0/schema.json>
- **Field Name Prefix:** gazetteer
- **Scope:** Collection, Catalog, Item
- **Extension [Maturity Classification](https://github.com/radiantearth/stac-spec/tree/master/extensions/README.md#extension-maturity):** Proposal
- **Owner**: @cholmes

This document explains the Gazetteer Boundaries Extension to the
[SpatioTemporal Asset Catalog](https://github.com/radiantearth/stac-spec)
(STAC) specification.

**Depends on [`gazetteer`](../../gazetteer/v1.0.0/README.md).** Where
`gazetteer` lets a client search names and get back a bbox, this extension
upgrades that bbox to a full polygon: it declares the per-level boundary
assets, their join columns back to the names index, and the `geometry_ref`
fast path that avoids a second lookup when the index row already points at
the matching row group.

The two extensions are split deliberately. `gazetteer` alone is already a
shippable product — a search box that resolves a place and flies the map to
its bbox, with zero geometry fetched. A publisher who already has boundary
files can adopt `gazetteer` by adding one small sorted names parquet, with no
change to those files. Adopting this extension on top is what lets a client
draw the actual outline, and it is additive: a level with no boundary asset
tagged simply has no polygon available for it, degrading to the bbox from
`gazetteer` alone.

- Examples:
  - [geoBoundaries CGAZ Collection](../../examples/geoboundaries-cgaz-collection.json):
    every level has `full` and `display` boundary assets, every value real,
    including `gazetteer:feature_id_column: null` on ADM0
- [JSON Schema](schema.json)
- [Changelog](../../CHANGELOG.md)

## Fields

The fields in the table below can be used in these parts of STAC documents:

- [ ] Catalogs
- [x] Collections (specifically, assets on Collections)
- [ ] Item Properties (incl. Summaries in Collections)
- [x] Assets (for both Collections and Items)
- [ ] Links
- [ ] Bands

Apply to each asset with `"roles"` including `"gazetteer-boundaries"`. There
is one such asset per admin level × geometry variant (so a level with both a
full-resolution and a pre-simplified file has two boundary assets, not one
with nested variants).

| Field Name | Type | Description |
| --- | --- | --- |
| gazetteer:level | integer | **REQUIRED.** The admin level this asset's features belong to, e.g. `0`. |
| gazetteer:variant | string | **REQUIRED.** One of `full` or `display`. A level with only one resolution available simply omits the other asset — it does not fall back by convention. |
| gazetteer:geometry_column | string | **REQUIRED.** Name of this asset's geometry column. |
| gazetteer:feature_id_column | string \| null | The boundary file's own join column against the names index. **An explicit JSON `null` is legal and meaningful** and distinct from the field being absent: it means the file genuinely has no id column at all (geoBoundaries CGAZ ADM0 has no `shapeID` column), so resolution must join through `gazetteer:index_key_column` / `gazetteer:country_column` instead. |
| gazetteer:index_key_column | string | Which column of the *names index* (not this asset) is the join key, used together with `gazetteer:country_column` when `gazetteer:feature_id_column` is `null`. |
| gazetteer:country_column | string | Name of this asset's country column, used to narrow candidate row groups before a full scan. |
| gazetteer:row_group_rows | integer | Rows per row group, when uniform. Required for the `geometry_ref` fast path: a ref's `row_group` is checked against `floor(row / row_group_rows)` before trusting it. |
| gazetteer:row_groups | integer | Number of row groups in this asset, for range-read planning. |
| gazetteer:footer_metadata_bytes | integer | Size in bytes of the thrift footer alone — what a range-reading client should request. |

### Additional field information

#### gazetteer:feature_id_column

Three distinct states matter here, and a client that only checks truthiness
will get this wrong:

- **A string** — the boundary file has its own stable id column, use it directly.
- **An explicit `null`** — the publisher is telling you there is no such
  column, by design, not by omission. Fall back to
  `gazetteer:index_key_column` against `gazetteer:country_column`.
- **Absent entirely** — the publisher has not declared either way. A client
  should treat this the same as `null` (fall back, or decline the join) but
  cannot assume the column is genuinely missing with the same confidence a
  declared `null` gives.

This is why the schema's type for this field is `["string", "null"]` rather
than just `"string"`: a schema that only allowed strings would make the
meaningful, deliberate `null` case indistinguishable from a validation error.

### geometry_ref (cross-extension column)

This extension adds one optional column to the `gazetteer` index asset's
schema: `geometry_ref`, a struct (`file`, `row_group`, `row`) that points
directly at the matching row in a boundary asset, skipping a second lookup.
Because it lives on the *index* asset but only has meaning in the presence of
this extension, it is documented here rather than in `gazetteer`'s own column
table — this is the one deliberate cross-extension dependency between the two.

A client should still validate a `geometry_ref` against
`gazetteer:row_group_rows` before trusting it (see above), since a rebuilt
boundary file can shift row groups without the index being regenerated.

## Relation types

This extension defines no `rel` types.

## Contributing

All contributions are subject to the
[STAC Specification Code of Conduct](https://github.com/radiantearth/stac-spec/blob/master/CODE_OF_CONDUCT.md).
For contributions, please follow the
[STAC specification contributing guide](https://github.com/radiantearth/stac-spec/blob/master/CONTRIBUTING.md).
