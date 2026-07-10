# WhatsApp                          source: hellointerview.com/learn/system-design/problem-breakdowns/whatsapp · date: 2026-07-10

## Key constraint 🔑
~200M **concurrent** persistent connections spread across hundreds of hosts — so sender and recipient are almost never on the same box; the whole design hangs on *routing a message to wherever the recipient is connected* while guaranteeing it survives if they're offline.

## Requirements
FR — what it does:
- Send/receive 1:1 and **group** messages (groups ≤100)
- Receive messages sent while **offline** (buffered up to 30 days)
- Send/receive **media** in messages
- Multiple **devices/clients** per user (≤3)     (parked: calling, business, registration/profile)
NFR — the pressures, with numbers:
- **latency: < 500ms** delivery   ← shapes the transport + routing choice
- scale: 4B msgs/day (~40K/s, ~100K writes/s with groups); **200M concurrent conns**, 1–2M/host
- **guaranteed deliverability** (durable before delivered) + resilient to component failure
- retention: stored centrally "no longer than necessary" (TTL)

## Diagram — v1 (simple)
```
┌────────┐  WS Connection   ┌────────┐        ┌─────────────┐   write   ┌──────────────┐
│ Client │ ════════════════▶│ L4 LB  │───────▶│ Chat Server │──────────▶│  Database    │
└────────┘                  └────────┘        └─────────────┘           │  (DynamoDB)  │
                                                                        └──────────────┘

┌─ Chat ──────────────┐     ┌─ Messages ──────────┐
│ Chat                │     │ Inbox               │
│   id                │     │   recipientId       │
│   name              │     │   messageId         │
│   metadata          │     │                     │
│                     │     │ Message             │
│ ChatParticipant     │     │   id                │
│   chatId            │     │   chatId            │
│   participantId     │     │   contents          │
└─────────────────────┘     │   creatorId         │
                            │   timestamp         │
                            └─────────────────────┘
```
> Client holds a WebSocket via an L4 LB to a single Chat Server; server keeps an in-memory
> `userId → WS` map and reads/writes Chat, ChatParticipant, Message, Inbox in DynamoDB.

## Evolve it   (text only — the moves, v1 → final)
- **v1:** one Chat Server behind an L4 LB, holding an in-memory `userId → WebSocket` map; a store for Chat/Message. Client sends over its socket → server looks up participants → pushes over their sockets. Clients `ack` on receipt.
- ➕ **group chats** → write Chat record + one ChatParticipant per user in a single transaction (≤100 items); composite key (chatId, participantId) + **GSI** (participantId → chatId) to list a user's chats.
- ➕ **offline delivery** → per-user **Inbox** of undelivered msgs: write Message + Inbox entries *before* attempting live delivery; on `ack`, delete from Inbox; on reconnect, drain Inbox. TTL cleans old entries.
- ➕ **media** → clients get a **presigned URL**, upload directly to blob storage, and send only the opaque URL through the Chat Server (server never proxies bytes). 30-day TTL on blobs.
- ❌ **(load) one Chat Server can't hold 200M conns** → many Chat Servers — but now recipient may be on a different host (the core routing break).
- ❌ **(load) routing across hosts** → *Kafka topic/user* dies (billions of topics, ~50TB overhead); *consistent-hashing + registry* needs server↔server mesh + thundering herd. → **Redis Pub/Sub**: on connect, server subscribes to the user's channel; senders publish to it; sharded by userId. "At-most-once" is fine because Inbox/Message are durable *before* publish.
- ➕ **multiple devices** → **Clients table**, make **Inbox per-client** (`recipientClientId`), fan out to all of a participant's clients (Pub/Sub still keyed by userId).
- ❌ **(crash) dead sockets look alive** (TCP keepalive too slow) → **app-level heartbeats**: server pings every 10–30s, client must pong in ~5s, else drop + client reconnects and syncs Inbox (~15s detection).
- ❌ **(crash) Redis drops a live message** (at-most-once) → **piggyback a per-user global sequence number on the heartbeat ping**; client requests a sync if behind. (Backed by Redis `INCR`; prod also adds periodic polling.)
- ❌ **(race) out-of-order messages** → don't strictly reorder; servers **NTP-sync** clocks, stamp receipt time, clients sort by timestamp — consistent across clients, occasional "pop-in" accepted.
- ➕ **last seen** → avoid write-per-heartbeat: store **only last disconnect** per user, update on disconnect with a **conditional (only-if-newer)** write; on request, check LastSeen table *and* ping the target's channel for a live ONLINE reply.
- **stop when:** all FRs (group/offline/media/multi-device) built + <500ms, billions-of-conns, and durability targets met.

