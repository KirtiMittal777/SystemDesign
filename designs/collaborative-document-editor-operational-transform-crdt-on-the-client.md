# Collaborative Document Editor (Frontend System Design)

**Category:** Frontend System Design · **Difficulty:** Hard

> Design the frontend for a Google-Docs-style collaborative editor:
> multiple users editing the same document concurrently, with presence
> (who's viewing/editing), live cursors, comments, and offline editing that
> syncs later.

## Requirements

**Functional**
- Rich-text editing (bold/italic/headings/lists), ~10k-word documents.
- Real-time collaboration: remote edits appear < 100 ms, no lost keystrokes.
- Presence: avatar stack of viewers, colored remote cursors + selections.
- Comments anchored to text ranges; resolve/reopen.
- Offline editing with conflict-free merge on reconnect.
- Version history with diff view and restore.

**Non-functional**
- 60 fps typing on low-end devices; no full re-renders per keystroke.
- Handles 50+ concurrent editors on one doc.
- Graceful degradation when the socket drops (banner + queued ops).
- Accessible editing surface (screen-reader announcements of remote edits).

## High-level frontend architecture

```
┌──────────────────────────────────────────────────────────┐
│                        React app                          │
│  ┌──────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │ Editor   │  │ Presence     │  │ Comments / History   │  │
│  │ surface  │  │ layer        │  │ panels               │  │
│  └────┬─────┘  └──────┬───────┘  └──────────┬───────────┘  │
│       │               │                     │              │
│  ┌────▼───────────────▼─────────────────────▼───────────┐  │
│  │              Sync engine (CRDT core)                  │  │
│  │  local doc model · op log · undo manager · snapshots  │  │
│  └────┬──────────────────────────────┬──────────────────┘  │
│       │                              │                      │
│  ┌────▼─────────┐              ┌───────▼────────┐            │
│  │ Persistence  │              │ Transport      │            │
│  │ IndexedDB    │              │ WebSocket      │            │
│  │ (doc+oplog)  │              │ (+SSE fallback)│            │
└──────────────────────────────────────────────────────────┘
```

**Key decision: CRDT over OT on the client.** Operational Transform requires a
central server to serialize and transform operations — every keystroke
round-trips for ordering, which hurts offline support and complicates the
client. A **CRDT** (e.g. Yjs-style, or Automerge) lets each client apply edits
locally and merge them deterministically later. Convergence is guaranteed
without a central authority, which is exactly what offline editing needs.

Concretely: use **Yjs** (`yjs` + `y-prosemirror` or `y-quill`) as the shared
document model. The editor binds to the Yjs doc; the sync engine exchanges
Yjs *updates* (binary diffs), not raw keystrokes.

## Component breakdown

| Component | Responsibility |
|---|---|
| `EditorSurface` | Rich-text editing UI. Binds a ProseMirror/TipTap instance to the Yjs doc via `y-prosemirror`. Local keystrokes mutate Yjs; Yjs remote updates render through the binding. |
| `SyncEngine` | Owns the Yjs `Doc`. Applies local transactions, emits binary updates over the socket, applies incoming updates, persists snapshots + op log to IndexedDB (debounced). Exposes connection state (`connected | reconnecting | offline`). |
| `Transport` | WebSocket client with heartbeat, exponential-backoff reconnect, and message framing (`{ type: 'update' | 'awareness' | 'snapshot-request', payload }`). Falls back to SSE + POST when WS is blocked. Queues outbound updates while offline. |
| `PresenceLayer` | Yjs **Awareness** protocol: ephemeral per-client state (cursor position, selection, user info, color). Renders remote carets/selections as absolutely-positioned overlays; avatar stack in the toolbar. Awareness is *not* persisted — it expires on disconnect. |
| `CommentThreads` | Comments anchored to Yjs relative positions (stable across remote edits). Panel lists threads; clicking a thread scrolls to its anchor. |
| `HistoryPanel` | Snapshot list; "restore" applies a reverse diff as a new commit (never rewrites history). |
| `ConnectionBanner` | Offline/reconnecting indicator + "N changes pending sync". |

**Why not put Yjs state in React state?** Typing at 60 fps through React
re-renders is a non-starter. The editor surface mutates the DOM imperatively
(ProseMirror does this natively); React owns the *chrome* (toolbar, panels,
presence avatars). The bridge is event subscriptions, not state.

## Data flow

**Local keystroke (optimistic by construction):**
1. User types → ProseMirror transaction → `y-prosemirror` writes to Yjs doc.
2. SyncEngine's `doc.on('update')` fires → binary update appended to IndexedDB
   op log (debounced 2 s) and sent over the WebSocket.
3. UI updates instantly — no server round-trip needed. With CRDTs, "optimistic
   UI" isn't a special case; it's the default.

**Remote edit:**
1. WebSocket receives binary update → `Y.applyUpdate(doc, update)` →
   ProseMirror binding re-renders only the changed nodes.
2. Cursor/selection positions are remapped through Yjs relative positions, so
   your caret doesn't jump when someone types above you.

**Presence:**
1. Local cursor moves → throttled (≤10 Hz) Awareness update broadcast.
2. Remote Awareness updates → PresenceLayer positions colored carets.
3. Disconnect → awareness entry expires after ~30 s timeout; avatar greys out.

**Offline → reconnect:**
1. While offline, all updates accumulate in the IndexedDB op log.
2. On reconnect: client sends a *state vector*; server replies with only the
   missing updates (delta sync, not full doc) → both sides converge.
3. Conflicts are impossible by construction (CRDT merge is commutative) —
   concurrent edits to the same word interleave deterministically by
   client-ID ordering. No "conflict resolution UI" needed for text.

## API shapes

WebSocket frames (JSON envelope, binary payload as base64 or ArrayBuffer):

```ts
// Client → server: document update
{ type: "doc-update", docId: "doc_123", update: "<base64 yjs update>", clientId: "c_abc" }

// Server → client: remote updates (batched, ~16ms window)
{ type: "doc-update", docId: "doc_123", updates: ["<b64>", "<b64>"], fromClientIds: ["c_xyz"] }

// Ephemeral presence (never persisted server-side)
{ type: "awareness", docId: "doc_123", clientId: "c_abc",
  state: { user: { name: "Kirti", color: "#7c3aed" },
           cursor: { anchor: 142, head: 158 }, viewport: { from: 0, to: 900 } } }

// Delta sync on reconnect
{ type: "sync-step1", docId: "doc_123", stateVector: "<b64>" }
{ type: "sync-step2", docId: "doc_123", diff: "<b64 updates since vector>" }
```

REST (documents, comments, history — cacheable, non-realtime):

```
GET  /api/docs/:id            → metadata + latest snapshot id
GET  /api/docs/:id/snapshot   → full Yjs snapshot (for cold start)
GET  /api/docs/:id/comments   → comment threads with Yjs anchors
POST /api/docs/:id/comments   → create thread
GET  /api/docs/:id/history    → snapshot list (paginated)
```

## Caching / loading strategy

- **Cold start:** fetch latest snapshot from CDN-cached REST endpoint →
  hydrate Yjs doc → open WebSocket → delta-sync anything newer than the
  snapshot. Editor becomes interactive from the snapshot in < 1 s; live
  collaboration attaches a moment later.
- **Warm start:** IndexedDB holds the last doc state + op log. Render
  instantly from disk, then delta-sync. True offline-first.
- **Snapshots:** server compacts the op log into a snapshot every N updates
  or M minutes; clients discard ops older than the snapshot they hold.
- **Presence:** never cached, never persisted — pure ephemeral state.
- **Comments/history:** React Query-style caching with the Yjs anchors as
  cache keys; invalidated on new comment events via the socket.

## Performance considerations

- **Typing latency:** ProseMirror + Yjs binding updates only affected DOM
  nodes; React is not in the keystroke path. Target: input-to-paint < 16 ms.
- **Large documents:** virtualize the *comment/history panels*, not the editor
  (editors need real DOM for caret math); cap undo depth (e.g. 500 steps);
  lazy-load images as blob URLs with placeholder dimensions to avoid layout
  shift.
- **50 concurrent editors:** throttle awareness to 10 Hz/client; batch remote
  updates in a 16 ms window and apply once per frame; render remote carets on
  a single absolutely-positioned overlay layer (no per-caret React
  re-renders — use refs + direct DOM updates).
- **Memory:** IndexedDB op log is compacted on each snapshot; in-memory Yjs
  doc for a 10k-word doc is ~1–2 MB — fine.
- **Bundle:** lazy-load `yjs`, history panel, and comment UI via dynamic
  `import()`; the core editor shell stays lean.

## Tradeoffs

| Decision | Gain | Cost |
|---|---|---|
| CRDT (Yjs) over OT | Offline editing, no central ordering server, merge is automatic | Larger payloads than OT ops; tombstones grow the doc (mitigated by snapshots); rich-text schema must map cleanly to Yjs types |
| ProseMirror over contentEditable-from-scratch | Battle-tested editing, accessibility, IME/mobile support | Learning curve; binding layer (`y-prosemirror`) is the trickiest integration point |
| WebSocket + SSE fallback | Real-time where possible, works behind hostile proxies | Two transport code paths to maintain |
| IndexedDB persistence | Instant warm start, offline editing | Async API complicates boot sequencing; quota errors need handling |
| Awareness for presence (ephemeral) | Cheap, auto-expiring | No "who was here" history — acceptable, presence is inherently live |

## What I'd whiteboard in the interview (5-minute arc)

1. "One shared model: Yjs CRDT doc. Everything — text, comments anchors —
   lives in it."
2. "Local edits apply instantly; binary diffs flow over a socket; merge is
   automatic — that's why offline works."
3. "Presence is a separate ephemeral channel (Awareness), not part of the
   document."
4. "Cold start from CDN snapshot + delta sync; warm start from IndexedDB."
5. "React owns the chrome; the editor surface mutates the DOM imperatively —
   React is not in the keystroke path."
