# Weather POC — Product Requirements

> Vocabulary follows `business-domain-context.md`. Decisions defer to `docs/adr/` and
> `Technical-Context.MD` per the authority order in `CLAUDE.md`.

## Problem Statement

A person wants to know the weather — right now and over the next week — for a place they care
about, without fuss. They don't want to hunt through a cluttered site, re-type the same city
every time, or squint at a wall of numbers. They want to glance at a clean desktop window, see
the current conditions and a readable forecast for one place, and quickly flip between the
handful of places they check often.

## Solution

A lightweight Windows + macOS desktop app that shows the weather for **one Location at a time**.
The user searches for a place, picks the right match, and sees its **Current Conditions** plus a
**7-day Forecast**. Each day is a tab with a featured headline and a scrollable hour-by-hour
strip. The user can keep places they check often as **Saved Locations** for one-click reselection
from a dropdown. A faint **Location Backdrop** — a classic image of the place — sits behind the
weather to give each Location a sense of place. The app keeps itself current when the user
engages with it, and states plainly when something goes wrong.

## Requirements

1. As a user, I want to type a place name and search for it, so that I can find the Location I care about without knowing its coordinates.
2. As a user, when my search matches several places, I want to see a list of Candidates (e.g. "Paris, FR" vs "Paris, TX"), so that I can pick the exact one I mean.
3. As a user, I want picking a Candidate to immediately become the current Location on screen, so that I see its weather right away.
4. As a user, I want to see the Current Conditions for the present moment at my Location, so that I know what it's like outside right now.
5. As a user, I want Current Conditions to show temperature, feels-like temperature, a condition icon and label, wind, humidity, and precipitation chance, so that I get a complete at-a-glance picture.
6. As a user, I want a 7-day Forecast for my Location, so that I can plan my week.
7. As a user, I want the Forecast presented as one tab per Forecast Day, so that I can flip between days easily.
8. As a user, I want today's tab to feature the live Current Conditions, so that the most relevant reading is front and centre.
9. As a user, I want every other day's tab to feature that Forecast Day's Daily Summary (high/low, condition, precipitation chance), so that I see the day's character even though "now" doesn't apply.
10. As a user, I want each Forecast Day to show a scrollable hour-by-hour strip of Hourly Readings, so that I can see how the day unfolds.
11. As a user, I want each Hourly Reading to show the time, temperature, a condition icon, and precipitation chance, kept compact, so that the strip stays glanceable.
12. As a user, I want today's hourly strip to start at the current hour, so that I see what's coming, not what's past.
13. As a user, I want to save the current Location as a Saved Location with one deliberate action (a star), so that I can return to it quickly later.
14. As a user, I want my Saved Locations in a dropdown, so that I can switch between the places I check often without searching again.
15. As a user, I want picking a Saved Location to behave exactly like a fresh search result — it just sets the current Location, so that there's no special second mode to learn.
16. As a user, I want saving a place I've already saved to be a no-op (deduplicated by its resolved coordinate), so that my list never fills with duplicates.
17. As a user, I want to unsave a Location by toggling the star off, so that I can curate the list.
18. As a user, I want the saved list capped at a sensible number (10), so that the dropdown stays a quick-pick, not a database.
19. As a user, I want to choose between Metric and Imperial units with a single toggle, so that everything (temperature, wind, precipitation, clock) matches how I think.
20. As a user, I want the app to default my units sensibly from my operating system's region on first launch (Metric if it can't tell), so that I rarely have to change it.
21. As a user, I want my unit choice remembered between runs, so that I set it once.
22. As a user, I want my weather to be fetched the moment a Location becomes current (from search, from the dropdown, or restored on launch), so that I always open onto fresh data.
23. As a user, I want the app to quietly refresh when I return to it after a while (if the data has gone stale, ~15 minutes), so that what I'm looking at is up to date without me asking.
24. As a user, I do NOT want a manual refresh button or constant background polling, so that the app stays simple and only works when I'm actually engaging with it.
25. As a user, I want to see when the data was last updated ("Updated HH:MM"), so that I always know how fresh it is.
26. As a user, when a weather fetch fails or I'm offline, I want a clear error state with a way to retry, so that I'm never staring at a blank or silently-stale screen.
27. As a user, I want the app to reopen on the Location I was last viewing, so that I pick up where I left off.
28. As a user, on my very first launch with nothing saved, I want a clear prompt to search for a Location, so that I know how to begin.
29. As a user, when a search finds no places, I want a plain "No places found" message, so that I understand the search worked but matched nothing.
30. As a user, I want a faint classic image of my Location behind the weather (a Location Backdrop), so that the app feels grounded in the place I'm viewing.
31. As a user, I want the Backdrop to stay readable behind the weather (subtle, low opacity), so that it sets mood without obscuring the data.
32. As a user, I want the same Backdrop across all of a Location's day-tabs, so that switching days doesn't flicker the background.
33. As a user, when no suitable image exists for my Location, I want the app to simply show the background colour, so that a missing image is never an error or a broken box.
34. As a user, I want any required image attribution shown unobtrusively, so that the app respects the image licence without getting in my way.
35. As a user, I want the app to look and behave the same on Windows and macOS, so that my experience doesn't depend on my machine.

## Implementation Decisions

**Architecture & seams** (all behind DI per Technical-Context Principle 2):

