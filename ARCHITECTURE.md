# Architecture

Version: 1.2.0
Last Updated: 2026-09-19 09:47 EDT

Target architecture for this library. This document describes where the code is
going, not what is here today. Most of the tree below is not yet implemented.

## Purpose

Offline US address geocoding. Everything runs locally: no address ever leaves
the network. This is the binding constraint for use in settings where
addresses are protected health information.

## Design

Seven functions, each with exactly one home.

| # | Function | Package | What it does |
|---|---|---|---|
| 1 | Acquisition | `geocoder/acquire/` | Fetch reference data resumably: TIGER/Line from the Census Bureau, address points from the National Address Database, county parcel data where available |
| 2 | Load | `geocoder/load/` | Unzip, read shapefiles, and load into SQLite, DuckDB, or PostGIS |
| 3 | Standardization | `geocoder/prestep/`, `geocoder/metaphones/` | A configurable pipeline: clean, triage, tag, harmonize, key |
| 4 | Matching | `geocoder/match/` | Staged ZIP/street/number matching against a reference, then scoring and placement |
| 5 | Geography | `geocoder/geography/` | Convert lat/lon to FIPS codes and census tract or block |
| 6 | Service | `geocoder/service/` | CLI, HTTP API, and batch pipeline |
| 7 | Evaluation | external | Comparison against reference geocoders. Deliberately a separate project |

Evaluation lives outside this library on purpose. A test harness that ships
with the thing it tests cannot credibly disagree with it.

## Layout

```
geocoder/
  acquire/
    tiger/              enumerate and fetch TIGER/Line, resumably
    nad/                National Address Database county partitions
    parcel/             county parcel data, per-county adapters
    manifest.py         sidecar: schema version, dataset year, SHA-256, counts

  load/
    schema/degauss.py   the place/edge/feature/feature_edge/range schema
    backends/           parquet.py (preferred), sqlite.py, duckdb.py, postgis.py
    unzip.py
    shapefile.py        typed shapefile record reading

  prestep/              the five-stage pipeline
    types.py            AddrNumber, AddrStreet, AddrPlace
    pipeline.py         PreStep protocol, PreStepPipeline, config loading
    clean.py            text normalization
    triage.py           PO box, institutional, and non-address classification
    tag/                usaddress_like.py, postgis_like.py, degauss_like.py
    harmonize.py        canonical USPS forms
    key.py              cache key and match keys

  metaphones/           phonetic keys, byte-compatible with DeGAUSS street_phone

  match/
    types.py            GeocodeResult, Precision, ResultStatus, Reference
    registry.py         backend registration by entry point
    staged.py           ZIP, then street, then number
    presets.py          strict | default | exact-zip | loose
    interpolate.py      position along an address range, side-of-street offset
    score.py            candidate scoring
    backends/           nad_point.py, parcel.py, degauss.py, postgis.py,
                        native_range.py, cascade.py

  geography/
    tracts.py           point-in-polygon against TIGER tract and block geometry
    fips.py             11 to 15 digit FIPS assembly

  service/
    cli.py
    api.py
    batch.py            cache-keyed bulk pipeline

  utils/
    logging.py
    file_structure.py
```

Reference storage prefers hive-partitioned Parquet, partitioned by ZIP with
county files, read lazily. That layout aligns storage with the first stage of
matching, so a lookup touches one partition rather than scanning. SQLite and
DuckDB remain for the DeGAUSS compatibility path.

## The pre-step is a pipeline

Address standardization is five stages, not one `parse()` call. Collapsing them
makes it impossible to express "tag with one implementation, canonicalize with
another," which is a combination worth having.

```python
class PreStep(Protocol):
    name: str
    def apply(self, batch: AddressBatch) -> AddressBatch: ...
```

| Stage | Does | Swappable for |
|---|---|---|
| `clean` | Whitespace, case, punctuation, encoding | A site-specific cleaner for known input quirks |
| `triage` | PO box, institutional, non-address classification | A local suppression list, loaded externally |
| `tag` | Raw text to typed components | `usaddress_like`, `postgis_like`, `degauss_like` |
| `harmonize` | Components to canonical USPS forms | Strict USPS, or PostGIS-parity forms |
| `key` | Cache key plus match keys | Phonetic algorithm choice |

Only `tag` differs between the parser implementations. The other four are
shared whatever tagger runs.

```python
pipeline = PreStepPipeline.from_config(config)
pipeline = pipeline.replace("tag", PostgisTagger())   # one stage, not the set
```

