---
status: accepted
---

# Local state persisted as a single JSON document behind an injected store seam

The app keeps a small amount of local state — the user's **Saved Locations** and their
unit-system preference. We persist it as a single JSON document (`{ unitSystem, currentLocation, savedLocations[] }`)
under `FileSystem.AppDataDirectory`, read and written behind an injected store interface
(`IAppStateStore`), rather than via `Preferences`, SQLite, or `SecureStorage`.

## Considered Options

- **`Microsoft.Maui.Storage.Preferences`** — trivial for the unit toggle, but the Saved Locations
  list collapses into an opaque serialized blob under one key, with no inspectable schema and no
  real file to exercise in a test.
- **SQLite** — a relational store for a list capped at ~10 entries; machinery without payoff.
- **`SecureStorage`** — reserved by Technical-Context Principle 1 for credentials. Saved Locations
  and a unit preference are not secrets, so this would misuse the secure store.
- **A single JSON document behind `IAppStateStore`** (chosen) — one file, one explicit schema.

## Consequences

- The persisted store gains an explicit schema, which is exactly what Technical-Context Principle 3
  protects: changing its shape later is a breaking change that requires its own ADR + migration
  consideration.
- Filesystem access lives behind a DI-registered interface (Principle 2), keeping it out of Views
  and ViewModels and making it cross-platform via the `FileSystem` abstraction (Principle 4).
- The store becomes a load-bearing seam: it gets a Tier-1 real-IO test (round-trip serialize →
  write → read → deserialize against real local I/O).
