# Chat Application — Frontend System Design

## Requirements

Design the client for a real-time 1:1 + group chat app (think WhatsApp /
Slack DMs) for millions of users.

**Functional**
- Conversation list + message thread view; text, images, files, emoji reactions.
- Typing indicators, read receipts, online/last-seen presence.
- Message search within a conversation; unread badges and counts.

**Non-functional**
- Messages feel instant: optimistic send with < 100ms perceived latency.
- Ordering guarantees: no duplicates, no gaps, correct order after reconnect.
- Works through network flakiness: offline compose, send queue, conflict-free
  sync on reconnect.
- Battery- and data-friendly on mobile web.

**Out of scope**: E2E encryption internals, media transcoding, push-notification
infrastructure (we consume it).

## High-Level Frontend Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│ App Shell (nav, conversation list, toasts)                         │
├──────────────────────────────────────────────────────────────────┤
│ ChatPage                                                          │
│  ├─ ConversationList (virtualized, unread badges)                 │
│  ├─ ThreadView (virtualized message list, newest at bottom)       │
│  │   ├─ MessageBubble (memoized; text / image / file / system)    │
│  │   ├─ DateDividers + unread marker                              │
│  │   └─ ReactionBar (emoji, optimistic)                           │
│  ├─ Composer (textarea, attachments, send queue state)            │
│  ├─ TypingIndicator  ├─ PresenceDot (online/last-seen)            │
├──────────────────────────────────────────────────────────────────┤
│ Data layer: chatStore (normalized: conversations, messages by id) │
│  + syncEngine (websocket, ack tracking, outbox, reconnect)        │
├──────────────────────────────────────────────────────────────────┤
│ Transport: WebSocket (messages, typing, presence, receipts)       │
│  + REST (history pagination, search, uploads); SSE/poll fallback  │
└──────────────────────────────────────────────────────────────────┘
```

## Component Breakdown

- **ConversationList**: virtualized; each row shows avatar, name, snippet,
  timestamp, unread count, presence dot. Sorted by `lastActivityAt`; updates
  arrive via websocket and re-sort without remounting rows (stable keys).
- **ThreadView**: virtualized list anchored to the bottom (`react-virtuoso`
  handles bottom-anchored infinite scroll well). Renders date dividers and a
  "new messages" marker at the first unread id.
- **MessageBubble**: memoized; renders by `message.kind`; shows status ticks
  (sending → sent → delivered → read) from the sync engine's ack state.
- **Composer**: textarea with auto-grow; Enter to send; attachments go through
  the upload manager first (progress bar), then the message references the
  uploaded file id.
- **TypingIndicator / PresenceDot**: driven by ephemeral websocket events with
  client-side timeouts (typing expires after ~5s without refresh).

## Data Flow

**Sending a message (optimistic UI)**:

1. User hits send → create a message with a **client-generated id**
   (`crypto.randomUUID()`) and status `sending`; insert into the store
   immediately — the user sees it in < 1 frame.
2. Push the payload into the **outbox** (persisted in IndexedDB) and transmit
   over the websocket: `{ clientId, conversationId, body, attachments }`.
3. Server acks with `{ clientId, serverId, serverTimestamp }` → replace the
   client id with the server id, status `sent`. Idempotency: the server dedupes
   on `clientId`, so retries after a dropped ack never duplicate.
4. `delivered` / `read` receipts arrive as separate events → update ticks.
5. On failure: status `failed` with a retry button; the outbox survives reloads.

**Receiving**: websocket `message.new` → insert into store (dedupe by server
id; ignore if `clientId` matches a pending optimistic message — the ack path
already handled it) → if the thread is open and scrolled to bottom, mark read
and emit a `read` receipt; otherwise bump the unread count.

**Reconnect**: on websocket close, exponential backoff with jitter; on
reconnect, send `sync.since = lastServerSeq` → server replays missed events
(messages, receipts, presence) in order → reconcile outbox (resend unacked).

**History pagination**: `GET /conversations/:id/messages?before=<serverSeq>&limit=50`
(cursor = server sequence number, not offset). Prepend pages above the current
viewport while preserving scroll position.

**Typing**: `typing.start` / `typing.stop` events per conversation; the client
shows "X is typing…" and clears it after 5s without a refresh (handles lost
`stop` events).

**Presence**: `presence.update { userId, status, lastSeenAt }`; subscribe only
for visible conversations to bound fan-out; heartbeat every ~30s while the tab
is visible.

## API Shapes

```ts
interface Message {
  id: string;            // server id after ack; clientId before
  clientId?: string;     // echoes back on ack for reconciliation
  conversationId: string;
  senderId: string;
  kind: "text" | "image" | "file" | "system";
  body: string;
  attachments?: { fileId: string; name: string; mime: string; size: number; url: string }[];
  reactions?: { emoji: string; userIds: string[] }[];
  serverSeq: number;     // monotonic per conversation — ordering + sync cursor
  createdAt: string;    // server timestamp (single source of truth)
  status?: "sending" | "sent" | "delivered" | "read" | "failed"; // own messages
}

