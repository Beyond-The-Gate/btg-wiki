# API Layer

## :material-information-outline: Conventions

| | |
|---|---|
| **Base URL** | `http://<host>:8080` |
| **Prefix** | all endpoints live under `/api/v1` |
| **Content type** | `application/json` (request & response) |
| **Timestamps** | ISO-8601, UTC — e.g. `2026-06-28T10:57:48.855Z` |
| **Durations** | milliseconds (long) |
| **IDs** | Players & dungeons use UUIDs; collections, gates, rooms, citizens, modifiers use string keys |
| **`target`** | where noted, a path/body field that accepts **either** a player UUID **or** a current name |

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
    | `401` | Missing or invalid credentials |
    | `403` | Authenticated, but not allowed |
    | `404` | Target not found |
    | `409` | Conflict (duplicate / already in desired state) |
    | `429` | Rate limit exceeded |

!!! info "Events"
    Many mutations publish messages to the `btg.events` exchange. See the dedicated **[Events](events.md)** page.

## :material-shield-key: Authentication & authorization

Two mechanisms behind one stateless filter chain:

| Caller | Header | Authority |
|---|---|---|
| MC servers | `X-Api-Key: <key>` | `ROLE_SERVICE` |
| Web players | `Authorization: Bearer <jwt>` | `ROLE_PLAYER` |

- **Public** (no auth): all `/api/v1/auth/**` and `/actuator/health`.
- **Default:** every other endpoint requires `ROLE_SERVICE`.
- **Exceptions** are marked inline as :material-account-check: **service or self** — a service, **or** the player whose token subject equals the path `{playerUuid}`.
- Over the rate limit → `429`. Services are exempt; players are limited per UUID; public auth routes per IP.

---

## :material-key: Web Authentication

