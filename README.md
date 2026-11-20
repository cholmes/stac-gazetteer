# stac-gazetteer

Two [STAC](https://github.com/radiantearth/stac-spec) extensions, built in the
shape of [the stac-extensions template](https://github.com/stac-extensions/template),
that let a STAC Collection declare itself a **cloud-native gazetteer**:
published static data — a sorted names-index parquet, optionally beside
per-level boundary parquets — that lets a browser do place-name search over
plain HTTP range requests, with no server and no build step.

**Status: draft for discussion**, written up from
[cholmes/cng-map-widgets#1](https://github.com/cholmes/cng-map-widgets/issues/1),
which is the authoritative design discussion and field-table source. Nothing
here is registered with the official
[`stac-extensions`](https://github.com/stac-extensions) GitHub org yet; the
schema URLs below (`cholmes.github.io/stac-gazetteer/...`) are real, stable
URLs this repo can be published to, not `example.org` placeholders.

## The two-extension split

**[`gazetteer`](gazetteer/v1.0.0/README.md)** — lets a client search names and
get back a bbox. One sorted names-index parquet, its column contract, and the
normalization/sort guarantee that makes a binary search over it valid.
Collection-level: it says nothing about Items.

**[`gazetteer-boundaries`](gazetteer-boundaries/v1.0.0/README.md)** — upgrades
that bbox to a polygon. Per-level, per-resolution-variant boundary assets,
their join columns back to the names index, and the `geometry_ref` fast path.
Depends on `gazetteer`.

They are split because `gazetteer` alone is already a shippable product: a
search box that resolves "Liechtenstein" and flies the map to its bbox, with
zero geometry fetched. A publisher who already has boundary files can adopt
`gazetteer` by adding one small sorted parquet, with no change to those files
and no pre-simplified variant required — the thing that makes
`gazetteer-boundaries` real work. Folding the two together would make that
simplification work a precondition for any of it, and the realistic adoption
answer becomes "later." Keeping them separate means "later" only has to apply
to the second extension.

## Schema URLs (this version)

```
https://cholmes.github.io/stac-gazetteer/gazetteer/v1.0.0/schema.json
https://cholmes.github.io/stac-gazetteer/gazetteer-boundaries/v1.0.0/schema.json
```

## Layout

```
gazetteer/v1.0.0/schema.json              # JSON Schema (draft-07)
gazetteer/v1.0.0/README.md                # field table, column contract, "what this cannot check"
gazetteer-boundaries/v1.0.0/schema.json
gazetteer-boundaries/v1.0.0/README.md
examples/geoboundaries-cgaz-collection.json      # both extensions, every value real
examples/fieldmaps-index-only-collection.json    # gazetteer alone, illustrative
CHANGELOG.md
```

## Examples

### [`examples/geoboundaries-cgaz-collection.json`](examples/geoboundaries-cgaz-collection.json)

Both extensions together. Every value is real — transformed from the live
manifest published at
`data.source.coop/portolan-mirrors/geoboundaries-catalog/names/index.json`,
with `table:columns` read from the actual parquet schema. Worth looking at for:

- `gazetteer:feature_id_column: null` on the ADM0 boundary assets — legal and
  meaningful, not an omission: CGAZ ADM0 has no id column at all, so
  resolution joins on `gazetteer:index_key_column: "country"` against the
  boundary asset's `shapeGroup` column instead.
- `attribution[]` and its `share_alike` casing hazard collapse into plain STAC
  `license` (an SPDX id) plus `providers[]` — the ecosystem's own fields,
  not reinvented ones.

### [`examples/fieldmaps-index-only-collection.json`](examples/fieldmaps-index-only-collection.json)

**`gazetteer` alone — illustrative, not a published catalog.** Shows what
adoption looks like for a catalog that already has boundary files: every
index column maps directly onto a column that catalog already publishes, so
building the index asset is close to free. Note what is deliberately *not*
tagged — the existing ADM2 polygon file keeps `"roles": ["data"]` rather than
`"gazetteer-boundaries"`, because tagging it would pull in the second
extension and a join column it doesn't have. Leaving it untagged is the whole
argument for splitting the two extensions.

## What a JSON Schema cannot check

Said once here, and said again — more fully — in the `gazetteer` extension's
own README, because it matters enough to repeat: the requirement that
actually makes this format work is that an index asset's rows ascend by its
sort column, compared **byte-wise on UTF-8 encoding**, produced by one
specific normalization. No JSON Schema expresses that, and no STAC validator
will check it. **Schema conformance is necessary but not sufficient** — a
catalog that validates against both schemas here can still be unusable to
every client. A conformance checker that actually reads the parquet and
verifies the sort is part of the proposal in
[issue #1](https://github.com/cholmes/cng-map-widgets/issues/1), not an
optional extra, and is not part of this repository yet.

## Verifying this repository

Every `.json` file in this repository must parse as JSON and validate against
the oneOf branch its `type` selects. There is no `npm test` wired up yet (the
template's usual CI); until there is, validate by hand:

```bash
for f in $(find . -name '*.json'); do
  python3 -c "import json; json.load(open('$f'))" && echo "ok $f" || echo "BAD $f"
done
```

## Contributing

All contributions are subject to the
[STAC Specification Code of Conduct](https://github.com/radiantearth/stac-spec/blob/master/CODE_OF_CONDUCT.md).
