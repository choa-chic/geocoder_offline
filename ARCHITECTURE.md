# Architecture

Version: 1.1.0
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
| 1 | Acquisition | `geocoder/acquire/` | Discover and download TIGER/Line shapefiles from the Census Bureau, resumably |
| 2 | Load | `geocoder/load/` | Unzip, read shapefiles, and load into SQLite, DuckDB, or PostGIS |
| 3 | Standardization | `geocoder/addr_parsing/`, `geocoder/metaphones/` | Parse a raw address string into components; compute phonetic keys |
| 4 | Matching | `geocoder/match/` | Find candidate street segments, score them, interpolate a position |
| 5 | Geography | `geocoder/geography/` | Convert lat/lon to FIPS codes and census tract or block |
| 6 | Service | `geocoder/service/` | CLI, HTTP API, and batch pipeline |
| 7 | Evaluation | external | Comparison against reference geocoders. Deliberately a separate project |

Evaluation lives outside this library on purpose. A test harness that ships
with the thing it tests cannot credibly disagree with it.

## Layout

```
geocoder/
  acquire/
    discover.py         enumerate available TIGER files by scraping census.gov
    downloader.py       fetch with retry
    progress.py         resumable download state
    url_patterns.py     state and territory FIPS, dataset types, URL construction

  load/
    schema/degauss.py   the place/edge/feature/feature_edge/range schema
    backends/           sqlite.py, duckdb.py, postgis.py
    unzip.py
    shapefile.py        typed shapefile record reading

  addr_parsing/
    types.py            ParsedAddress. The canonical component model
    tables.py           USPS suffixes, directionals, state abbreviations
    degauss_like/       parse rules following the DeGAUSS/Geocoder::US lineage
    postgis_like/       table-driven parse following PostGIS standardize_address()
    usaddress_like/     usaddress + usaddress-scourgify

  metaphones/           phonetic keys, byte-compatible with DeGAUSS street_phone

  match/
    types.py            GeocodeResult, Precision, ResultStatus
    interpolate.py      position along an address range, side-of-street offset
    score.py            candidate scoring
    triage.py           PO box, institutional, and non-address classification
    backends/           native.py, degauss.py, postgis.py

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

## The two interfaces

Everything else is implementation detail behind these.

```python
class AddressParser(Protocol):
    def parse(self, raw: str) -> ParsedAddress: ...

class GeocodeBackend(Protocol):
    def geocode(self, addr: ParsedAddress) -> GeocodeResult: ...
    def geocode_batch(self, addrs: Iterable[ParsedAddress]) -> Iterator[GeocodeResult]: ...
```

`geocode_batch` is not a convenience wrapper around `geocode`. Batch is the
primary path. Backends that fork a process or open a connection per address do
not scale to millions of addresses, and that constraint drives the whole design.

Multiple parser and backend implementations coexist by design. Address parsing
has several defensible answers, and which one is best is an empirical question
that depends on the address corpus. The protocol lets them be compared rather
than argued about.

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
    backend:        str
    tiger_vintage:  int
```

Three things about this are load-bearing:

**`precision` is an enum, not a number.** A score of 0.8 at `range` precision
and a score of 0.8 at `city` precision are not comparable claims. Scores are
only meaningful within a precision level. Thresholding on score alone is unsafe.

**Address triage happens before geocoding, not after.** PO boxes, known
institutional addresses, and non-address text are classified and excluded up
front. `geocode_result` records why, so a non-geocoded row is distinguishable
from a failed one.

**`backend` and `tiger_vintage` are on every row.** A cached geocode that
cannot be attributed to an engine version and a data vintage cannot be
selectively invalidated, which forces a full re-geocode every time either
changes.

## Separation of geocoding from tract assignment

Geocoding (address to lat/lon) and geography (lat/lon to tract) are separate
functions with separate versions. They are deliberately not fused.

A new census vintage changes tract boundaries but not street geometry. Keeping
the two steps separate means a new vintage requires only re-running a spatial
join over already-cached coordinates, rather than re-geocoding every address.
For a corpus in the millions, that is the difference between minutes and days.

## Backends

| Backend | Engine | Status |
|---|---|---|
| `degauss` | The DeGAUSS SQLite database, read in process | Planned |
| `postgis` | PostGIS `geocode()` against a loaded TIGER database | Planned |
| `native` | Pure Python range interpolation over SQLite | Planned |

DeGAUSS is a supported backend, not a thing being replaced. It works, it is
validated, and it stays. The native backend is measured against it.

## Lineage

This library consolidates work spread across several repositories and borrows
design from three upstreams, none of which are modified here:

- [`degauss-org/geocoder`](https://github.com/degauss-org/geocoder): the output
  contract, precision levels, score thresholding, and address triage
- [`geocommons/geocoder`](https://github.com/geocommons/geocoder): the original
  Ruby `Geocoder::US`. Parse tables, candidate ranking, metaphone
- [`postgis/postgis`](https://github.com/postgis/postgis): table-driven
  `standardize_address()` and `geocode()` candidate ranking

## Parser selection

Three parser backends are planned, and the choice is evidence-based rather
than preferential. All four candidates were measured against a labeled
hard-case corpus:

| Backend | Role |
|---|---|
| `usaddress_like` | Default. Led on parse accuracy against a USPS-conventional reading |
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