// WebSocket frames (JSON)
type ClientFrame =
  | { t: "message.send"; clientId: string; conversationId: string; body: string; attachments: string[] }
  | { t: "message.read"; conversationId: string; upToSeq: number }
  | { t: "typing.start" | "typing.stop"; conversationId: string }
  | { t: "sync.since"; lastServerSeq: number };

type ServerFrame =
  | { t: "message.ack"; clientId: string; serverId: string; serverSeq: number; createdAt: string }
  | { t: "message.new"; message: Message }
  | { t: "message.receipt"; serverId: string; status: "delivered" | "read" }
  | { t: "conversation.read"; conversationId: string; userId: string; upToSeq: number }
  | { t: "typing"; conversationId: string; userId: string; active: boolean }
  | { t: "presence"; userId: string; status: "online" | "offline"; lastSeenAt: string }
  | { t: "sync.replay"; events: ServerFrame[] };
```

## Caching & Loading Strategy

- **Normalized store**: `messagesById` + per-conversation ordered id lists;
  conversation list rows and thread views share the same entities.
- **IndexedDB**: last N messages per open conversation + the outbox; cold start
  renders cached thread instantly, then syncs.
- **Media**: thumbnails inline; full images lazy-load on tap; files show
  metadata first, download on demand.
- **Search**: `GET /conversations/:id/messages/search?q=` server-side; highlight
  matches client-side; jump-to-message scrolls the virtualized list to the id.
- **Connection fallback**: if websockets are blocked, degrade to long-polling
  (or SSE for server→client + REST for sends) — same frame shapes, different
  transport.

## Performance Considerations

- Virtualize both lists; threads can hold 100k+ messages — never mount them all.
- Keep the websocket handler off the render path: batch store updates
  (e.g. one state update per animation frame) during replay bursts.
- Memoize bubbles; reactions update only the reaction bar, not the bubble.
- Cap typing/presence re-renders: these are high-frequency ephemeral events —
  keep them in a separate lightweight store slice or refs + targeted updates.
- Message list images: fixed aspect-ratio placeholders to avoid layout shift
  as media loads mid-scroll.

## Tradeoffs

| Choice | Pro | Con |
|---|---|---|
| Optimistic send + clientId | Feels instant | Reconciliation + rollback complexity |
| Server sequence numbers | Total order, gap detection | Server must be the single sequencer per conversation |
| WebSocket primary | Low latency, bidirectional | Connection mgmt, battery, corp-network blocks → need fallback |
| IndexedDB outbox | Survives reloads/offline | Async API, quota limits, cleanup policy needed |
| Bottom-anchored virtualization | Natural chat UX | Scroll-position preservation on prepend is fiddly |
| Ephemeral typing events | Cheap, no storage | Can show stale "typing…" if stop is lost → client timeout required |

**Scaling notes**: the client never invents order — `serverSeq` is the truth.
Presence subscriptions are scoped to visible conversations to avoid fanning out
to thousands of users. Everything the client does optimistically must be
undoable, because the network will eventually disagree.
