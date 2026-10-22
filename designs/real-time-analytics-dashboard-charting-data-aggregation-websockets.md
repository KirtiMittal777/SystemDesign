# Real-Time Analytics Dashboard — Frontend Design

## Requirements

Design the frontend for a real-time analytics dashboard (think Mixpanel/Amplitude-style): users pick a date range and filters, see KPI cards, time-series charts, and a live event table that updates as events stream in.

**Functional:**
- Date-range picker + filters (event name, user segment, device) that update all widgets.
- KPI cards (total events, unique users, conversion rate) computed from the same query.
- Time-series line/bar charts, a funnel chart, and a live event feed table.
- Live updates: new data points arrive every few seconds without a full page reload.
- Shareable URLs: the full dashboard state (range, filters, selected widgets) is encoded in the URL.

**Non-functional:**
- First meaningful paint < 1.5s on a warm cache; chart interactions (zoom/hover) at 60fps.
- Graceful degradation when the socket drops: pause live mode, show staleness, auto-resume.
- Must handle 100k+ data points per chart without freezing the main thread.

## High-Level Frontend Architecture

```
┌─────────────────────────────────────────────────────────┐
│  App Shell (route, auth, feature flags)                 │
├─────────────────────────────────────────────────────────┤
│  URL State Manager (range, filters, widget layout → URL) │
├──────────────┬─────────────────────┬────────────────────┤
│ Query Layer  │  Live Stream Layer  │  Rendering Layer   │
│ (React Query │  (WebSocket +       │  (widget registry, │
│  + chart     │   event buffer +    │   canvas charts,   │
│  data cache) │   merge strategy)   │   virtualized feed)│
└──────────────┴─────────────────────┴────────────────────┘
```

**Key architectural choice:** separate the *historical query path* (REST, cacheable, React Query) from the *live stream path* (WebSocket, append-only buffer). Widgets consume a merged view. This avoids refetching the whole range every few seconds and keeps each path independently testable.

## Component Breakdown

- **`AppShell`** — layout, nav, auth gate, error boundary per widget region (one broken widget never takes down the dashboard).
- **`UrlStateProvider`** — single source of truth for range/filters/layout, serialized to query params with a schema version (`?v=2&range=7d&event=signup&segment=ios`). Deep-linkable and shareable; back/forward works for free.
- **`QueryClientProvider`** (React Query / TanStack Query) — historical data: one query per widget (`queryKey: ["timeseries", range, filters, bucket]`). Stale-while-revalidate: show cached data instantly while refetching on filter change.
- **`LiveSocketProvider`** — owns one WebSocket connection for the whole dashboard. Multiplexes subscriptions per widget; handles reconnect with exponential backoff + jitter; exposes connection state (`live | reconnecting | paused`).
- **Widget registry** — `widgets: Record<WidgetType, { component, defaultQuery, title }>`. Adding a chart type means registering an entry, not touching the shell. Funnel, KPI, timeseries, table all register here.
- **`ChartRenderer`** — thin wrapper around a canvas charting library (uPlot, ECharts with canvas renderer). Canvas, not SVG: 100k points is a DOM-killer in SVG but fine on canvas.
- **`EventFeed`** — virtualized table (windowing) for the live event stream; renders ~30 rows regardless of buffer size.
- **`DataDownsampler`** (worker) — a Web Worker that buckets raw points (per-minute → per-hour) so the main thread never loops over 100k points on each tick.

## Data Flow

1. **Initial load:** `UrlStateProvider` parses URL → widgets fire React Query requests in parallel (`/api/query/timeseries`, `/api/query/kpis`, `/api/query/funnel`). Cached responses render immediately; skeleton shimmer for cache misses.
2. **Live tick:** socket emits `data-point` messages → `LiveSocketProvider` appends to a per-widget ring buffer (cap ~2k points) → widgets merge: historical series (from query cache) + buffered live points, re-bucketed in the worker → chart re-renders only the canvas layer.
3. **Filter change:** URL updates → query keys change → React Query refetches historical data → socket provider re-subscribes (`{ type: "subscribe", filters }`) and clears the live buffer (old live points no longer match the filter).
4. **Reconnect:** backoff retries; on success, request a *delta snapshot* (`since: lastEventId`) instead of a full refetch, then resume streaming. Show a "Live" / "Reconnecting…" / "Paused — data may be stale" pill driven by socket state.

