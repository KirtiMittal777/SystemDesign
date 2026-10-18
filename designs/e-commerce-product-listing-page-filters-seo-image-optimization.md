# System Design: E-commerce Product Listing Page (PLP)

## Requirements

**Functional**

- Browse/search products with filters (category, brand, price range, rating, availability),
  sorting (relevance, price ↑↓, rating, newest), and pagination or infinite scroll.
- Product cards: image, name, price, discount, rating, badges (e.g. "Bestseller").
- Filter state must be shareable/bookmarkable (URL-driven) and survive back/forward navigation.
- Responsive: mobile filter drawer, desktop sidebar.

**Non-functional**

- LCP < 2.5s on mid-tier mobile; filter interactions feel instant (< 100ms perceived).
- SEO: listing pages must be crawlable and indexable (they drive organic traffic).
- Handle 100k+ SKUs per category; images at multiple resolutions.

**Out of scope**: checkout, recommendations engine internals, admin/CMS.

## High-level frontend architecture

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│   PLP page  │────▶│ Filter state │────▶│  Data layer     │
│  (route)    │     │  (URL = src  │     │  (React Query / │
│             │     │   of truth)  │     │   SWR + cache)  │
└─────────────┘     └──────────────┘     └────────┬────────┘
       │                                          │ REST/GraphQL
       ▼                                          ▼
┌─────────────┐                          ┌─────────────────┐
│  Components │                          │  Search/PLP API │
│  Grid, Card,│                          │  (faceted query│
│  Filters…   │                          │   endpoint)     │
└─────────────┘                          └─────────────────┘
```

**Rendering strategy: hybrid.** The first page of results is **server-rendered
(SSR)** for SEO and fast LCP — crawlers get full HTML with product data. Filter
changes, sorting, and page 2+ are **client-side fetches** (SPA behavior) so
interactions stay instant. A `?page=` / filter query param scheme keeps every
state crawlable if needed.

## Component breakdown

- `<ProductListingPage>` — route component; parses URL params into filter state,
  owns the data query, renders layout + `<SeoHead>`.
- `<FilterSidebar>` / `<FilterDrawer>` (mobile) — controlled inputs bound to URL
  state; debounces price-range sliders before committing to the URL.
- `<ActiveFilterChips>` — removable chips for each applied filter; clear-all.
- `<SortDropdown>` — sort key; part of the same URL state.
- `<ProductGrid>` — **virtualized** (windowing) for long lists; renders
  `<ProductCard>` rows.
- `<ProductCard>` — image with `srcset`/`sizes`, lazy-loaded below the fold,
  priority-loaded for the first row; price, rating stars, badges.
- `<Pagination>` or `<InfiniteScrollSentinel>` — see tradeoffs.
- `<FilterSkeleton>` — skeleton cards during refetch (keeps layout stable, avoids CLS).

## Data flow

1. User toggles a filter → updates URL query params (`router.replace`, shallow) →
   debounced 150–250ms for sliders/text → query key changes →
   data layer fetches `/api/products?category=…&brand=…&price_min=…&sort=…&page=…`.
2. Response returns `{ items, total, facets, page, pageSize }`. **Facets come back
   with the results** so filter counts (e.g. "Nike (128)") always reflect the
   current query — no separate facet request.
3. `stale-while-revalidate`: show cached page instantly, revalidate in background;
   scroll position restored on back-navigation via cached queries + `scrollRestoration`.

## API shapes

```ts
// GET /api/products?category=shoes&brand=nike,adidas&price_min=50&price_max=200
//     &rating_min=4&in_stock=true&sort=price_asc&page=2&page_size=48
interface PLPResponse {
  items: ProductCard[];      // page of results
  total: number;             // total matching SKUs
  page: number;
  pageSize: number;
  facets: {
    brands: { value: string; count: number }[];
    priceBuckets: { min: number; max: number; count: number }[];
    ratings: { stars: number; count: number }[];
  };
}
interface ProductCard {
  id: string; slug: string; name: string;
  price: number; listPrice?: number; currency: string;
  rating: number; reviewCount: number;
  image: { src: string; srcSet: string; alt: string; width: number; height: number };
  badges: ('bestseller' | 'new' | 'sale')[];
  inStock: boolean;
}
```

Prefer **one faceted endpoint** over N filter endpoints — a single round trip
keeps filter counts consistent with results.

## Caching / loading strategy

- **HTTP cache**: PLP API responses cached at the CDN keyed by normalized query
  string (`stale-while-revalidate: 60, stale-if-error: 300`) — filter combos are
  highly repeatable across users.
- **Client cache**: React Query with `keepPreviousData` — changing page/sort keeps
  the old grid visible while the new one loads (no flash of skeletons on every click).
- **Prefetching**: on card hover/focus, prefetch the PDP (product detail) data and
  the next page when the user nears the end of the current one.
- **Images**: `loading="lazy"` below the fold, `fetchpriority="high"` for the
  first 2–4 cards; AVIF/WebP via `srcset`, fixed aspect-ratio boxes to prevent CLS.

## Performance considerations

- **Virtualize the grid** past ~50 rendered cards — DOM nodes are the #1 PLP
  killer on low-end phones. (With SSR page 1, hydrate into the virtualized list
  on the client.)
- **Debounce expensive filters** (price slider, search-within-results) 150–250ms;
  instant-apply for checkboxes.
- **Bundle**: code-split the filter drawer and sort dropdown (mobile-only code
  shouldn't ship to desktop); tree-shake icon sets.
- **INP**: filter commits are URL replaces + cache lookups — keep main-thread work
  under 50ms; move facet-count computation to the server.
- **Web Vitals**: track LCP/CLS/INP per PLP template in RUM, segmented by device
  class — regressions here directly cost revenue.

## Tradeoffs

| Decision | Chosen | Alternative | Why |
|---|---|---|---|
| Pagination vs infinite scroll | **Numbered pagination** (SEO pages) + optional infinite scroll on mobile | Infinite scroll everywhere | Numbered pages are crawlable and let users jump; infinite scroll hurts footer access and back-button UX |
| SSR first page | **Yes** | Full CSR | SEO + LCP win; CSR-only listing pages are invisible to some crawlers |
| Facets with results | **Single response** | Separate facet endpoint | Consistency + one round trip; slightly larger payload |
| Filter state location | **URL query params** | In-memory/Zustand | Shareable, bookmarkable, back-button-safe; minor URL-encoding care for arrays |

## Follow-up angles an interviewer may probe

- How do you keep facet counts correct when multiple filters combine? (Server
  computes facets against the query *excluding* each facet's own dimension —
  standard "disjunctive faceting".)
- 100k SKUs, user jumps to page 2000 — deep pagination? (Cursor-based pagination
  or cap page depth; `OFFSET` doesn't scale.)
- How do you A/B test the ranking algorithm on the PLP without layout shift?
  (Server-side assignment, render variant in SSR HTML.)