Stages are individually versioned, so the cache key records which pipeline
produced it and changing the tagger invalidates only what that tagger touched.
A stage may also short-circuit: `triage` marking a row as a PO box stops the
pipeline for that row, and the result carries the reason.

## The component model follows the federal standard

Components are typed against the *United States Thoroughfare, Landmark, and
Postal Address Data Standard* rather than an ad hoc record:

```python
AddrNumber(prefix, digits, suffix)
AddrStreet(predirectional, premodifier, pretype, name, posttype, postdirectional)
AddrPlace(name, state, zipcode)
```

`premodifier`, `pretype` and `postmodifier` are not decoration. An address like
`1 North Ave NW`, a real Atlanta street, cannot be represented correctly
without `pretype`, and parsers whose model lacks it get it wrong for exactly
that reason. Adopting a federal standard also means two systems can exchange
addresses without a translation table.

Every component is canonicalized to uppercase, so comparison is
case-insensitive by construction.

## Backends are registered, not imported

```python
class GeocodeBackend(Protocol):
    name: str
    capabilities: BackendCapabilities
    def geocode_batch(self, batch: AddressBatch) -> Iterator[GeocodeResult]: ...

@dataclass(frozen=True)
class BackendCapabilities:
    precisions: frozenset[Precision]   # what this backend can emit
    reference:  Reference              # tiger_ranges | nad_points | parcel
    batch:      bool
    needs_db:   bool
```

Registration is by entry point, so a deployment can add a backend without
forking the library:

```python
backend = get_backend("cascade", config)
```

`capabilities` lets a caller reason about a backend without running it. A
pipeline that requires `range` precision can reject a `zip`-only backend at
configuration time rather than discovering it mid-run.

`geocode_batch` is the primary path, not a convenience wrapper around
`geocode`. A backend that forks a process or opens a connection per address
does not scale to millions of addresses, and that constraint drives the design.

## Matching is staged

Within a backend: match ZIP codes first, then streets within each matched ZIP,
then address numbers within each ZIP/street group. This keeps matching fast and
lets tolerance be set per stage independently, which is what makes presets
possible.

| Preset | Behavior |
|---|---|
| `strict` | Exact street and ZIP, no fuzzy distance |
| `default` | Moderate fuzzy street distance, directionals respected |
| `exact-zip` | ZIP exact, street may vary |
| `loose` | Fuzzy distance 3, ignore street type and directionals. Recall over precision |

Precision versus recall is a per-use-case dial, not a library constant.

## The result type

The output contract follows DeGAUSS, extended with the fields a persistent
geocode cache requires.

```python
@dataclass(frozen=True)
class GeocodeResult:
    matched_street: str | None
    matched_city:   str | None
    matched_state:  str | None
    matched_zip:    str | None
    precision:      Precision      # range | street | intersection | zip | city
    score:          float
    lat:            float | None
    lon:            float | None
    geocode_result: ResultStatus   # geocoded | imprecise_geocode | po_box |
                                   # institutional | non_address_text
    census_tract:   str | None
    census_block:   str | None
    backend:        str            # which engine produced this
    reference:      Reference      # tiger_ranges | nad_points | parcel
    vintage:        int            # reference data year
    bundle_digest:  str            # SHA-256 of the reference bundle
```

Three things about this are load-bearing:

**`precision` is an enum, not a number.** A score of 0.8 at `range` precision
and a score of 0.8 at `city` precision are not comparable claims. Scores are
only meaningful within a precision level. Thresholding on score alone is unsafe.

**Address triage happens before geocoding, not after.** PO boxes, known
institutional addresses, and non-address text are classified and excluded up
front. `geocode_result` records why, so a non-geocoded row is distinguishable
from a failed one.

**`backend`, `reference`, `vintage` and `bundle_digest` are on every row.** A
cached geocode that cannot be attributed to an engine, a reference dataset and
a data vintage cannot be selectively invalidated, which forces a full
re-geocode every time any of them changes. `reference` carries additional
weight under a cascade backend: two rows from the same engine may have come
from a recorded address point and from an interpolated street range, and
pooling those in analysis hides the difference that matters most.

## Separation of geocoding from tract assignment

Geocoding (address to lat/lon) and geography (lat/lon to tract) are separate
functions with separate versions. They are deliberately not fused.

