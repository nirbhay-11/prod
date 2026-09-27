
Client (WebSocket) ──┐
Client (WebSocket) ──┼──► Load Balancer (sticky sessions NOT required if done right)
Client (WebSocket) ──┘         │
                        ┌───────┼────────┐
                        ▼       ▼        ▼
                   API Server API Server API Server
                        │       │        │
                        └───────┼────────┘
                                ▼
                       Redis Pub/Sub (fan-out between servers)
                                │
                                ▼
                           Postgres (durable state + snapshots)
