# AGENTS.md

Guidance for LLM and code agents working in this repository.

## Repository

- Name: `geocoder_offline`
- Purpose: offline US address geocoding, for settings where addresses cannot
  leave the local network
- Read [`ARCHITECTURE.md`](ARCHITECTURE.md) before making any structural change

## Status

Early. The package skeleton reflects the target architecture; most modules are
not yet implemented. `ARCHITECTURE.md` describes the destination, not the
current state. Do not assume a module exists because the tree in that document
lists it.

## Tech stack

- Python 3.12+
- `uv` for environment and package management
- `polars` for tabular data
- `click` for CLI
- `FastAPI` for HTTP
- `pytest` for tests

## Conventions

- `from __future__ import annotations` at the top of every module
- Type hints throughout. Dataclasses for records, `Protocol` for interfaces
- Module docstrings state what the module does and where the design came from
- One function per package. If code does not fit the seven functions in
  `ARCHITECTURE.md`, that is a signal the architecture needs discussion, not
  that the code needs a new top-level package

## Hard rules

**No real addresses in this repository.** Test fixtures use synthetic addresses
or public landmarks. This repository is public, and address data may be
protected health information in the settings this library targets.

**No deployment-specific configuration.** Institutional address suppression
lists, hostnames, credentials, and site-specific settings are external
configuration. They are not committed here.

**Metaphone output must be byte-identical to DeGAUSS.** The DeGAUSS SQLite
database stores precomputed `street_phone` and `city_phone` values. A phonetic
implementation that disagrees with them silently breaks every lookup against a
prebuilt database. Verify against a real `geocoder.db` before building anything
on top of `metaphones/`.

**`precision` is an enum, not a number.** Scores are comparable only within a
precision level. Do not threshold on score alone.

**Batch is the primary path.** A backend that forks a process or opens a
connection per address does not scale. Implement `geocode_batch` first, and
make `geocode` the special case.

## Upstreams

Design is borrowed from `degauss-org/geocoder`, `geocommons/geocoder`, and
`postgis/postgis`. Those are references, not dependencies to patch. Do not open
pull requests against them from work here.

## Testing

```bash
uv venv
source .venv/bin/activate
uv pip install -e ".[dev]"
pytest
```
