# API Layer

!!! warning "Authentication is not yet enforced"
    A temporary permit-all configuration is active in development. Do **not** deploy publicly until role-based security (`ROLE_SERVICE` / `ROLE_PLAYER`) is in place.

## :material-information-outline: Conventions

| | |
|---|---|
| **Base URL** | `http://<host>:8080` |
| **Content type** | `application/json` (request & response) |
| **Timestamps** | ISO-8601, UTC — e.g. `2026-06-28T10:57:48.855Z` |
| **IDs** | Players & dungeons use UUIDs; collections, gates, rooms, citizens, modifiers use string keys |

??? abstract "Error format & status codes"
    Every error shares one shape:

    ```json
    { "status": 404, "message": "Player '...' not found" }
    ```

    | Code | Meaning |
    |---|---|
    | `200` | OK, body returned |
    | `204` | OK, no body |
    | `400` | Invalid input / business rule violated |
    | `403` | Forbidden (not allowed to access the target) |
    | `404` | Target not found |
    | `409` | Conflict (duplicate / already in desired state) |

---

## :material-account: Players

### Join

```http
POST /game/players/{uuid}/join
```

Upserts the player on connect: creates them on first join, otherwise refreshes name + `last_seen`, reclaiming the name from any stale holder. Returns the player and any active punishments.

| Param | In | Type | Description |
|---|---|---|---|
| `uuid` | path | UUID | Minecraft player UUID |

=== "Request"

    ```json
    { "name": "Steve" }
    ```