Public endpoints under `/api/v1/auth/**` that drive the [website login flow](auth.md). On success, token-returning endpoints respond with an [`AuthTokenResponse`](#authtokenresponse) (JWT + player identity).

### Identify

```http
POST /api/v1/auth/identify
```

Step 1. Tells the frontend whether to log in or register.

=== "Request"

    ```json
    { "username": "Steve" }
    ```

=== "Response `200`"

    [`IdentifyResponse`](#identifyresponse) — `LOGIN` (an ACTIVE account exists) or `REGISTER` (no account, or PENDING)

    ```json
    { "state": "REGISTER" }
    ```

=== "Errors"

    `404` no player matches the username

---

### Login

```http
POST /api/v1/auth/login
```

=== "Request"

    ```json
    { "username": "Steve", "password": "…" }
    ```

=== "Response `200`"

    [`AuthTokenResponse`](#authtokenresponse)

=== "Errors"

    `401` invalid credentials · `404` unknown username

---

### Register

```http
POST /api/v1/auth/register
```

Sets a password (creating/overwriting a PENDING account) and sends an activation code in-game. Rejected if an ACTIVE account already exists.

=== "Request"

    ```json
    { "username": "Steve", "password": "…" }
    ```

=== "Response"

    `204` no content — emits `verification.code`

=== "Errors"

    `409` an account already exists · `404` unknown username

---

### Resend activation code

```http
POST /api/v1/auth/register/resend
```

Issues a fresh activation code (invalidating the previous one). Requires a PENDING account.

=== "Request"

    ```json
    { "username": "Steve" }
    ```

=== "Response"

    `204` no content — emits `verification.code`

=== "Errors"

    `404` no pending registration

---

### Verify

```http
POST /api/v1/auth/verify
```

Step 3. Confirms the activation code, activates the account, and issues a token.

=== "Request"

    ```json
    { "username": "Steve", "code": "123456" }
    ```

=== "Response `200`"

    [`AuthTokenResponse`](#authtokenresponse)

=== "Errors"

    `400` invalid or expired code · `404` no pending registration

---

### Request password reset

```http
POST /api/v1/auth/reset/request
```

Sends a reset code in-game. Requires an ACTIVE account.

=== "Request"

    ```json
    { "username": "Steve" }
    ```

=== "Response"

    `204` no content — emits `verification.code`

=== "Errors"

    `404` no account to reset

---

### Confirm password reset

```http
POST /api/v1/auth/reset/confirm
```

Verifies the reset code, replaces the password, and issues a token.

=== "Request"

    ```json
    { "username": "Steve", "code": "123456", "newPassword": "…" }
    ```

=== "Response `200`"

    [`AuthTokenResponse`](#authtokenresponse)

=== "Errors"

    `400` invalid or expired code · `404` no account to reset

---

## :material-account: Players

### Join

```http
POST /api/v1/players/{uuid}/join
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
POST /api/v1/players/{uuid}/quit
```

Adds the elapsed session time to `playtime` and resets `last_seen`. The delta is computed in the database for clock-consistency.

**Responses:** `204` no content

---

### Get playtime

```http
GET /api/v1/players/{target}/playtime?online={bool}
```

Current playtime in **milliseconds**, resolved by UUID or current name.

| Param | In | Type | Description |
|---|---|---|---|
| `target` | path | string | player UUID or current name |
| `online` | query | boolean | `true` adds the live, in-progress session; `false` returns the stored value |

=== "Response `200`"

    [`PlaytimeResponse`](#playtimeresponse)

    ```json
    { "playtime": 977847 }
    ```

=== "Errors"

    `404` — no player matches the target

---

## :material-castle: Dungeons

### Get dungeon

```http
GET /api/v1/dungeons/{dungeonUuid}
```

Full dungeon aggregate by its own UUID.

**Responses:** `200` → [`DungeonDto`](#dungeondto) · `404` unknown dungeon

---

### Spawn (resolve on join)

```http
POST /api/v1/players/{uuid}/spawn
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
POST /api/v1/players/{uuid}/current-dungeon/{dungeonUuid}
```

Moves the player to an **existing** dungeon they may enter (owner or trusted), updating their current dungeon so a later rejoin returns them here. Never creates.

**Responses:** `200` → [`DungeonDto`](#dungeondto) · `403` not trusted · `404` unknown dungeon

---

### Mark world initialized

```http
POST /api/v1/dungeons/{dungeonUuid}/world-initialized
```

Paper callback after it has created and saved the dungeon's world. Idempotent; clears the `created` flag for future spawns.

**Responses:** `204` no content · `404` unknown dungeon

---

### List accessible dungeons

```http
GET /api/v1/players/{playerUuid}/dungeons
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
POST /api/v1/dungeons/{dungeonUuid}/members/{playerUuid}
```

Adds a trusted member. Returns the updated member UUID list.

**Responses:** `200` → `string[]` (member UUIDs) · `400` target is the owner · `409` already trusted · `404` unknown dungeon

---

### Untrust a player

```http
DELETE /api/v1/dungeons/{dungeonUuid}/members/{playerUuid}
```

Removes a trusted member. Returns the updated member UUID list.

**Responses:** `200` → `string[]` · `400` target is the owner · `404` unknown dungeon / not trusted

---

### Create a room

```http
POST /api/v1/dungeons/{dungeonUuid}/rooms
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
PATCH /api/v1/dungeons/{dungeonUuid}/rooms/{room}
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
POST /api/v1/dungeons/{dungeonUuid}/gate/open
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
PUT /api/v1/dungeons/{dungeonUuid}/gate/data
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
POST /api/v1/dungeons/{dungeonUuid}/gate/complete
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
POST /api/v1/dungeons/{dungeonUuid}/door/open
POST /api/v1/dungeons/{dungeonUuid}/door/close
```

Toggles the door only — gate and modifiers are **kept**.

**Responses:** `204` no content · `404` unknown dungeon

---

## :material-account-group: Friends

Mutations publish to the [`btg.events`](events.md#friends) exchange after the DB commit.

### Send a friend request

```http
POST /api/v1/players/{uuid}/friend-requests
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
GET /api/v1/players/{uuid}/friend-requests/outgoing
```

Requests you've sent.

**Responses:** `200` → array of [`FriendRequestDto`](#friendrequestdto) (the *receiver*)

---

### List incoming requests

```http
GET /api/v1/players/{uuid}/friend-requests/incoming
```

Requests you've received.

**Responses:** `200` → array of [`FriendRequestDto`](#friendrequestdto) (the *sender*)

---

### Cancel a request

```http
DELETE /api/v1/players/{uuid}/friend-requests/outgoing/{targetUuid}
```

Withdraws a request you sent.

**Responses:** `204` · `404` no such outgoing request — emits `friend.request.cancelled`

---

### Accept a request

```http
POST /api/v1/players/{uuid}/friend-requests/incoming/{senderUuid}/accept
```

Accepts an incoming request, creating the friendship.

**Responses:** `204` · `404` no such incoming request — emits `friend.request.accepted`

---

### Deny a request

```http
POST /api/v1/players/{uuid}/friend-requests/incoming/{senderUuid}/deny
```

Rejects an incoming request without befriending.

**Responses:** `204` · `404` no such incoming request — emits `friend.request.denied`

---

### List friends

```http
GET /api/v1/players/{uuid}/friends
```

**Responses:** `200` → array of [`FriendDto`](#frienddto) (uuid, name, `friendsSince`)

---

### Unfriend

```http
DELETE /api/v1/players/{uuid}/friends/{friendUuid}
```

Removes an existing friendship.

**Responses:** `204` · `404` not friends — emits `friendship.removed`

---

## :material-gavel: Moderation

Five actions (ban, mute, kick, unban, unmute) plus active/history queries. **Every** action is appended to the moderation log; **ban/mute** additionally upsert a single active punishment per type (a new one overrides any existing one, active or not). Every successful action publishes an event — see [Events › Moderation](events.md#moderation).

!!! note "Common fields"
    - `target` — player UUID or current name.
    - `moderatorUuid` — the acting moderator, or `null` for **console/system**.
    - `durationMillis` — ban/mute length in ms, or `null` for **permanent**. `expires_at` is computed in the DB so `endsAt − startsAt` equals the duration exactly.

### Ban / Mute

```http
POST /api/v1/moderation/ban
POST /api/v1/moderation/mute
```

Issues (or overrides) the punishment and logs it.

=== "Request"

    [`PunishRequest`](#punishrequest)

    ```json
    { "target": "Steve", "moderatorUuid": "…", "reason": "griefing", "durationMillis": 86400000 }
    ```

=== "Response `200`"

    [`PunishmentResponse`](#punishmentresponse)

    ```json
    { "type": "BAN", "reason": "griefing", "startsAt": "…", "endsAt": "…" }
    ```

=== "Errors"

    `404` unknown target

→ emits `moderation.ban` / `moderation.mute`.

---

### Unban / Unmute

```http
POST /api/v1/moderation/unban
POST /api/v1/moderation/unmute
```

Removes the **active** punishment and logs the lift. Fails if the player isn't currently banned/muted (a leftover expired row doesn't count).

=== "Request"

    [`LiftRequest`](#liftrequest)

    ```json
    { "target": "Steve", "moderatorUuid": "…" }
    ```

=== "Response"

    `204` no content

=== "Errors"

    `404` unknown target / not currently banned (muted)

→ emits `moderation.unban` / `moderation.unmute`.

---

### Kick

```http
POST /api/v1/moderation/kick
```

Logs a kick. Creates no lasting state (the proxy disconnects the player inline).

=== "Request"

    [`KickRequest`](#kickrequest)

    ```json
    { "target": "Steve", "moderatorUuid": "…", "reason": "afk" }
    ```

    `reason` is optional.

=== "Response"

    `204` no content

=== "Errors"

    `404` unknown target

→ emits `moderation.kick`.

---

### Active punishments

```http
GET /api/v1/moderation/{target}/active
```

The player's currently active (non-expired) punishments — 0 to 2 entries.

**Responses:** `200` → array of [`ActivePunishmentDto`](#activepunishmentdto) · `404` unknown target

---

### History

```http
GET /api/v1/moderation/{target}/history?actions={A,B,…}
```

Full moderation log for the player, newest first, with the moderator's current name.

| Param | In | Type | Description |
|---|---|---|---|
| `target` | path | string | player UUID or current name |
| `actions` | query | enum[] | optional filter; repeat or comma-separate (e.g. `?actions=BAN,KICK`) |

**Responses:** `200` → array of [`ModerationEntryDto`](#moderationentrydto) · `404` unknown target

---

## :material-trophy: Collections

Player progression against the static collection catalog. A progress row exists only once tracked (amount ≠ 0).

### Summary

:material-account-check: **service or self**

```http
GET /api/v1/players/{playerUuid}/collections/summary
```

Global discovered-vs-total counts.

=== "Response `200`"

    [`CollectionSummaryDto`](#collectionsummarydto)

    ```json
    { "discovered": 1, "total": 3 }
    ```

---

### List categories

:material-account-check: **service or self**

```http
GET /api/v1/players/{playerUuid}/collections/categories
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

:material-account-check: **service or self**

```http
GET /api/v1/players/{playerUuid}/collections/categories/{categoryId}
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
POST /api/v1/players/{playerUuid}/collections
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

## :material-code-braces: Schemas

### IdentifyResponse
| Field | Type | Notes |
|---|---|---|
| `state` | enum | `LOGIN` \| `REGISTER` |

### AuthTokenResponse
| Field | Type | Notes |
|---|---|---|
| `token` | string | signed JWT (`sub`=uuid, `name`, `role=PLAYER`) |
| `expiresAt` | timestamp | token expiry |
| `playerUuid` | UUID | |
| `playerName` | string? | may be temporarily unknown |

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

### PunishRequest
| Field | Type | Notes |
|---|---|---|
| `target` | string | UUID or current name |
| `moderatorUuid` | UUID? | `null` = console/system |
| `reason` | string | |
| `durationMillis` | long? | `null` = permanent |

### LiftRequest
| Field | Type | Notes |
|---|---|---|
| `target` | string | UUID or current name |
| `moderatorUuid` | UUID? | `null` = console/system |

### KickRequest
| Field | Type | Notes |
|---|---|---|
| `target` | string | UUID or current name |
| `moderatorUuid` | UUID? | `null` = console/system |
| `reason` | string? | optional |

### PunishmentResponse
| Field | Type | Notes |
|---|---|---|
| `type` | enum | `BAN` \| `MUTE` |
| `reason` | string | |
| `startsAt` | timestamp | when the punishment began |
| `endsAt` | timestamp? | `null` = permanent |

### ActivePunishmentDto
| Field | Type | Notes |
|---|---|---|
| `type` | enum | `BAN` \| `MUTE` |
| `reason` | string | |
| `expiresAt` | timestamp? | `null` = permanent |

### ModerationEntryDto
| Field | Type | Notes |
|---|---|---|
| `id` | long | |
| `action` | enum | `BAN` \| `MUTE` \| `UNBAN` \| `UNMUTE` \| `KICK` |
| `reason` | string? | |
| `moderatorUuid` | UUID? | `null` = console/system |
| `moderatorName` | string? | `null` = console/system or unknown |
| `expiresAt` | timestamp? | for ban/mute entries; `null` = permanent / n/a |
| `createdAt` | timestamp | when the action happened |

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
