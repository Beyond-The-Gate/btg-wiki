# Backend

The central service of Beyond the Gate — the single source of truth for player identity, dungeons, social features, progression and moderation. Source: [`btg-backend`](https://github.com/Beyond-The-Gate/btg-backend).

It is consumed by the **Velocity proxy** and **Paper servers** over `/game/**`, and (in future) the **website** over `/web/**`.

## Responsibilities

- Player identity & sessions (join/quit, playtime)
- Dungeons (rooms, gates, trusted members, completion tracking)
- Friends (requests, friendships, live events)
- Moderation reads (active punishments) — write side planned
- Publishing domain events to RabbitMQ for the rest of the network

## At a glance

| Area | Choice |
|---|---|
| Language | Kotlin (JDK 25 toolchain) |
| Framework | Spring Boot 4.1 |
| Data access | jOOQ (schema-first) |
| Schema | Flyway + PostgreSQL 18 |
| Messaging | RabbitMQ (topic exchange `btg.events`) |
| Build | Gradle (Kotlin DSL), multi-module |

Read on:

- **[Architecture](architecture.md)** — stack, modules, layers, conventions
- **[Data Model](data-model.md)** — tables, keys, design decisions
- **[API Layer](api.md)** — endpoints, error model, events
