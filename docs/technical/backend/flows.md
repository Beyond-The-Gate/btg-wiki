# Logical Flows

## Playtime accumulation

Playtime is accumulated on quit, computed entirely in the database so it's atomic and clock-consistent.

```mermaid
sequenceDiagram
    participant P as Proxy
    participant B as Backend
    participant D as PostgreSQL

    P->>B: POST join
    B->>D: upsert player, last_seen = now()
    Note over P,D: ...player is online...
    P->>B: POST quit
    B->>D: playtime += now() - last_seen, last_seen = now()
```

- **Online playtime** = stored `playtime` + (`now()` − `last_seen`).
- **Offline playtime** = stored `playtime`.

## Name reclaim on join

A player is the authoritative source for their current name. On join, the name is freed from any stale holder first.

```mermaid
flowchart TD
    a[Player joins with name N] --> b{Another player holds N?}
    b -->|Yes| c[Set that player's name to NULL]
    b -->|No| d[Continue]
    c --> e[Upsert joining player with N]
    d --> e
```

The whole thing runs in one transaction, so the unique constraint is never violated.

## Dungeon door & gate lifecycle

```mermaid
stateDiagram-v2
    [*] --> NoGate
    NoGate --> GateActive: open gate (set gate + modifiers, door open)
    GateActive --> GateActive: open / close door
    GateActive --> NoGate: complete gate
    note right of GateActive
        complete gate:
        record completion (count++ / first time),
        clear modifiers, gate = null, door closed
    end note
```

- **open gate** starts a run; **complete gate** finishes it and records it in `dungeon_completed_gate`.
- **open/close door** only toggle `door_open`, keeping the active gate and modifiers.

## Friend request lifecycle (with auto-accept)

```mermaid
flowchart TD
    a[A sends request to B by name] --> b{Already friends?}
    b -->|Yes| x[409 Conflict]
    b -->|No| c{B already requested A?}
    c -->|Yes| d[Delete B→A request, create friendship]
    d --> e[Publish accepted, return FRIENDED]
    c -->|No| f{A→B request exists?}
    f -->|Yes| y[409 Conflict]
    f -->|No| g[Create A→B request]
    g --> h[Publish sent, return REQUESTED]
```

Cancel (by sender), accept/deny (by receiver) and unfriend each delete the relevant row and publish their event — returning `404` if the row didn't exist.

## After-commit event publishing

```mermaid
sequenceDiagram
    participant S as Service
    participant T as Transaction
    participant R as Event Relay
    participant MQ as RabbitMQ

    S->>S: mutate DB + publishEvent(domain event)
    S->>T: commit
    T-->>R: AFTER_COMMIT fires
    R->>MQ: convertAndSend(btg.events, routingKey, event)
```

If the transaction rolls back, the relay never fires — so events are emitted only for changes that actually persisted.