=== "Response `200`"

    [`PlayerJoinResponse`](#playerjoinresponse)

    ```json
    {
      "player": {
        "uuid": "…", "name": "Steve",
        "firstSeen": "…", "lastSeen": "…", "playtime": 0
      },
      "activePunishments": [
        { "type": "BAN", "reason": "…", "expiresAt": null }
      ]
    }
    ```

---

### Quit

```http
POST /game/players/{uuid}/quit
```

Adds the elapsed session time to `playtime` and resets `last_seen`. The delta is computed in the database for clock-consistency.

**Responses:** `204` no content

---

### Get playtime by UUID

```http
GET /game/players/{uuid}/playtime?online={bool}
```

Current playtime in **milliseconds**.

| Param | In | Type | Description |
|---|---|---|---|
| `uuid` | path | UUID | Player UUID |
| `online` | query | boolean | `true` adds the live, in-progress session; `false` returns the stored value |

=== "Response `200`"

    [`PlaytimeResponse`](#playtimeresponse)

    ```json
    { "playtime": 977847 }
    ```

=== "Errors"

    `404` — unknown player

---

### Get playtime by name

```http
GET /game/players/by-name/{name}/playtime?online={bool}
```

Same as above, resolved by current name.

**Responses:** `200` → [`PlaytimeResponse`](#playtimeresponse) · `404` no player holds that name

---

## :material-castle: Dungeons

### Get dungeon

```http
GET /game/dungeons/{dungeonUuid}
```

Full dungeon aggregate by its own UUID.

**Responses:** `200` → [`DungeonDto`](#dungeondto) · `404` unknown dungeon

---

### Spawn (resolve on join)

```http
POST /game/players/{uuid}/spawn
```

Resolves where to send the player on join. Returns their **current** dungeon if it's still a valid destination (it exists and they're owner or trusted), otherwise falls back to their **own** dungeon — created and seeded with a default `main` room on the first ever join. Never resolves to nothing.

`created` tells Paper whether it must build and save the world; it stays `true` on every join until Paper confirms via [world-initialized](#mark-world-initialized).

=== "Response `200`"

    [`SpawnResponse`](#spawnresponse)

    ```json
    {
      "dungeon": {
        "uuid": "…", "ownerUuid": "…", "doorOpen": false,
        "gate": null, "gateData": null,
        "gateModifiers": [], "members": [],
        "rooms": [ { "name": "main", "level": 1 } ],
        "citizens": [], "completedGates": []
      },
      "created": true,
      "reason": "NOT_EXISTENT"
    }
    ```

    `reason` — `CURRENT` (sent to current dungeon) · `NOT_EXISTENT` (no/deleted current → sent home) · `NOT_TRUSTED` (lost access to current → sent home).

---

### Travel to a dungeon

```http
POST /game/players/{uuid}/current-dungeon/{dungeonUuid}
```

Moves the player to an **existing** dungeon they may enter (owner or trusted), updating their current dungeon so a later rejoin returns them here. Never creates.

**Responses:** `200` → [`DungeonDto`](#dungeondto) · `403` not trusted · `404` unknown dungeon

---

### Mark world initialized

```http
POST /game/dungeons/{dungeonUuid}/world-initialized
```

Paper callback after it has created and saved the dungeon's world. Idempotent; clears the `created` flag for future spawns.

**Responses:** `204` no content · `404` unknown dungeon

---

### List accessible dungeons

```http
GET /game/players/{playerUuid}/dungeons
```

Summaries of every dungeon the player **owns or is trusted in** (includes their own). Never creates.

=== "Response `200`"

    array of [`DungeonSummaryDto`](#dungeonsummarydto)

    ```json
    [ { "uuid": "…", "ownerUuid": "…", "ownerName": "Steve" } ]
    ```

---

### Trust a player

```http
POST /game/dungeons/{dungeonUuid}/members/{playerUuid}
```

Adds a trusted member. Returns the updated member UUID list.

**Responses:** `200` → `string[]` (member UUIDs) · `400` target is the owner · `409` already trusted · `404` unknown dungeon

---

### Untrust a player

```http
DELETE /game/dungeons/{dungeonUuid}/members/{playerUuid}
```

Removes a trusted member. Returns the updated member UUID list.

**Responses:** `200` → `string[]` · `400` target is the owner · `404` unknown dungeon / not trusted

---

### Create a room

```http
POST /game/dungeons/{dungeonUuid}/rooms
```

Creates a room at **level 1**. Room names are unique per dungeon.

=== "Request"

    ```json
    { "name": "vault" }
    ```

=== "Response `200`"

    [`DungeonRoomDto`](#dungeonroomdto)

    ```json
    { "name": "vault", "level": 1 }
    ```

=== "Errors"

    `409` name already exists · `404` unknown dungeon

---

### Update a room

```http
PATCH /game/dungeons/{dungeonUuid}/rooms/{room}
```

Updates a room (e.g. level-up). The room **name is immutable**.

=== "Request"

    ```json
    { "level": 5 }
    ```

=== "Response `200`"

    [`DungeonRoomDto`](#dungeonroomdto)

=== "Errors"

    `404` unknown room

---

### Open a gate

```http
POST /game/dungeons/{dungeonUuid}/gate/open
```

Starts a gate run: opens the door, sets the gate, applies its modifiers, and sets the initial gate data (a fresh gate starts with no data if omitted).

=== "Request"

    ```json
    { "gate": "fire", "modifiers": ["hardcore", "timed"], "gateData": "{\"phase\":1}" }
    ```

    `gateData` is optional, opaque raw JSON.

=== "Response"

    `204` no content

=== "Errors"

    `400` invalid gate data JSON · `404` unknown dungeon

---

### Set gate data

```http
PUT /game/dungeons/{dungeonUuid}/gate/data
```

Replaces the opaque gate JSON during a run. The body is raw JSON, stored as-is and returned verbatim on the dungeon aggregate. Cleared automatically when the gate is completed/cleared.

=== "Request"

    ```json
    { "phase": 2, "bossHp": 1500 }
    ```

=== "Response"

    `204` no content

=== "Errors"

    `400` invalid JSON · `404` unknown dungeon

---

### Complete a gate

```http
POST /game/dungeons/{dungeonUuid}/gate/complete
```

Marks the active gate complete: records the completion (increments count, preserves first-completed time), then clears the gate, removes modifiers, and closes the door.

=== "Response `200`"

    [`DungeonCompletedGateDto`](#dungeoncompletedgatedto)

    ```json
    { "gate": "fire", "completionCount": 2, "firstCompletedAt": "…" }
    ```

=== "Errors"

    `400` no active gate · `404` unknown dungeon

---

### Open / close the door

```http
POST /game/dungeons/{dungeonUuid}/door/open
POST /game/dungeons/{dungeonUuid}/door/close
```

Toggles the door only — gate and modifiers are **kept**.

**Responses:** `204` no content · `404` unknown dungeon

---

## :material-account-group: Friends

!!! info "Events"
    Every mutation publishes a JSON event to the `btg.events` topic exchange **after the DB commit**, so other services can update both parties' menus. See [Events](#events).

### Send a friend request

```http
POST /game/players/{uuid}/friend-requests
```

Sends a request **by name**. If the target already has a pending request to you, this **auto-accepts** into a friendship.

=== "Request"

    ```json
    { "targetName": "Alex" }
    ```

=== "Response `200`"

    [`SendFriendResponse`](#sendfriendresponse) — `REQUESTED`, or `FRIENDED` on mutual auto-accept

    ```json
    { "result": "REQUESTED" }
    ```

=== "Errors"

    `400` befriending yourself · `404` unknown name · `409` already friends / request already sent

→ emits `friend.request.sent`, or `friend.request.accepted` on auto-accept.

---

### List outgoing requests

```http
GET /game/players/{uuid}/friend-requests/outgoing
```

Requests you've sent.

**Responses:** `200` → array of [`FriendRequestDto`](#friendrequestdto) (the *receiver*)

---

### List incoming requests

```http
GET /game/players/{uuid}/friend-requests/incoming
```

Requests you've received.

**Responses:** `200` → array of [`FriendRequestDto`](#friendrequestdto) (the *sender*)

---

### Cancel a request

```http
DELETE /game/players/{uuid}/friend-requests/outgoing/{targetUuid}
```

Withdraws a request you sent.

**Responses:** `204` · `404` no such outgoing request — emits `friend.request.cancelled`

---

### Accept a request

```http
POST /game/players/{uuid}/friend-requests/incoming/{senderUuid}/accept
```

Accepts an incoming request, creating the friendship.

**Responses:** `204` · `404` no such incoming request — emits `friend.request.accepted`

---

### Deny a request

```http
POST /game/players/{uuid}/friend-requests/incoming/{senderUuid}/deny
```

Rejects an incoming request without befriending.

**Responses:** `204` · `404` no such incoming request — emits `friend.request.denied`

---

### List friends

```http
GET /game/players/{uuid}/friends
```

**Responses:** `200` → array of [`FriendDto`](#frienddto) (uuid, name, `friendsSince`)

---

### Unfriend

```http
DELETE /game/players/{uuid}/friends/{friendUuid}
```

Removes an existing friendship.

**Responses:** `204` · `404` not friends — emits `friendship.removed`

---

## :material-trophy: Collections

Player progression against the static [collection catalog](../data-model/#collections). A progress row exists only once tracked (amount ≠ 0).

### Summary

```http
GET /game/players/{playerUuid}/collections/summary
```

Global discovered-vs-total counts.

=== "Response `200`"

    [`CollectionSummaryDto`](#collectionsummarydto)

    ```json
    { "discovered": 1, "total": 3 }
    ```

---

### List categories

```http
GET /game/players/{playerUuid}/collections/categories
```

All categories with per-category totals and the player's discovered count, ordered by `display_order`.

=== "Response `200`"

    array of [`CollectionCategoryDto`](#collectioncategorydto)

    ```json
    [ { "id": "resource", "name": "Resources", "description": "…",
        "icon": "…", "total": 2, "discovered": 1 } ]
    ```

---

### List collections in a category

```http
GET /game/players/{playerUuid}/collections/categories/{categoryId}
```

Every collection in the category with the player's `amount` (0 if untracked) and its levels, ordered by `display_order`.

=== "Response `200`"

    array of [`CollectionDto`](#collectiondto)

    ```json
    [ {
      "key": "resource.coal", "name": "Coal", "description": "…", "icon": "…",
      "amount": 12,
      "levels": [ { "level": 1, "requiredAmount": 10, "reward": "…" } ]
    } ]
    ```

=== "Errors"

    `404` unknown category

---

### Track progress

```http
POST /game/players/{playerUuid}/collections
```

Adds an amount to a collection (creating the row on first track). The key is validated against the catalog. Returns the new total.

=== "Request"

    ```json
    { "key": "resource.coal", "amount": 3 }
    ```

=== "Response `200`"

    [`TrackCollectionResponse`](#trackcollectionresponse)

    ```json
    { "amount": 8 }
    ```

=== "Errors"

    `400` amount ≤ 0 · `404` unknown collection key

---

## :material-rabbit: Events

Friend mutations publish to the **`btg.events`** topic exchange. Payloads are JSON; each carries both players (uuid + name) so consumers can render without a lookup.

| Routing key | Payload |
|---|---|
| `friend.request.sent` | sender, receiver |
| `friend.request.cancelled` | sender, receiver |
| `friend.request.accepted` | sender, receiver, `friendsSince` |
| `friend.request.denied` | sender, receiver |
| `friendship.removed` | remover, removed |

---

## :material-code-braces: Schemas

### PlayerJoinResponse
| Field | Type | Notes |
|---|---|---|
| `player` | [PlayerDto](#playerdto) | |
| `activePunishments` | [ActivePunishmentDto](#activepunishmentdto)[] | empty if none |

### PlayerDto
| Field | Type | Notes |
|---|---|---|
| `uuid` | UUID | |
| `name` | string | |
| `firstSeen` | timestamp | |
| `lastSeen` | timestamp | |
| `playtime` | long | milliseconds |

### ActivePunishmentDto
| Field | Type | Notes |
|---|---|---|
| `type` | enum | `BAN` \| `MUTE` |
| `reason` | string | |
| `expiresAt` | timestamp? | `null` = permanent |

### PlaytimeResponse
| Field | Type | Notes |
|---|---|---|
| `playtime` | long | milliseconds |

### DungeonDto
| Field | Type | Notes |
|---|---|---|
| `uuid` | UUID | |
| `ownerUuid` | UUID | |
| `doorOpen` | boolean | |
| `gate` | string? | `null` when no active gate |
| `gateData` | string? | opaque gate JSON; `null` when none |
| `gateModifiers` | string[] | set |
| `members` | UUID[] | trusted players |
| `rooms` | [DungeonRoomDto](#dungeonroomdto)[] | |
| `citizens` | [DungeonCitizenDto](#dungeoncitizendto)[] | |
| `completedGates` | [DungeonCompletedGateDto](#dungeoncompletedgatedto)[] | |

### DungeonSummaryDto
| Field | Type | Notes |
|---|---|---|
| `uuid` | UUID | |
| `ownerUuid` | UUID | |
| `ownerName` | string? | may be temporarily unknown |

### SpawnResponse
| Field | Type | Notes |
|---|---|---|
| `dungeon` | [DungeonDto](#dungeondto) | the resolved target |
| `created` | boolean | Paper must build + save the world |
| `reason` | enum | `CURRENT` \| `NOT_EXISTENT` \| `NOT_TRUSTED` |

### DungeonRoomDto
| Field | Type | Notes |
|---|---|---|
| `name` | string | immutable |
| `level` | int | defaults to 1 |

### DungeonCitizenDto
| Field | Type | Notes |
|---|---|---|
| `name` | string | |

### DungeonCompletedGateDto
| Field | Type | Notes |
|---|---|---|
| `gate` | string | |
| `completionCount` | long | |
| `firstCompletedAt` | timestamp | preserved across re-completions |

### SendFriendResponse
| Field | Type | Notes |
|---|---|---|
| `result` | enum | `REQUESTED` \| `FRIENDED` |

### FriendRequestDto
| Field | Type | Notes |
|---|---|---|
| `playerUuid` | UUID | the other party |
| `playerName` | string? | may be temporarily unknown |

### FriendDto
| Field | Type | Notes |
|---|---|---|
| `uuid` | UUID | |
| `name` | string? | may be temporarily unknown |
| `friendsSince` | timestamp | |

### CollectionSummaryDto
| Field | Type | Notes |
|---|---|---|
| `discovered` | int | collections with amount ≠ 0 |
| `total` | int | total valid collections |

### CollectionCategoryDto
| Field | Type | Notes |
|---|---|---|
| `id` | string | |
| `name` | string | |
| `description` | string | |
| `icon` | string | |
| `total` | int | collections in category |
| `discovered` | int | discovered by player |

### CollectionDto
| Field | Type | Notes |
|---|---|---|
| `key` | string | |
| `name` | string | |
| `description` | string | |
| `icon` | string | |
| `amount` | long | 0 if untracked |
| `levels` | [CollectionLevelDto](#collectionleveldto)[] | 0+ |

### CollectionLevelDto
| Field | Type | Notes |
|---|---|---|
| `level` | int | |
| `requiredAmount` | long | amount to unlock |
| `reward` | string | reward description |

### TrackCollectionResponse
| Field | Type | Notes |
|---|---|---|
| `amount` | long | new total after tracking |
