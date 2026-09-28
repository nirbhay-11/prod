# Sync

**A real-time collaborative text editor built to scale horizontally.**

Sync lets multiple users edit the same document at the same time and see each other's changes live. It uses a CRDT (Yjs) for conflict-free merging, WebSockets for transport, Redis Pub/Sub to fan out edits across server instances, and Postgres for durable storage.

> The interesting part isn't the editor UI. It's keeping state consistent when two people type in the same spot at the same time, while connected to *different* servers.

---

## Features

- Live multi-user editing with sub-200ms sync
- Conflict-free concurrent edits (CRDT, no "last write wins" data loss)
- Presence: see who is online and where their cursor is
- Offline tolerance: keep typing with no network, merge cleanly on reconnect
- Durable documents via snapshots + update log (survives restarts)
- Horizontal scaling: any client can connect to any server instance
- Auth and role-based access (owner / editor / viewer)

---

## Architecture

```
Client (WebSocket) ──┐
Client (WebSocket) ──┼──► Load Balancer
Client (WebSocket) ──┘         │
                        ┌──────┼───────┐
                        ▼      ▼       ▼
                   API Server  API Server  API Server
                        │      │       │
                        └──────┼───────┘
                               ▼
                      Redis Pub/Sub  (fan-out + presence)
                               │
                               ▼
                    Postgres (snapshots + updates)
```

### Request flow

1. Client opens a document and connects over WebSocket. The server sends the latest snapshot plus any updates since.
2. Client edits. Yjs produces a small binary diff, sent to the server.
3. Server publishes the diff to the Redis channel `doc:{id}:updates`.
4. Every server instance subscribed to that channel forwards the diff to its own connected clients.
5. The diff is persisted to `document_updates` asynchronously, off the broadcast path.
6. A background worker compacts recent updates into a new `document_snapshots` row.

---

## Tech stack

| Layer | Choice |
|---|---|
| Runtime | Node.js / Bun + TypeScript |
| Transport | WebSockets (`ws`) |
| Conflict resolution | Yjs (CRDT) |
| Fan-out / presence | Redis (Pub/Sub + hashes with TTL) |
| Database | Postgres |
| Auth | JWT |
| Frontend | React + a Yjs-compatible editor binding (e.g. TipTap or CodeMirror) |
| Infra | Docker, Docker Compose |

---

## Key design decisions

### Why a CRDT (Yjs) instead of Operational Transformation?
OT needs a central server to order and transform operations, and is notoriously hard to implement correctly. CRDTs converge regardless of the order updates arrive in, which makes offline editing and multi-server fan-out straightforward. I chose Yjs over writing my own because it is battle-tested, and the goal is a correct, scalable system, not a novel algorithm.

### Why Redis Pub/Sub?
User A on Server 1 and User B on Server 2 would otherwise never see each other's edits. Pub/Sub decouples servers so any instance can serve any client, with no sticky sessions required. Tradeoff: Pub/Sub is fire-and-forget. A server that is briefly disconnected can miss messages, which is why clients resync via Yjs state vectors on reconnect.

### Why snapshots plus an update log?
Replaying every update since a document's creation makes loads progressively slower. Snapshots bound load time: load the latest snapshot, replay only the updates after it. Compaction runs every N updates or every few minutes.

### Why keep presence out of Postgres?
Cursor positions are ephemeral and high-frequency and need no durability. Redis hashes with a short TTL, refreshed by client heartbeats, are the right fit.

### Reconnect strategy
On reconnect the client sends its Yjs state vector. The server diffs it against its own state and returns only what the client is missing, not the whole document.

---

## Database schema

