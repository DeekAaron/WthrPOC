# Roadmap

**Product:** Weather POC — a Windows + macOS desktop app showing weather for one Location at a time.
**Last reviewed:** 2026-06-30

## Sequencing

Features are listed in delivery order. Each Feature gets its own `/brainstorming` session, Spec, and Plan.

---

## Feature 1: See the Current Conditions for a Location 🔫 *tracer bullet*

The app launches, fetches live weather for **one fixed Location** (a hard-coded coordinate) from Open-Meteo through `IWeatherProvider`, and renders its **Current Conditions** — temperature, feels-like, condition icon + label (via the Condition Map), wind, humidity, precipitation chance — bound from a ViewModel that walks the three explicit states (loading → result → error-with-retry). The `IWeatherProvider` call returns the full payload (Current Conditions + 7 Forecast Days + Hourly Readings) from the outset; this Feature renders only the Current Conditions slice of it.

**Out of scope:** searching for a place (the Location is hard-coded); the 7-day Forecast and day-tabs; the units toggle (defaults to Metric); Saved Locations; persistence / launch-restore; freshness / stale-refetch; the Location Backdrop.

**Dependencies:** None (this is the tracer bullet).

**Why first:** It is the thinnest vertical slice that exercises every layer of the stack — MAUI DI host → `IHttpClientFactory`-backed service seam → Open-Meteo → domain mapping → ViewModel lifecycle → XAML binding, on both Windows and macOS — and it establishes the patterns the rest of the project follows: the Tier-1 recorded-replay fixture for `IWeatherProvider` and the ViewModel ⇄ View binding test.

---

## Feature 2: Search for a place and pick the Location

The user types a **Search Query**, the app resolves it through `IGeocoder` (the Open-Meteo geocoding endpoint) into a list of **Candidates** (e.g. "Paris, FR" vs "Paris, TX"), and picking a Candidate immediately becomes the current **Location** — driving the Current Conditions view from Feature 1 instead of the hard-coded coordinate. Includes the "No places found" empty result and the first-launch "search for a Location" prompt.

**Out of scope:** the 7-day Forecast; the units toggle; Saved Locations; persistence / launch-restore (the chosen Location is not yet remembered across runs); freshness; the Location Backdrop.

**Dependencies:** Feature 1 (specifically: picking a Candidate sets the current Location, which feeds Feature 1's weather fetch and Current Conditions view through the same fetch lifecycle). Adds the second service seam, `IGeocoder`, with its own Tier-1 recorded-replay fixture.

---

## Feature 3: The 7-day Forecast — day-tabs with Daily Summaries and Hourly Readings

The current Location's weather is shown as the full **Forecast**: one tab per **Forecast Day**. Today's tab features the live **Current Conditions** (from Feature 1); every other day's tab features that day's **Daily Summary** (high/low, condition, precipitation chance). Each Forecast Day shows a scrollable hour-by-hour strip of **Hourly Readings** (time, temperature, condition icon, precipitation chance), and today's strip starts at the current hour. Switching tabs does **not** refetch — the single `IWeatherProvider` call already returned all 7 days with their Hourly Readings.

**Out of scope:** the units toggle; Saved Locations; persistence / launch-restore; freshness / stale-refetch; the Location Backdrop.

**Dependencies:** Feature 1 (the `IWeatherProvider` call already returns Current Conditions + 7 Forecast Days + Hourly Readings — this Feature surfaces the parts Feature 1 left unrendered, with no change to the seam contract) and Feature 2 (a real user-chosen Location to forecast).

---

## Feature 4: Metric / Imperial units, defaulted and remembered

A single toggle switches everything — temperature, feels-like, wind, precipitation, and the clock — between **Metric** and **Imperial** via the Unit Formatter. On first launch the default is derived from the OS region (US / LR / MM ⇒ Imperial, else Metric; Metric if it can't tell), and the choice is remembered between runs. The active unit system is passed to `IWeatherProvider` so data arrives in the right system.

**Out of scope:** Saved Locations and launch-restore of the current Location (the store exists but only holds `unitSystem` at this point); freshness; the Location Backdrop.

**Dependencies:** Feature 1 (the Current Conditions values being formatted) and Feature 3 (the Forecast / Hourly values too). Introduces the **`IAppStateStore`** seam (ADR-0001) — the single JSON document under `FileSystem.AppDataDirectory` — on its simplest possible payload (one enum), establishing the Tier-1 real-IO round-trip test (serialize → write → read → deserialize against a real temporary file). Feature 5 then extends this store.

---

## Feature 5: Saved Locations

The user saves the current Location with one deliberate action (a **star**), and Saved Locations appear in a **dropdown** for one-click reselection — picking one behaves exactly like a fresh search result (it just sets the current Location; no special mode to learn). Saving is **deduplicated by resolved coordinate** (re-saving is a no-op), the list is **capped at 10**, and toggling the star off unsaves. This Feature also adds **launch-restore**: the app reopens on the Location last viewed (the Location *identity* is restored from the store and then re-fetched fresh — weather is never cached to disk).

**Out of scope:** custom names for Saved Locations; automatic "recently viewed" history; freshness / stale-refetch; the Location Backdrop.

**Dependencies:** Feature 2 (a resolved Location to save, and picking a Saved Location reuses the same "set current Location → fetch" path) and Feature 4 (extends the `IAppStateStore` JSON document it introduced — adding `currentLocation` and `savedLocations[]` alongside `unitSystem`).

---

## Feature 6: Engagement-driven freshness — "Updated HH:MM" and stale-foreground refetch

The app shows when the data was last updated (**"Updated HH:MM"**) and quietly refreshes when the user returns to it after a while — but only if the data has gone stale (~15 min). The decision is gated by the **Freshness Policy** pure module — `(lastFetchedAt, now, window) → shouldRefetch?` — extracted so the rule is unit-testable with injected timestamps (boundary cases just-inside vs just-outside the window), no clock or UI required. Deliberately **no manual refresh button and no background polling**: a refetch happens only on Location-becomes-current (already built) and this staleness-gated foreground check.

**Out of scope:** the Location Backdrop. (The three-state error + retry path was established in Feature 1; this Feature is only the timestamp plus the staleness-gated foreground refetch.)

**Dependencies:** Feature 1 (the fetch lifecycle and the three-state ViewModel this hooks into) and Feature 5 (a restored / current Location whose freshness is judged on resume).

---

## Feature 7: Location Backdrop

A faint, representative image of the current Location (its "classic" image) sits behind the weather, sourced key-less from the **Wikimedia REST page-summary lead image** through a new `ILocationImageProvider` seam (ADR-0002). It stays subtle / low-opacity so the data stays readable, is the **same across all of a Location's day-tabs** (no flicker on tab switch), and shows any **required attribution** unobtrusively. When no confident image exists, the view simply shows the background colour — a miss is **silent and decorative, explicitly not a Principle-5 error** (which remains reserved for failed weather fetches).

**Out of scope:** the Openverse fallback image source (documented in ADR-0002 but deferred); keyed photo APIs (which would activate the dormant `SecureStorage` secrets path).

**Dependencies:** Feature 1 / Feature 2 (a current Location to fetch an image *of*) and Feature 3 (the day-tabs the Backdrop sits behind, unchanged across tab switches). Adds the third external dependency (Wikimedia) and its seam, with a focused test on the graceful-miss contract (no / failed image → returns "none", never throws). Per ADR-0002, building this triggers a `/sync-project-docs` update to the factual *3rd-party tech* section of `Technical-Context.MD`.
