# places-cache

Permanent cache of Google Places API responses, keyed by `place_id`. Eliminates
repeated paid calls for the same place across the TrueSightDAO codebase.

## Why this repo exists

Several scripts in `TrueSightDAO/go_to_market` (the `market_research` repo) hit
the Google Places API for the same place_ids over time:

- `discover_apothecaries_la_hit_list.py` — Nearby Search + Place Details
- `hit_list_enrich_contact.py` — Place Details for gap-filling
- `hit_list_research_photo_review.py` — Find Place + Details + Photos
- `field_agent_location_places_pull.py` — Place Details
- `backfill_instagram_la_discovery.py`, `hit_list_extract_email_gemini.py` —
  Details for `website` field

Without caching, each script-run re-pays the same per-call price for places
already enriched. This repo caches the API responses permanently so subsequent
calls are free.

## Schema

Each cached place lives at `places/<2-char-prefix>/<place_id>.json` where the
2-char prefix is the first 2 chars of `place_id`. Prefix sharding keeps any
single directory under ~1k files.

```json
{
  "place_id": "ChIJN1t_tDeuEmsRUsoyG83frY4",
  "name": "Empress Organics",
  "fetched_at": "2026-05-01T16:23:00Z",
  "fields_requested": [
    "place_id", "name", "formatted_address", "geometry", "types",
    "business_status", "address_component", "vicinity", "photos",
    "formatted_phone_number", "website", "opening_hours"
  ],
  "result": { "(the Place Details result block, verbatim)": "..." }
}
```

`fields_requested` records which fields were paid for. A cache hit only
satisfies a request when `fields_requested` is a superset of `fields_needed`.
On a miss-by-fields the cache writer fetches the union and overwrites.

## Freshness

Cache is permanent by default. Three Place Details fields decay:
`business_status`, `opening_hours`, `formatted_phone_number`. A future
`scripts/refresh_status.py` sweep re-fetches Basic-tier fields only (free)
for cached entries to catch closures and basic contact updates without
re-paying for the Contact tier.

Explicit cache-bust available via `--refresh PLACE_ID` on any caller.

## Read

- HTTP / CDN: `https://raw.githubusercontent.com/TrueSightDAO/places-cache/main/places/<prefix>/<place_id>.json`
- Programmatic via Contents API: `GET /repos/TrueSightDAO/places-cache/contents/places/<prefix>/<place_id>.json`

## Write

Writes happen via the GitHub Contents API from the consuming script
(`market_research/scripts/places_cache.py`). Each write is a single
`PUT /repos/.../contents/<path>` with sha semantics to avoid lost-update on
concurrent runs.

The PAT used for writes must have contents:write on this repo. The canonical
token name is `PLACES_CACHE_PAT` (see `market_research/.env`); a matching
repository secret exists in `TrueSightDAO/go_to_market` for CI runs.

## Plug-and-play architecture

This is the same materialized-view pattern used by `treasury-cache` and
`agroverse-inventory`: one repo, one schema, many readers. See [Plug-and-play
architecture](https://truesight.me/blog/posts/plug-and-play-architecture-why-every-service-reads-from-one-sheet.html)
and [Discovered protocols](https://truesight.me/blog/posts/discovered-protocols-how-truesight-grows-its-dao-rules-through-use.html)
for broader context.