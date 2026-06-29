# Backend

The central service of Beyond the Gate — the single source of truth for player identity, dungeons, social features, progression and moderation. Source: [`btg-backend`](https://github.com/Beyond-The-Gate/btg-backend).

It is consumed under `/api/v1/**` by the **Velocity proxy** and **Paper servers** (as trusted services) and the **website** (as authenticated players), plus an event stream over RabbitMQ.

## :material-book-open-page-variant: Read on:

- **[Architecture](architecture.md)** — stack, modules, layers, security, conventions
- **[Data Model](data-model.md)** — diagram
- **[API Layer](api.md)** — endpoints, auth, error model, schemas
- **[Events](events.md)** — exchange, routing keys, payloads
- **[Web Login](auth.md)** — login and verification flow
