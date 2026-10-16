# Video Player Page — Adaptive Streaming UI, Buffering States

## Requirements

### Functional
- Play / pause / seek (click + drag on seek bar), volume, mute, playback speed.
- **Adaptive bitrate streaming** (HLS or DASH): quality auto-switches with bandwidth; manual quality override.
- **Buffering states:** distinct UI for initial load, rebuffering mid-playback, and seeking — with buffered-range display on the seek bar.
- Captions/subtitles (multi-language), fullscreen, picture-in-picture, keyboard shortcuts.
- Resume playback from last position; thumbnail previews while scrubbing.
- Playback analytics (QoE): time-to-first-frame, rebuffer count/ratio, bitrate switches.

### Non-functional
- Time-to-first-frame < 2s on broadband; rebuffer ratio < 1%.
- 60fps controls; no dropped frames caused by UI work.
- Works on Safari (native HLS), Chrome/Firefox/Edge (MSE), mobile browsers.

## High-level frontend architecture

```
┌─────────────────────────────────────────────────────┐
│ VideoPlayerPage                                     │
│  ┌───────────────────────────────────────────────┐  │
│  │ <video> element (dumb renderer)               │  │
│  └───────────────────────────────────────────────┘  │
│  PlayerController (state machine + ABR + buffer)   │
│   ├── ManifestLoader → variant/segment URLs        │
│   ├── ABREngine → picks rendition per bandwidth    │
│   ├── SegmentFetcher → fetch → MSE SourceBuffer    │
│   └── AnalyticsReporter → QoE beacons              │
│  PlayerUI (React, low-frequency state only)        │
│   ├── ControlsBar  SeekBar  Volume  QualityMenu    │
│   ├── OverlayStates (spinner / error / ended)      │
│   └── ScrubThumbnails (sprite sheet)               │
└─────────────────────────────────────────────────────┘
```

Key split: the **controller is imperative and ref-held** (it deals with millisecond-level media events); **React owns only low-frequency UI state** (paused, currentTime rounded, quality label). High-frequency `timeupdate` never goes through React state directly.

Use **hls.js** (or Shaka Player) on MSE browsers; **native HLS** (`video.src = manifestUrl`) on Safari. Wrap both behind a small `PlayerAdapter` interface so the UI never branches on engine.

## Component breakdown

| Component | Responsibility |
|---|---|
| `VideoPlayerPage` | Route-level: fetches video metadata, owns controller lifecycle, error boundary |
| `PlayerController` (non-visual class) | Playback state machine, ABR decisions, segment pipeline, MSE management |
| `SeekBar` | Buffered ranges (grey), played range, hover scrubber, keyboard seek |
| `OverlayStates` | Loading spinner, rebuffer indicator, error + retry, "up next"/ended |
| `QualityMenu` / `CaptionsMenu` / `SpeedMenu` | Settings popovers; quality menu shows current rendition + "Auto" |
| `ScrubThumbnails` | Sprite-sheet thumbnail under cursor while hovering seek bar |

### Playback state machine

`idle → loading → buffering → playing ⇄ paused → seeking → buffering → playing → ended`, with `error` reachable from anywhere (→ retry → `loading`). The UI renders overlays purely from this state — no ad-hoc `isLoading` flags scattered across components.

## Data flow

1. `GET /videos/:id` → metadata + `manifestUrl` (signed, short-lived).
2. Manifest parsed → variant list (bitrates, resolutions, codecs).
3. ABR engine picks initial rendition from measured/estimated bandwidth (start conservatively — fast first frame beats max quality).
4. Segment fetch loop: download → append to MSE `SourceBuffer` → `buffered` ranges update seek bar.
5. Every segment download updates the bandwidth estimator (EWMA); ABR up/downshifts at segment boundaries (never mid-segment).
6. QoE events (`first_frame`, `rebuffer_start/end`, `bitrate_change`, `error`) batched via `sendBeacon` on pause/unload + interval flush.

## API shapes