```
 REST (history):  Widget → React Query → GET /api/query/* → cache → render
 WS   (live):     Server → data-point → ring buffer → worker merge → render
 URL  (state):    controls → UrlState → ?params → drives both paths
```

## API Shapes

```ts
// Historical query
GET /api/query/timeseries?event=signup&range=7d&bucket=1h&segment=ios
// → { buckets: [{ t: 1698000000, value: 412 }, ...], meta: { truncated: false } }

// KPI rollup
GET /api/query/kpis?event=signup&range=7d&segment=ios
// → { totalEvents: 120431, uniqueUsers: 88412, conversion: 0.073 }

// Live stream (WebSocket)
→ { type: "subscribe", filters: { event: "signup", range: "7d", segment: "ios" } }
← { type: "data-point", id: "evt_9f2", t: 1698600123, event: "signup", props: { device: "ios" } }
← { type: "heartbeat", serverTime: 1698600125 }   // staleness detection
← { type: "delta", since: "evt_9f2", points: [...] } // after reconnect
```

Pagination for the event feed table uses cursor-based `?before=<id>&limit=50` — offset pagination breaks under live inserts.

## Caching / Loading Strategy

- **React Query cache** keyed by full filter tuple; `staleTime: 30s` for historical ranges, `gcTime: 10min`. Switching back to a recent filter is instant.
- **Request deduplication:** React Query dedupes identical in-flight queries across widgets (KPI card and chart share the same timeseries range).
- **Bucketed prefetch:** on dashboard load, prefetch the next-likely ranges (e.g., `30d` when viewing `7d`) during idle time (`requestIdleCallback`).
- **Live buffer cap:** ring buffer per widget; overflow drops oldest. Prevents unbounded memory growth on a dashboard left open overnight.
- **Skeleton + stale data:** never blank the chart on refetch — keep the stale series visible with a subtle loading overlay.

## Performance Considerations

- **Canvas rendering:** uPlot/ECharts-canvas for time series; SVG only for small multiples/funnels. Target <16ms per frame on hover/zoom by redrawing canvas, not reconciling DOM.
- **Web Worker aggregation:** bucketing 100k raw points → ~500 chart points happens off the main thread; the worker posts a transferable `Float64Array`.
- **Virtualized live table:** fixed row height, `content-visibility: auto`, render window of ~30 rows.
- **Socket message batching:** server sends points in 1s batches (not per event) — 60 renders/sec → 1 render/sec for the feed, charts update on `requestAnimationFrame` throttle.
- **Memoization boundaries:** `React.memo` on each widget; the socket provider publishes via a subscription (zustand/vanilla store), not React context re-renders, so a live tick doesn't re-render the filter bar.
- **Bundle:** code-split each widget type; the charting library is the biggest chunk — load it only on dashboard routes.

## Tradeoffs

| Decision | Pro | Con |
|---|---|---|
| WebSocket (not SSE/polling) | Bidirectional: subscribe/unsubscribe on filter change; low latency | Harder ops (sticky sessions, reconnect logic); falls back to long-poll on restrictive networks |
| Separate history + live paths | Cacheable history, cheap live appends | Merge complexity at the widget layer; filter change must reset both |
| Canvas charts | 100k points at 60fps | Worse a11y than SVG — mitigate with a data table toggle + `aria-label` summaries |
| Ring buffer cap | Bounded memory | Data older than the cap disappears from "live" view (history query still has it) |
| URL-encoded state | Shareable, no backend storage | Long URLs with many filters; schema versioning needed (`v=2`) |
| React Query over hand-rolled cache | Dedup, retries, stale-while-revalidate free | Learning curve; cache key discipline required |

## Failure Modes

- **Socket drops:** pill shows "Reconnecting…", live buffer freezes, historical data stays visible. Auto-resubscribe with delta catch-up; if delta fails, full refetch.
- **Query API slow:** per-widget error boundaries + retry buttons; other widgets unaffected.
- **Clock skew:** heartbeat messages let the client detect server lag and show "data delayed by Xs".
- **Huge ranges (1y of minute data):** server enforces max bucket count (`truncated: true` flag) → client shows "aggregated to daily" notice rather than silently wrong data.