```sql
CREATE TABLE users (
    id            BIGSERIAL PRIMARY KEY,
    email         VARCHAR(255) UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,
    display_name  VARCHAR(100) NOT NULL,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE documents (
    id            BIGSERIAL PRIMARY KEY,
    title         VARCHAR(255) NOT NULL DEFAULT 'Untitled',
    owner_id      BIGINT NOT NULL REFERENCES users(id),
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE document_collaborators (
    document_id   BIGINT NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
    user_id       BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role          VARCHAR(20) NOT NULL DEFAULT 'editor', -- owner | editor | viewer
    added_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (document_id, user_id)
);

CREATE TABLE document_snapshots (
    id            BIGSERIAL PRIMARY KEY,
    document_id   BIGINT NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
    state_vector  BYTEA NOT NULL,   -- Yjs binary state
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_snapshots_document_id ON document_snapshots(document_id, created_at DESC);

CREATE TABLE document_updates (
    id            BIGSERIAL PRIMARY KEY,
    document_id   BIGINT NOT NULL REFERENCES documents(id) ON DELETE CASCADE,
    user_id       BIGINT REFERENCES users(id),
    update_data   BYTEA NOT NULL,   -- Yjs update diff
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_updates_document_id ON document_updates(document_id, created_at);
```

---

## Redis keys and channels

| Key / Channel | Purpose |
|---|---|
| `doc:{id}:updates` | Pub/Sub channel for document diffs |
| `doc:{id}:presence` | Pub/Sub channel for presence changes |
| `presence:{docId}` | Hash of `userId → {cursor, lastSeen}` with TTL |

---

## Getting started

### Prerequisites
- Node.js 20+ (or Bun)
- Docker and Docker Compose

### Run locally

```bash
git clone https://github.com/<your-username>/sync.git
cd sync
cp .env.example .env

# start Postgres, Redis, and two server instances
docker compose up --build

# run migrations
npm run migrate

# start the frontend
cd client && npm install && npm run dev
```

Open `http://localhost:3000` in two browser windows to see live sync. Compose runs two server instances behind a load balancer, so the two windows will often be on different servers.

### Environment variables

```env
DATABASE_URL=postgres://sync:sync@localhost:5432/sync
REDIS_URL=redis://localhost:6379
JWT_SECRET=change-me
PORT=4000
SNAPSHOT_EVERY_N_UPDATES=100
```

---

## Project structure

```
sync/
├── server/
│   ├── src/
│   │   ├── ws/            # WebSocket handlers, Yjs sync protocol
│   │   ├── pubsub/        # Redis fan-out + presence
│   │   ├── routes/        # REST: auth, documents, collaborators
│   │   ├── workers/       # snapshot compaction
│   │   └── db/            # migrations, queries
│   └── Dockerfile
├── client/                # React editor
├── docker-compose.yml
└── README.md
```

---

## API overview

### REST

| Method | Endpoint | Description |
|---|---|---|
| POST | `/auth/register` | Create account |
| POST | `/auth/login` | Get JWT |
| POST | `/documents` | Create document |
| GET | `/documents` | List documents you can access |
| GET | `/documents/:id` | Document metadata |
| POST | `/documents/:id/collaborators` | Share with a user and role |

### WebSocket

Connect to `/ws/documents/:id?token=<jwt>`. Messages are binary Yjs sync and awareness frames.

---

## Testing the interesting parts

- **Concurrent edits:** two tabs type in the same paragraph at once and both converge to the same text.
- **Cross-server sync:** run two instances, connect one client to each, and confirm edits propagate.
- **Offline merge:** disable the network in DevTools, keep typing, reconnect, and confirm a clean merge.
- **Restart durability:** kill all servers, restart, and confirm the document loads intact.

---

## Roadmap

- [x] Single-server WebSocket sync with Yjs
- [x] Postgres persistence (snapshots + updates)
- [x] Redis Pub/Sub for multi-server fan-out
- [ ] Auth and role-based access control
- [ ] Presence and live cursors
- [ ] Version history / restore
- [ ] Load test results (target: N concurrent connections per instance)

---

## What would break at 10x scale

- **Hot documents:** thousands of editors on one doc make one Pub/Sub channel a bottleneck. Mitigation: shard by document and consider Redis Streams or Kafka.
- **Update log growth:** compact more aggressively and archive old updates to object storage.
- **Connection limits:** tune per-instance limits and autoscale on open-connection count.
- **Pub/Sub message loss:** move to a log-based broker if stronger delivery guarantees are needed.

---

## License

MIT
