# Handoff — Weather POC: ready for `/roadmap`

**Branch:** `claude/grill-with-docs-au7571` (all work below is committed and pushed here)
**Next action:** run `/roadmap` to break `PRD.md` into an ordered Feature list → `Roadmap.md`

## Where things stand

The HITL design phase has defined the product. Everything needed is committed on the branch
above — a fresh session should read these rather than reconstruct anything:

- `PRD.md` — product requirements (problem, solution, 35 user-story requirements, module/seam
  breakdown, testing decisions, out-of-scope). **This is `/roadmap`'s primary input.**
- `business-domain-context.md` — domain glossary (10 terms). Use this vocabulary verbatim.
- `docs/adr/0001-local-state-as-json-document.md` — single JSON store behind `IAppStateStore`.
- `docs/adr/0002-location-backdrop-from-wikimedia.md` — key-less Wikimedia images for the backdrop.
- `Technical-Context.MD` — engineering contract (pre-existing; not authored this session).

Authority order (from `CLAUDE.md`): ADR > Technical-Context > business-domain-context > PRD >
Roadmap > Spec > Plan.

## What `/roadmap` should do

Break the PRD into Features with sequencing/dependencies (this is a multi-feature product).
Suggested ordering, smallest-coherent-slice first — validate against the PRD, don't take on faith:

1. **Search & resolve** — Search Query → Candidates → current Location (the `IGeocoder` seam).
2. **Weather view** — Current Conditions + 7-day Forecast, tabbed days, hourly strips
   (`IWeatherProvider`, Condition Map, the three async states).
3. **Saved Locations** — dropdown, save/unsave star, dedupe-by-coordinate, cap 10 (`IAppStateStore`).
4. **Units** — Metric/Imperial toggle, OS-inferred default, persisted (Unit Formatter).
5. **Location Backdrop** — faint Wikimedia image, graceful degradation (`ILocationImageProvider`).

Freshness Policy (staleness-gated foreground refetch, ~15 min) and launch-restore cut across the
weather view — decide whether they ride with Feature 2 or get called out separately.

## Context not in the artifacts (only thing the cold session can't read off disk)

- **Out-of-scope is deliberate, not forgotten** — geolocation, independent next-48h hourly strip,
  recents history, custom names for saved locations, timed auto-refresh, weather caching,
  UV/sunrise-sunset, keyed providers. All listed in `PRD.md` "Out of Scope"; carry them forward
  as explicit non-goals, don't let the Roadmap quietly re-introduce them.
- **Build-time follow-up (not now):** adopting Wikimedia adds a 2nd 3rd-party dependency; the
  factual *3rd-party tech* section of `Technical-Context.MD` needs updating at build time via
  `/sync-project-docs` (flagged in ADR-0002).

## After `/roadmap`

The "product is defined" milestone is complete once `Roadmap.md` exists. That's the natural point
to open **one PR** into `main` covering glossary + both ADRs + `PRD.md` + `Roadmap.md` as a single
reviewable unit. No PR has been opened yet — deliberately deferred through the HITL design phase.

## Kick-off prompt for the new session

> Continue the Weather POC on branch `claude/grill-with-docs-au7571`. Read
> `handoffs/2026-06-30-14 - roadmap-kickoff.md` first, then run `/roadmap` to break `PRD.md` into
> an ordered Feature list. The PRD, domain glossary, and ADRs are already committed on this branch.