## Diagram — final
```
                                            ┌─────────────┐  write   ┌──────────────┐
┌────────┐  WS Connection   ┌────────┐      │ Chat Server │ ×2       │  Database    │
│ Client │ ════════════════▶│ L4 LB  │─────▶│             │─────────▶│  (DynamoDB)  │
└────────┘                  └────────┘      └─────────────┘          └──────────────┘
                                                   ▲
                                        subscribe  ║  publish
                                                   ▼
                                            ┌───────────────┐
                                            │ Redis Pub/Sub │ ×2
                                            └───────────────┘

┌─ Chat ──────────────┐   ┌─ Messages ──────────┐   ┌─ Clients ──────┐
│ Chat                │   │ Inbox               │   │ Clients        │
│   id                │   │   recipientClientId │   │   id           │
│   name              │   │   messageId         │   │   userId       │
│   metadata          │   │                     │   └────────────────┘
│                     │   │ Message             │
│ ChatParticipant     │   │   id                │
│   chatId            │   │   contents          │
│   participantId     │   │   creatorId         │
└─────────────────────┘   │   timestamp         │
                          └─────────────────────┘
```
> Chat Servers are now horizontally scaled (×2 shown). Each subscribes on connect to its
> connected users' channels on a sharded Redis Pub/Sub (×2); senders publish to the
> recipient's channel (double-headed = subscribe + publish). Durability still comes from
> DynamoDB (Message/Inbox written before publish). Inbox is now per-client
> (`recipientClientId`) with a Clients table for multi-device fan-out.

## Decisions (choice · why · alt rejected)
- **Transport → WebSocket over TLS** · bi-directional, high-freq send+receive, low latency · alt SSE/long-poll: no client→server push · alt HTTP polling: latency + load
- **LB → L4** · only need connection balancing, no L7 routing features · alt L7: needless cost/overhead
- **Cross-host routing → Redis Pub/Sub (sharded by user)** · lightweight in-memory fan-out, no per-user persistence · alt Kafka-topic-per-user: billions of topics, ~50TB overhead · alt consistent-hash + ZooKeeper/Etcd registry: server-mesh + thundering-herd
- **Data store → DynamoDB** (Chat, ChatParticipant+GSI, Message, Inbox, Clients, LastSeen) · key-value access patterns, TTL, transactions · alt SQL: don't need joins/relational at this scale
- **Media → presigned direct upload to blob storage** · Chat Server never proxies large blobs · alt blob-in-DB: DBs bad at binaries · alt proxy-through-server: wastes server bandwidth
- **Delivery guarantee → durable-before-publish (Inbox) + ack + heartbeat seq** · makes at-most-once Pub/Sub safe · alt rely on Pub/Sub delivery: drops messages
- **Failure detection → app heartbeats** · seconds not minutes · alt TCP keepalive: too slow · alt ack-timeout only: only during active sends
- **Ordering → NTP timestamp, client-side sort** · cheap, cross-client-consistent · alt per-chat total order: coordination cost

## Reusable pattern ♻️
- When I see **persistent connections spread across many hosts + "recipient on a different box"** → reach for a **pub/sub layer keyed by recipient** (subscribe-on-connect), not a topic-per-entity or a server mesh.
- When I see **"must not lose messages" over a lossy/at-most-once channel** → **write durably (inbox) BEFORE you attempt live delivery**, then ack-and-delete; live delivery is an optimization, not the source of truth.
- When I see **large blobs in a message system** → **presigned direct-to-blob upload**, pass an opaque URL; never proxy bytes through the app server.
- When I see **"is X online / dead connection"** → **app-level heartbeats**, and **piggyback state (seq numbers, last-seen) on the heartbeat** you're already paying for.
- When I see **high-frequency status writes (last seen)** → collapse to **one record, conditional only-if-newer write, on state change** instead of per-event.

## Gotchas
- **At-most-once Pub/Sub** silently drops to *connected* clients — (solved) heartbeat sequence numbers force a resync.
- **Per-user Inbox breaks with multi-device** — (solved) Inbox made per-client (`recipientClientId`) via Clients table.
- **Group transaction limit** — DynamoDB txn caps at 100 items = the 100-participant group cap; (solved/parked) batch near the limit.
- **Partition by user vs chat** — user is right for 1:1-heavy WhatsApp; (parked) large groups (>~25) may warrant chat-level channels.
- **CDN for media** — marginal given ≤100 recipients per attachment; (parked).