```ts
// GET /api/videos/:id
{
  "id": "vid_abc123",
  "title": "Design Systems at Scale",
  "durationSec": 754,
  "posterUrl": "https://cdn.example.com/posters/vid_abc123.jpg",
  "manifestUrl": "https://cdn.example.com/hls/vid_abc123/master.m3u8?sig=...",
  "thumbnailSpriteUrl": "https://cdn.example.com/sprites/vid_abc123.jpg",
  "spriteMeta": { "tileWidth": 160, "tileHeight": 90, "intervalSec": 5, "columns": 10 },
  "captions": [{ "lang": "en", "label": "English", "url": ".../en.vtt" }],
  "resumePositionSec": 132,          // null if never watched
  "chapters": [{ "title": "Intro", "startSec": 0 }]
}

// POST /api/analytics/playback-events  (batched QoE)
{
  "videoId": "vid_abc123",
  "sessionId": "sess_789",
  "events": [
    { "type": "first_frame", "at": 1718049600123, "startupMs": 1180 },
    { "type": "rebuffer", "at": 1718049615400, "durationMs": 840, "bitrateKbps": 1200 },
    { "type": "bitrate_change", "at": 1718049620000, "fromKbps": 2500, "toKbps": 1200 }
  ]
}
```

## Caching & loading strategy

- **Poster + blurred placeholder:** render a tiny blurred poster instantly (LQIP) so the page never shows a black box; swap to full poster, then first frame.
- **`preload="metadata"`** on the `<video>` tag — fetch just enough for duration/dimensions; don't download video bytes until the user hits play (autoplay muted preview is the exception, and only on fast connections via `navigator.connection.effectiveType`).
- **CDN + HTTP cache:** manifests short TTL (live) or immutable (VOD); segments immutable with long `max-age` (content-addressed URLs). Do **not** put multi-MB segments in a service worker cache — you'll evict everything else; rely on browser HTTP cache + CDN.
- **Resume position:** `localStorage` (or IndexedDB for watch history) — restore on load, offer "Resume from 2:12" chip.
- **Player library code-split:** `import('hls.js')` lazily only when MSE path is needed — saves ~100KB on Safari where native HLS is used.
- **Prefetch next segments** one rendition ahead during idle network time; cancel in-flight fetches immediately on seek (AbortController) to avoid wasting bandwidth.

## Performance considerations

- **Decouple UI from media events.** `timeupdate` fires ~4Hz and `progress` unpredictably — throttle seek-bar updates with rAF and keep them in a ref-updated DOM node or a memoized component; never let them re-render the whole control bar.
- **Thumbnail sprites:** one JPEG sprite + CSS `background-position` math — zero extra requests while scrubbing, decoded once.
- **Avoid layout thrash** in the seek bar: compute hover time from `getBoundingClientRect()` once per pointermove (cached on resize).
- **Memory:** evict played segments from the `SourceBuffer` (`remove()`) behind the playhead for long videos — unbounded MSE buffers crash tabs.
- **Web worker for ABR math** is overkill; the estimator is O(1) per segment. Keep it on the main thread, keep it trivial.

## Tradeoffs

| Decision | Chosen | Alternative | Why |
|---|---|---|---|
| Streaming lib | hls.js / Shaka + native HLS fallback | Build on MSE from scratch | ABR, DRM, caption parsing are solved problems; don't re-solve |
| ABR placement | Client-side (bandwidth estimation) | Server-side rendition choice | Client sees real throughput + buffer health; standard for VOD |
| Thumbnails | Sprite sheet | Per-seek API thumbnails | 1 request total vs N; slight manifest complexity |
| Resume storage | localStorage | Server-side watch state | Instant, works logged-out; sync to server opportunistically |
| Autoplay | Muted preview only, gated on connection | Always autoplay with sound | Browser policies block it anyway; respects user + data |
| Error recovery | Retry with rendition downshift, then error UI | Fail immediately | Most playback errors are transient network blips — one silent retry saves the session |
