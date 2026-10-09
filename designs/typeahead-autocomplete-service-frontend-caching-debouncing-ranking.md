# System Design: Typeahead / Autocomplete Service

Design the frontend for a search-as-you-type experience (e.g., product search at
Amazon-scale), covering request strategy, caching, ranking display, and
performance.

## Requirements

**Functional**
- Show suggestions as the user types, ranked by relevance/popularity.
- Handle 100M+ daily users; p95 suggestion latency < 200ms.
- Support recent searches and trending suggestions on empty query.

**Non-functional (frontend)**
- Don't DDoS your own backend: bound request rate per keystroke.
- Stale or out-of-order results must never be shown.
- Work well on slow networks and low-end devices.

## High-level architecture

```
┌──────────┐  debounce  ┌─────────────┐  cache   ┌────────────────┐
│  Input   │ ─────────► │ Suggestion  │ ───────► │ Suggestion API │
│ component│  200-300ms │  service    │  lookup  │ (edge-cached)  │
└──────────┘            │ (client)    │          └────────────────┘
                        └─────────────┘
  L1: in-memory LRU (prefix trie / Map) ── L2: edge/CDN cached GETs
```

The client owns a **SuggestionService** layer between the UI and the network:
debounce → normalize → L1 cache lookup → network (with dedupe + cancellation)
→ rank/merge → render. The UI component itself stays dumb.

## Component breakdown

- **`SearchInput`** — controlled input, emits raw keystrokes; owns focus/blur UX.
- **`SuggestionService`** (framework-agnostic class):
  - `query(prefix)`: returns `{ suggestions, source: 'cache' | 'network' }`.
  - In-flight request map keyed by normalized prefix → automatic dedupe.
  - `AbortController` per request; superseded requests abort on new keystroke.
- **`SuggestionList`** — virtualized if > ~50 rows; keyboard nav (roving
  `aria-activedescendant`); renders loading / empty / error states.
- **`RecentSearchesStore`** — localStorage-backed, capped at N=10, shown on
  empty query before any network call.
- **`Telemetry`** — logs keystroke→render latency, cache hit rate, abandonment.

## Data flow

1. Keystroke → debounce 250ms (150ms on desktop, 300ms on mobile is a common split).
2. Normalize: trim, lowercase, collapse whitespace → cache key.
3. **Prefix reuse:** if the cached result for `"iph"` exists, the query for
   `"ipho"` can be *filtered client-side* from the parent prefix's results when
   the parent result set was "complete" (server flags `truncated: false`). This
   eliminates most keystrokes' network calls.
4. Otherwise: GET `/api/suggest?q=<prefix>&limit=8` — cacheable GET, edge-cached
   by full query string, `stale-while-revalidate` for trending.
5. Response tagged with the request's prefix; UI applies only if it still
   matches the current input (stale guard).

## API shapes

```ts
// GET /api/suggest?q=ipho&limit=8
interface SuggestResponse {
  query: string;                 // echoed normalized prefix
  truncated: boolean;            // false → safe for client-side prefix filtering
  suggestions: Array<{
    id: string;
    text: string;
    type: "product" | "category" | "query";
    score: number;               // server rank score
    meta?: Record<string, unknown>;
  }>;
}
```

Ranking happens server-side (popularity, personalization, business rules); the
client merges: **recent searches (local) → cached → network**, deduped by id,
preserving server order within each tier.

## Caching & loading strategy

| Layer | What | TTL |
|---|---|---|
| L1 in-memory LRU (Map, cap ~200 prefixes) | prefix → suggestion list | session |
| L2 edge/CDN on GET | popular prefixes | 60s, SWR |
| localStorage | recent searches only | 30 days |

- Prefetch: on focus with empty query, fetch trending once (warm the dropdown).
- Loading UX: show cached/prior results immediately with a subtle spinner —
  never blank the list on each keystroke (layout stability).

## Performance considerations

- Debounce + prefix filtering typically cuts network calls by 70–90%.
- Render budget: cap at 8–10 visible suggestions; highlight matches with
  `<mark>` — cheap string split, no regex per keystroke on long lists.
- Avoid re-fetch on re-render: the service is a singleton, not per-component state.
- Measure: keystroke→paint (target < 100ms from cache), cache hit rate, and
  "zero-result rate" as a relevance signal.

## Tradeoffs

- **Client-side prefix filtering** saves requests but risks hiding fresher
  server results; mitigate with the `truncated` flag and short TTLs.
- **Aggressive debounce (400ms+)** reduces load but feels laggy; 200–300ms is
  the usual compromise, tuned per device class.
- **Edge caching** is great for head queries but useless for the long tail —
  the L1 in-memory cache covers the tail within a session.
- **WebSocket push for suggestions** is overkill; request/response with
  cancellation is simpler and sufficient. Reserve sockets for the *results*
  page's live updates, not the typeahead.