- **Geocoder** (`IGeocoder`): `Search Query → Candidates`. Wraps the Open-Meteo geocoding endpoint. Separate from the weather fetch so either can be swapped independently.
- **Weather Provider** (`IWeatherProvider`): `Location → Current Conditions + Forecast`. Wraps the Open-Meteo forecast endpoint; a single call returns Current Conditions plus all 7 Forecast Days with their Hourly Readings. Requests data in the active unit system.
- **Location Image Provider** (`ILocationImageProvider`): `Location → Location Backdrop or none`. Wraps the Wikimedia REST page-summary lead image (ADR-0002). Returns "no image" rather than throwing on a miss.
- **App State Store** (`IAppStateStore`): `load() / save(state)` over a single JSON document `{ unitSystem, currentLocation, savedLocations[] }` under `FileSystem.AppDataDirectory` (ADR-0001).

**Pure-logic modules** (no I/O, unit-tested directly):

- **Condition Map**: `WMO weather code → (label, icon)`. Owned by the app, consumed at the weather seam.
- **Unit Formatter**: `(value, unitSystem) → display string` for temperature/wind/precipitation, plus `OS region → default unit system` (US/LR/MM ⇒ Imperial, else Metric; Metric on failure). One bundled toggle, not per-quantity knobs.
- **Freshness Policy**: `(lastFetchedAt, now, window) → shouldRefetch?`, window ≈ 15 minutes. Extracted as a pure function so the "refetch on foreground if stale" rule is testable without a clock or UI.

**Presentation** (XAML + MVVM, thin Views):

- **Weather ViewModel**: orchestrates the fetch lifecycle and the three explicit states (loading / result / error per Technical-Context User Feedback Approach), owns day-tab selection (switching tabs does NOT refetch — the data is already in hand), exposes the "Updated HH:MM" timestamp, and triggers a Freshness-Policy-gated refetch on app foreground/resume.
- **Search ViewModel**: Search Query → Candidates → selection; "No places found" empty result.
- **Saved-Locations ViewModel**: the dropdown, the save/unsave star (dedupe by coordinate, cap 10), and setting the current Location on pick.

**Behaviour contracts:**

- Fetch triggers: Location-becomes-current (search pick, dropdown pick, launch-restore) and a staleness-gated foreground refetch. No manual refresh control, no timed polling.
- Weather is never cached to disk; on launch the Location *identity* is restored and re-fetched fresh.
- A failed weather fetch is a visible error state with retry. A missing Backdrop is silent (decorative) — explicitly NOT a Principle-5 error.
- Persisted store schema is a guarded contract (Technical-Context Principle 3); changing its shape later needs its own ADR.
- Key-less throughout — neither provider needs credentials, so the `SecureStorage` secrets path (Principle 1) stays dormant for the POC.

## Testing Decisions

A good test asserts on **observable, external behaviour** — the deterministic envelope (output shape, state transition, the rendered/persisted result) — never on internal implementation detail or generated provider text. Per Technical-Context, every seam gets a real-IO test on at least one side, and coverage is planned per Feature (tiers, fixtures, Tier-2 ceiling).

- **App State Store** — real-IO: round-trip serialize → write → read → deserialize against a real temporary file; assert the restored state equals what was saved.
- **Weather Provider** & **Geocoder** — Tier-1 recorded-replay: drive against recorded Open-Meteo response fixtures; assert the mapped domain shape (Candidates; Current Conditions + 7 Forecast Days + Hourly Readings), independent of live network.
- **ViewModel ⇄ View binding** — a real-IO binding test on this named seam (the other load-bearing seam in Technical-Context).
- **Condition Map** — unit: every WMO code maps to a label/icon; unknown codes degrade safely.
- **Unit Formatter** — unit: Metric/Imperial formatting for each quantity; OS-region → default unit system including the Metric fallback.
- **Freshness Policy** — unit: boundary cases around the staleness window (just-inside vs just-outside ~15 min) with injected timestamps.
- **Location Image Provider** — focused: the graceful-miss contract (no image / failed fetch → returns "none", never throws); the backdrop is decorative so this behaviour, not image quality, is what's asserted.
- **Views** are not tested directly — they are thin and covered via the binding test.

Prior art: none yet — this is the first Feature build, so these tests establish the patterns (recorded-replay fixtures, real-temp-file round-trips, injected-clock pure tests) the rest of the project will follow.

## Out of Scope

Explicitly deferred — clean fast-follows, not part of this POC:

- **Geolocation / "weather where I am"** — auto-detecting the user's position (new OS permission flow, new seam, denial handling). First-launch is an empty search prompt instead.
- **Independent "next 48 hours" hourly strip** — hourly is only ever a drill-down of a selected Forecast Day.
- **Recently-viewed history** — only explicit Saved Locations appear in the dropdown; no automatic recents.
- **Custom names for Saved Locations** — they always show their geocoded display name.
- **Timed background auto-refresh** — refresh is engagement-driven only (select + stale-foreground).
- **Weather caching for offline viewing** — a failed fetch is an error, not stale data.
- **UV index, sunrise/sunset, and other extended metrics** — beyond the chosen data-point set.
- **Keyed image/weather providers** — both providers are key-less; the secrets path is not exercised.

## Further Notes

- **Authority order** (from `CLAUDE.md`): ADR > Technical-Context > business-domain-context > PRD > Roadmap > Spec > Plan. This PRD yields to the two ADRs and the engineering contract above it.
- **Multi-feature product** — this PRD feeds `/roadmap`, which will sequence Features (e.g. search & resolve, weather view, saved locations, units, backdrop) each becoming its own Spec → Plan.
- **Build-time doc sync** — adopting Wikimedia adds a second 3rd-party dependency; the factual *3rd-party tech* section of `Technical-Context.MD` must be updated when the Backdrop is built (flagged in ADR-0002), via `/sync-project-docs`.
- **Open-Meteo** supplies both geocoding and forecast (key-less); **Wikimedia** supplies backdrops (key-less); **Openverse** is the documented fallback image source (ADR-0002).
