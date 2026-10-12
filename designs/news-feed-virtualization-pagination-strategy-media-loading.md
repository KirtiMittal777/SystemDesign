# News Feed — Frontend System Design

Design the frontend of a social news feed (think Facebook / X home timeline):
an infinite-scrolling list of posts with text, images, and video, plus
like/comment/share interactions.

## Requirements

**Functional**

- Infinite-scrolling feed of posts: text, single/multiple images, video.
- Like, comment (drawer/modal), share; counts update live-ish.
- "N new posts" pill when fresh content arrives — never yanks the list.
- Pull-to-refresh; graceful read-only offline mode.

**Non-functional**

- First paint < 2s on 4G; scrolling holds 60fps; images cause zero layout shift.
- Ranking/personalization lives server-side — the client renders ordered pages
  and must tolerate inserts and reorders without duplicating or dropping posts.
- Assumed scale: millions of users; feed reads dominate writes ~100:1.

## High-level frontend architecture

```
┌─────────────┐      ┌──────────────┐      ┌──────────────────┐
│  FeedPage   │─────▶│  Feed API    │─────▶│ Ranking / posts  │
│  (React)    │◀─────│  (BFF)       │      │ services         │
└──────┬──────┘      └──────────────┘      └──────────────────┘
       │
       ├──▶ Server cache (React Query, keyed by page cursor)
       ├──▶ Virtualized list (TanStack Virtual, overscan ~5)
       ├──▶ Media layer (IntersectionObserver, blur placeholders)
       └──▶ Offline cache (service worker + IndexedDB, last N pages)
```

Three state layers, kept separate on purpose:

1. **Server state** — pages of posts, keyed by cursor; owned by the cache
   library (stale-while-revalidate, normalized by post id).
2. **UI state** — composer draft, open comment thread, optimistic like flags.
3. **Ephemeral render state** — scroll position, measured row heights, visible
   media — never persisted, rebuilt on mount.

## Component breakdown

| Component | Responsibility |
|---|---|
| `<FeedPage>` | Owns pagination, prefetching, virtualization, "new posts" pill |
| `<FeedItem>` | One post row; memoized; self-reports height if dynamic |
| `<MediaAttachment>` | Responsive image (`srcset`/`sizes`, blur-up placeholder) or video (lazy load, play on visible) |
| `<InteractionBar>` | Like/comment/share with optimistic updates + rollback |
| `<FeedSkeleton>` | Shimmer rows matching real row heights for first paint |
| `<NewPostsPill>` | "12 new posts" button; prepends on click without moving scroll |

## Data flow

1. **First paint:** SSR the first page (SEO + TTI) or hydrate from the
   service-worker's cached page. Skeletons only on cold cache.
2. **Scrolling:** when the user passes ~70% of loaded content, fetch the next
   page by cursor and append. Prefetch starts *before* the bottom is hit.
3. **New posts:** websocket (or short-poll) returns only a *count* since the
   last timestamp. Clicking the pill fetches the new head page and prepends —
   the list under the user's thumb never shifts.
4. **Like:** optimistic UI increments immediately, request fires, failure rolls
   back with a toast. (Low-risk action → optimistic; deletes → pessimistic.)
5. **Scroll restore:** back-navigation restores the cursor + scroll offset from
   the page cache instead of refetching page one.

## API shapes

```http
GET /api/feed?cursor=<postId>&limit=20
→ {
    "items": [Post, ...],
    "nextCursor": "p_9f3a…" | null,
    "hasMore": true
  }

Post {
  "id": "p_9f3a…",
  "author": { "id": "u_12", "name": "Ada", "avatarUrl": "https://cdn/…" },
  "body": "…",
  "media": [
    { "type": "image", "url": "…", "width": 1600, "height": 900,
      "blurHash": "LKO2?U%2Tw=w]~RBVZRi};RPxuwH" }
  ],
  "likeCount": 128, "likedByMe": false,
  "commentCount": 14, "createdAt": "2026-10-09T15:30:00Z"
}

POST /api/posts/:id/like   → { "likeCount": 129 }
GET  /api/feed/updates?since=<ISO ts> → { "newCount": 12 }
```

Media URLs are content-hashed and immutable → `Cache-Control: public, max-age=31536000, immutable`.

## Caching / loading strategy

- **Pages:** React Query cache keyed by `["feed", cursor]`; 5-minute stale
  time, stale-while-revalidate on remount; normalize by post id so a like on
  page 3 doesn't re-render pages 1–2 (object identity per post).
- **Prefetch:** `prefetchQuery` for the next cursor when the sentinel row
  enters the viewport; cancel in-flight page fetches on unmount/refresh
  (`AbortController`).
- **Images:** responsive `srcset` + `sizes`; blur-up placeholder from the
  `blurHash` while loading; `width`/`height` (or `aspect-ratio`) reserves the
  box → zero CLS; `decoding="async"`, `fetchpriority="high"` for the first
  viewport only.
- **Video:** `preload="none"`; load + play/pause driven by
  `IntersectionObserver` (threshold ~0.5); muted autoplay where allowed,
  otherwise show a tap-to-play poster.
- **Offline:** service worker caches the last ~5 pages + media in IndexedDB;
  banner switches to "You're offline — showing cached feed."

## Performance considerations

- **Virtualization is non-negotiable** past ~50 complex rows: render visible
  rows + overscan, absolutely positioned in a tall spacer. Stable keys by post
  `id` — never array index (reorders would scramble component state).
- **Row heights:** prefer fixed/estimated heights; if dynamic, measure once
  and cache per post id. `content-visibility: auto` + `contain-intrinsic-size`
  as a CSS assist for the non-virtualized fallback.
- **Re-render budget:** `React.memo` on `<FeedItem>` keyed to the normalized
  post object; interaction state (liked) lives in the normalized store, not in
  row-local state that would force siblings to re-render.
- **Scroll handlers:** passive listeners, rAF-throttled; never `setState` per
  scroll event — the virtualizer reads scroll position via ref.
- **Bundle:** code-split the comments drawer, share sheet, and composer;
  prefetch them on hover/focus intent.
- **Measure:** track INP on like taps, CLS per session, and % of images that
  completed before entering viewport (media pipeline health).

## Tradeoffs

- **Cursor vs. offset pagination:** cursor survives inserts and ranking
  changes (no dupes/skips); offset is simpler but breaks the moment the feed
  reorders. Cursor wins here.
- **Optimistic vs. pessimistic updates:** optimistic for likes (cheap,
  reversible); pessimistic for destructive actions. State the rule, don't
  guess per feature.
- **SSR first page vs. client-only:** SSR buys TTI and SEO; after hydration,
  all pagination is client-side against the same API. Cost: SSR infra + cache
  invalidation complexity.
- **Websocket vs. polling for updates:** a socket per user is expensive at
  millions of DAU; short-polling a lightweight `newCount` endpoint (or SSE)
  plus a user-triggered prepend is the pragmatic default.
- **Virtualization vs. full render:** virtualization adds complexity with
  dynamic heights and breaks find-in-page; keep a non-virtualized path for
  short feeds and print views.