A new census vintage changes tract boundaries but not street geometry. Keeping
the two steps separate means a new vintage requires only re-running a spatial
join over already-cached coordinates, rather than re-geocoding every address.
For a corpus in the millions, that is the difference between minutes and days.

## Backends

| Backend | Reference data | Role | Status |
|---|---|---|---|
| `nad_point` | National Address Database points | **Primary** | Planned |
| `parcel` | County parcel data | Best available, county-scoped | Planned |
| `degauss` | TIGER ranges via `geocoder.db`, read in process | Fallback, compatibility path | Planned |
| `postgis` | TIGER ranges via PostGIS `geocode()` | Fallback, production-parity path | Planned |
| `native_range` | TIGER ranges, pure Python | Fallback | Planned |
| `cascade` | Composes the above | **Default** | Planned |

### Why address points come first

Street-range interpolation guesses a position along a street segment from an
address range. Address-point matching looks up a coordinate that a local
authority recorded. Published evaluation against authoritative county parcel
data found street-range matching agreeing 7.2% to 59.2% of the time against
65.1% to 76.1% for address points, and found the street-range error
concentrated in denser and more materially deprived neighborhoods rather than
spread evenly.

That last part is why this is a design decision and not a tuning preference.
Error that correlates with neighborhood characteristics is differential
misclassification, and it biases any downstream exposure analysis rather than
just blurring it.

Street-range interpolation stays, because point coverage is incomplete and
partial coverage is worse than a documented fallback. The `cascade` backend
tries the most accurate reference first and falls through on a miss, and every
result records which step produced it in `backend` and `reference`, so the two
are never silently pooled.

DeGAUSS remains a supported backend. It works, it is validated, and it is the
compatibility path. It is no longer the target.

## Lineage

This library consolidates work spread across several repositories and borrows
design from three upstreams, none of which are modified here:

- [`degauss-org/geocoder`](https://github.com/degauss-org/geocoder): the output
  contract, precision levels, score thresholding, and address triage
- [`geocommons/geocoder`](https://github.com/geocommons/geocoder): the original
  Ruby `Geocoder::US`. Parse tables, candidate ranking, metaphone
- [`postgis/postgis`](https://github.com/postgis/postgis): table-driven
  `standardize_address()` and `geocode()` candidate ranking
- [`geomarker-io/addr`](https://github.com/geomarker-io/addr) (MIT): the
  five-stage pre-step, the federal component standard, staged ZIP/street/number
  matching, match presets, ZIP-partitioned Parquet reference storage, and the
  sidecar manifest model for versioning a data bundle

## Tagger selection

Three tagger implementations are planned for the `tag` stage, and the choice is
evidence-based rather than preferential. All four candidates were measured
against a labeled hard-case corpus:

| Tagger | Role |
|---|---|
| `usaddress_like` | Default. Led on parse accuracy against a USPS-conventional reading. Independently, `geomarker-io/addr` also embeds usaddress |
| `postgis_like` | Production-parity path. Reproduces PostGIS `standardize_address()` behavior, including its quirks, for agreement with data geocoded by a PostGIS geocoder |
| `degauss_like` | Required by the DeGAUSS backend, whose matching engine is tuned to its own parser's output. Not a general-purpose parser |

Three things that measurement established and inspection would not have:

- **Parsing is not the throughput bottleneck.** The slowest candidate handles
  10M addresses in well under an hour. Choose on accuracy.
- **Well-formed addresses do not discriminate.** Agreement between parsers
  runs above 91% on clean input and falls sharply on hard cases. Any
  evaluation corpus made of tidy addresses will wrongly conclude the choice
  does not matter.
- **PostGIS's stock rules fold the pre-directional into `house_num`** and
  return nothing at all when there is no house number. Normalizing that away
  is the right call for a general parser and the wrong call for parity with
  existing geocoded data, which is why both forms exist.

Note the scope of this: tagging is one stage of five. A better tagger improves
component extraction, which is necessary but not sufficient. The reference data
a backend matches against, covered above, moves accuracy considerably more.

## Conventions

Python 3.12+, uv, `pyproject.toml`. polars for tabular data, click for CLI,
FastAPI for HTTP, pytest for tests. Type hints throughout with
`from __future__ import annotations`.

Pre-commit and CI gate every change, including two custom content checks that
exist because this repository is public. See `AGENTS.md`.

No real addresses in this repository. Test fixtures use synthetic addresses or
public landmarks. Deployment-specific configuration, including any
institutional address suppression lists, is external configuration and is never
committed here.
