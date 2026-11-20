# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
per extension (each extension version lives at its own `vX.Y.Z/` path, per
[stac-extensions](https://github.com/stac-extensions/template) convention).

## [Unreleased]

### Added

- `gazetteer` extension, v1.0.0: Collection-level `gazetteer:levels`,
  `gazetteer:normalization`, `gazetteer:sort`, `gazetteer:stats_truncated`,
  `gazetteer:dropped_unnamed`; asset-level `gazetteer:row_groups`,
  `gazetteer:feature_id_column`, `gazetteer:footer_metadata_bytes`,
  `gazetteer:row_group_rows` on the required `gazetteer-index` asset.
  Forbids `gazetteer:*` in Item properties.
- `gazetteer-boundaries` extension, v1.0.0: asset-level `gazetteer:level`,
  `gazetteer:variant`, `gazetteer:geometry_column` (required on any asset with
  role `gazetteer-boundaries`), plus optional `gazetteer:feature_id_column`
  (string or explicit null), `gazetteer:index_key_column`,
  `gazetteer:country_column`, `gazetteer:row_group_rows`,
  `gazetteer:row_groups`, `gazetteer:footer_metadata_bytes`.
- Worked examples: `geoboundaries-cgaz-collection.json` (both extensions, real
  published values) and `fieldmaps-index-only-collection.json` (`gazetteer`
  alone, illustrative).

Written up from
[cholmes/cng-map-widgets#1](https://github.com/cholmes/cng-map-widgets/issues/1).
