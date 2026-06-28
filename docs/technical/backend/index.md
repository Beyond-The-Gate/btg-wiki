# Backend

The central service of Beyond the Gate — the single source of truth for player identity, dungeons, social features, progression and moderation. Source: [`btg-backend`](https://github.com/Beyond-The-Gate/btg-backend).

It is consumed by the **Velocity proxy** and **Paper servers** over `/game/**` and events, and the **website** over `/web/**`.

## :material-book-open-page-variant: Read on:

- **[Architecture](architecture.md)** — stack, modules, layers, conventions
- **[Data Model](data-model.md)** — diagram
- **[API Layer](api.md)** — endpoints, error model, schemas
- **[Events](events.md)** — exchange, routing keys, payloads
- **[Web Login](auth.md)** — login and verification flow
