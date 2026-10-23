# Photo Gallery with Lazy Loading + Placeholders

## Requirements

Design the frontend for a photo gallery app (think Google Photos / Unsplash
grid) serving millions of photos:

**Functional**
- Infinite-scroll masonry-ish grid of photos with titles.
- Click a photo → full-screen viewer with prev/next navigation.
- Search by text; filter by album/tag.

**Non-functional**
- First paint fast on 3G; no layout shift as images load (CLS ≈ 0).
- Smooth 60fps scroll with thousands of items in the DOM budget.
- Respect data-saver users; don't download what isn't seen.

**Out of scope**: upload pipeline, auth, server-side image processing
(assume a CDN image service that can resize/crop/format by URL params).

## High-level frontend architecture

```
┌──────────────┐   ┌──────────────┐   ┌──────────────────┐
│  GalleryPage │──▶│  VirtualGrid │──▶│  PhotoCard       │
└──────────────┘   └──────────────┘   │  ├─ Placeholder  │
       │                    │         │  └─ LazyImage    │
       ▼                    ▼         └──────────────────┘
┌──────────────┐   ┌──────────────┐   ┌──────────────────┐
│ PhotoStore   │◀──│ ImageLoader  │──▶│  LightboxViewer  │
│ (paged cache)│   │ (priority q) │   └──────────────────┘
└──────────────┘   └──────────────┘
```

- **GalleryPage**: route-level component; owns search/filter state in the URL
  (shareable links) and wires the store to the grid.
- **VirtualGrid**: windowed rendering — only mounts rows near the viewport.
- **PhotoStore**: paged cache of photo metadata (`Map<page, Photo[]>`);
  dedupes in-flight fetches; serves the grid and prefetches the next page.
- **ImageLoader**: priority queue for actual image bytes (visible > near
  viewport > prefetch), with concurrency limit and cancellation.
- **LightboxViewer**: full-screen viewer, preloading neighbors.

## Component breakdown

| Component | Responsibility |
|---|---|
| `GalleryPage` | URL-synced filters, error/empty states, page shell |
| `VirtualGrid` | Windowing, scroll container, overscan rows |
| `PhotoCard` | Aspect-ratio box, placeholder → image swap, title overlay |
| `LazyImage` | IntersectionObserver-driven load, blur-up transition |
| `LightboxViewer` | Full-res load, keyboard nav, neighbor prefetch, focus trap |

## Data flow

1. **Metadata first, bytes later.** `GET /api/photos?page=N&limit=50&q=…`
   returns lightweight JSON including each photo's `width`, `height`,
   `blurhash`, and URL templates — typically < 50KB per page. Images are
   never fetched from metadata alone.
2. **Render placeholders immediately** from known dimensions: a box with
   `aspect-ratio: w / h` reserves exact space → zero layout shift. Paint the
   `blurhash` (or a tiny 20px thumbnail, ~1KB) as the background.
3. **IntersectionObserver** on each `PhotoCard` fires → enqueue the real
   image URL in `ImageLoader` with priority by distance from viewport.
4. **On load**, crossfade placeholder → image (`opacity` transition on a
   stacked `<img>`); on error, show a retry affordance.
5. **Scroll** triggers the store to fetch page N+1 when the user nears the
   end (sentinel div observed at the grid bottom); already-fetched pages are
   served from the `Map` cache.
6. **Viewer**: opening sets `?photo=<id>` in the URL (deep-linkable);
   preload `id±1` at medium priority; `Escape`/backdrop click closes and
   restores focus to the originating card.

## API shapes

```ts
// GET /api/photos?page=2&limit=50&q=sunset&album=nature
interface PhotosResponse {
  items: PhotoMeta[];
  nextCursor: string | null;   // cursor pagination — stable under inserts
  total: number;
}

interface PhotoMeta {
  id: string;
  title: string;
  width: number;               // intrinsic dims → aspect-ratio placeholder
  height: number;
  blurhash: string;            // ~30-char placeholder payload
  urls: {
    thumb: string;             // w=400 WebP/AVIF
    large: string;             // w=1600 for viewer
    srcset: string;            // responsive candidates
  };
}
```

- **Cursor pagination** over offset: new uploads don't shift pages and
  duplicate the user's scroll position.
- **CDN URL params** (`?w=400&fm=avif&q=70`) let the client request exactly
  the pixels it will paint — the grid never downloads 4K originals.

## Caching / loading strategy

- **HTTP cache**: immutable, long-lived (`Cache-Control: public, max-age=31536000, immutable`)
  on versioned image URLs; `stale-while-revalidate` on metadata.
- **In-memory**: `PhotoStore` page cache survives filter toggles within a
  session; an LRU cap (~20 pages) bounds memory.
- **Service worker (optional v2)**: cache-first for thumbnails enables
  offline browsing of previously viewed grids.
- **Priority queue**: max 6 concurrent image fetches (browser connection
  limit per origin is ~6 anyway); viewport-visible items jump the queue;
  scrolling fast cancels queued offscreen requests via `AbortController`.
- **Data-saver**: when `navigator.connection.saveData` is true, load
  `thumb` only, skip neighbor prefetch and blur-up animation.

## Performance considerations

- **CLS**: aspect-ratio boxes + blurhash placeholders → no reflow when
  bytes arrive. This is the single highest-leverage decision.
- **DOM budget**: virtualize — render ~3 viewports of rows (overscan),
  recycle via keys. Thousands of `<img>` nodes kill scroll jank even when
  offscreen.
- **Decode cost**: `decoding="async"` and `loading="lazy"` as a baseline;
  prefer AVIF/WebP with JPEG fallback via `<picture>` or CDN content
  negotiation.
- **Scroll handler**: never measure layout in scroll listeners; the
  IntersectionObserver + sentinel approach keeps the main thread free.
- **Metrics to watch**: LCP (hero/first large image), CLS, INP on filter
  changes, and image bytes per session in RUM.

## Tradeoffs

| Decision | Pro | Con |
|---|---|---|
| Blurhash vs tiny thumbnail | ~30 chars inline, no extra request | Slightly blocky; needs a decoder (~2KB) |
| Virtualization vs full render | 60fps at any scale | Breaks find-in-page, complicates variable heights (measure + cache row heights) |
| Cursor vs offset pagination | Stable under inserts | Can't jump to "page 47" — acceptable for infinite scroll |
| Client-side search vs server | Instant for cached pages | Full corpus search must be server-side; hybrid: server search, client filter |
| Prefetch neighbors in viewer | Instant prev/next | Wasted bytes if the user closes immediately — gate on data-saver and idle network |
