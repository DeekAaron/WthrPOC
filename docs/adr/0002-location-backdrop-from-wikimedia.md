---
status: accepted
---

# Location Backdrop images sourced from Wikimedia, key-less, behind an image-provider seam

The app shows a faint **Location Backdrop** behind each Location's weather. We source it from the
**Wikimedia/Wikipedia REST API page-summary lead image** (the place's canonical "classic" image),
fetched key-less through a dedicated `ILocationImageProvider` seam — not from a keyed photo API.

## Considered Options

- **Keyed photo API (Unsplash / Pexels / Pixabay)** — best-quality, on-topic photos, but each needs
  an API key (activating the Technical-Context Principle 1 secrets/`SecureStorage` path the POC has
  otherwise avoided), carries rate limits, and adds mandatory attribution.
- **Openverse** — 800M+ CC images, key-less anonymous search; but returns many results requiring
  ranking/selection logic and variable relevance. Kept as a documented fallback source.
- **Bundled/static imagery** — no dependency, but not an image *of the location*; rejected by intent.
- **Wikimedia REST page-summary lead image** (chosen) — one deterministic representative image per
  place, one request, no key, no monetary cost.

## Consequences

- **No new secret.** Key-less, so Principle 1 (credentials in `SecureStorage`) stays dormant.
- **A second external dependency** (Wikimedia) is adopted — the factual *3rd-party tech* section of
  `Technical-Context.MD` must be updated when this is built, and it sits behind `ILocationImageProvider`
  per Principle 2.
- **Attribution is required.** Wikimedia/Commons images are CC-licensed (often CC BY-SA) or public
  domain; most demand a visible author/source/licence credit on the view. "Free" ≠ "no strings."
- **Graceful degradation, not a hard error.** Name→image matching can miss (ambiguous/obscure places,
  disambiguation pages). On no-confident-image or fetch failure the view shows only the background
  colour. Because the Backdrop is decorative, a miss is silent — it is *not* a Principle-5 error state
  (which remains reserved for failed weather fetches).
- **Openverse** remains the documented fallback/alternative source should Wikimedia coverage prove
  insufficient.
