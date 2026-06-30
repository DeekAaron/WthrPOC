# Weather POC

The domain language for a desktop app that shows a person the current weather and a short
forecast for a single location they choose, with the ability to keep locations for quick reuse.

## Language

**Location**:
A *resolved* place a user views weather for — a geographic coordinate plus a display name, the
single subject currently on screen. Not the text the user typed (see **Search Query**).
_Avoid_: place, city, spot

**Search Query**:
The free text a user types to find a place (e.g. "Paris"). Resolved by geocoding into zero or
more **Candidates**; it is not itself a **Location**.
_Avoid_: search term, query string, input

**Candidate**:
One resolved place returned by geocoding a **Search Query** (e.g. "Paris, FR" vs "Paris, TX").
The user picks a **Candidate** to set the current **Location**.
_Avoid_: result, match, suggestion, hit

**Current Conditions**:
The weather *right now* at a **Location**. Exists only for the present moment — it is the
featured reading on today's view and has no meaning for any other day.
_Avoid_: current weather, now, observation

**Forecast**:
The weather ahead for a **Location**, spanning a short horizon of **Forecast Days**.
_Avoid_: prediction, outlook

**Forecast Day**:
One day within the **Forecast** — the unit the user selects to view. Owns a **Daily Summary**
and a sequence of **Hourly Readings**.
_Avoid_: day card, tab, date

**Daily Summary**:
The single headline reading that characterises a **Forecast Day** (its overall character for the
day). The featured reading shown for any day that is not today.
_Avoid_: headline, daily forecast, overview

**Hourly Reading**:
The weather at one hour within a **Forecast Day** — the small, scrollable per-hour entries.
_Avoid_: hourly forecast, hour card

**Location Backdrop**:
A faint, representative image of the current **Location** (its "classic" image), composited
behind the view's background colour. One per **Location**, the same across every **Forecast Day**.
Decorative — when none is available the view simply shows the background colour.
_Avoid_: wallpaper, background image, hero image, banner

**Saved Location**:
A **Location** the user has deliberately kept for quick re-selection from the dropdown. Explicit
and sticky — it stays until the user removes it. There is no automatic "recently viewed" history.
_Avoid_: favorite (a UI verb only), bookmark, recent, pinned

## Relationships

- The app shows exactly one **Location** at a time.
- A **Location** has **Current Conditions** (today only) and a **Forecast**.
- A **Forecast** is a sequence of **Forecast Days** (a short horizon).
- A **Forecast Day** owns one **Daily Summary** and many **Hourly Readings**.
- Today's view features the **Current Conditions**; every other **Forecast Day** features its
  **Daily Summary**.
- A **Location** may also be a **Saved Location**; saving is an explicit user action.
- A **Saved Location** is one entry in the quick-select dropdown.

## Example dialogue

> **Dev:** "When the user picks a **Saved Location** from the dropdown, is that different from
> searching for it fresh?"
> **Domain expert:** "No — picking a **Saved Location** just sets the current **Location**. Same
> view, same **Current Conditions** and **Forecast**; it's only a faster way to choose."

## Flagged ambiguities

- "save" vs "favorite" — resolved: one concept, **Saved Location** (explicit). "Favorite" is a UI
  verb at most, not a separate kind of thing.
