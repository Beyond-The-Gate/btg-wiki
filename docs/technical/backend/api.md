# API Layer

Reference for the backend's HTTP API. All endpoints are on the **`/game/**`** surface (Minecraft servers).

## Conventions

- **Content type:** `application/json` for requests and responses.
- **Acting player:** identified by `{uuid}` in the path (becomes the JWT subject on `/web/**` later).
- **Timestamps:** ISO-8601 in UTC (e.g. `2026-06-28T10:00:00Z`).
- **UUIDs:** canonical string form (e.g. `00000000-0000-0000-0000-000000000001`).
- **Names are nullable** everywhere they appear (cached Minecraft names can be temporarily unknown).

### Error model

Errors return an [`ApiError`](#apierror) body with the matching status:

| Status | Thrown when |
|---|---|
| `400 Bad Request` | invalid input (e.g. friending yourself) |
| `404 Not Found` | the target entity/row doesn't exist |
| `409 Conflict` | state conflict (e.g. already friends) |

```json
{ "status": 404, "message": "Player 00000000-0000-0000-0000-000000000001 not found" }
```

---

## Players

### Register on join

`POST /game/players/{uuid}/join`

Creates the player on first join, or refreshes their name and last-seen on subsequent joins. Returns the profile plus any active punishments the proxy must enforce.

**Path parameters**

| Name | Type | Description |
|---|---|---|
| `uuid` | uuid | Player UUID |

**Request body** — [`PlayerJoinRequest`](#playerjoinrequest)

```json
{ "name": "Steve" }
```

**Responses**

`200 OK` — [`PlayerJoinResponse`](#playerjoinresponse)

```json
{
  "player": {
    "uuid": "00000000-0000-0000-0000-000000000001",
    "name": "Steve",
    "firstSeen": "2026-06-28T10:00:00Z",
    "lastSeen": "2026-06-28T10:00:00Z",
    "playtime": 0
  },
  "activePunishments": []
}
```

---

### Record quit

`POST /game/players/{uuid}/quit`

Adds the elapsed session time to the player's total playtime and resets last-seen. Call on disconnect.

**Path parameters**

| Name | Type | Description |
|---|---|---|
| `uuid` | uuid | Player UUID |

**Responses**

`204 No Content`

---

### Get playtime (by uuid)

`GET /game/players/{uuid}/playtime`

Returns total playtime in **milliseconds**, accurate to now.

**Path parameters**

| Name | Type | Description |
|---|---|---|
| `uuid` | uuid | Player UUID |

**Query parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `online` | boolean | yes | `true` adds the live session delta since last-seen; `false` returns the stored value |

**Responses**

`200 OK` — [`PlaytimeResponse`](#playtimeresponse)

```json
{ "playtime": 12493 }
```

`404 Not Found` — unknown player.

---

### Get playtime (by name)

`GET /game/players/by-name/{name}/playtime`

Same as above, keyed by current name.

**Path parameters**

| Name | Type | Description |
|---|---|---|
| `name` | string | Current Minecraft name |

**Query parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `online` | boolean | yes | See above |

**Responses**

`200 OK` — [`PlaytimeResponse`](#playtimeresponse) · `404 Not Found` — unknown name.

---

## Dungeons

### Get a dungeon

`GET /game/dungeons/{dungeonUuid}`

Returns the full dungeon aggregate.

**Path parameters**

| Name | Type | Description |
|---|---|---|
| `dungeonUuid` | uuid | The dungeon's own UUID |

**Responses**

`200 OK` — [`DungeonDto`](#dungeondto) · `404 Not Found`.

---

### Get-or-create the player's dungeon

`POST /game/players/{playerUuid}/dungeon`

Ensures the player has a dungeon and returns it. On first creation, seeds a `"main"` room. Idempotent — repeated calls return the same dungeon.

**Path parameters**

| Name | Type | Description |
|---|---|---|
| `playerUuid` | uuid | The owning player |

**Responses**

`200 OK` — [`DungeonDto`](#dungeondto)

```json
{
  "uuid": "11111111-1111-1111-1111-111111111111",
  "ownerUuid": "00000000-0000-0000-0000-000000000001",
  "doorOpen": false,
  "gate": null,
  "gateModifiers": [],
  "members": [],
  "rooms": [{ "name": "main", "level": 1 }],
  "citizens": [],
  "completedGates": []
}
```

---

### List accessible dungeons

`GET /game/players/{playerUuid}/dungeons`

Summaries of every dungeon the player **owns or is trusted in** (their own included). Never creates.

**Path parameters**

| Name | Type | Description |
|---|---|---|
| `playerUuid` | uuid | The player |

**Responses**

`200 OK` — array of [`DungeonSummaryDto`](#dungeonsummarydto)

```json
[
  {
    "uuid": "11111111-1111-1111-1111-111111111111",
    "ownerUuid": "00000000-0000-0000-0000-000000000001",
    "ownerName": "Steve"
  }
]
```

---

### Trust a member

`POST /game/dungeons/{dungeonUuid}/members/{playerUuid}`

Adds a trusted player who may enter the dungeon. Returns the updated member list.

**Path parameters**

| Name | Type | Description |
|---|---|---|
| `dungeonUuid` | uuid | The dungeon |
| `playerUuid` | uuid | Player to trust |

**Responses**

`200 OK` — array of member UUIDs

```json
["22222222-2222-2222-2222-222222222222"]
```

`400 Bad Request` — the owner cannot be trusted · `409 Conflict` — already trusted · `404 Not Found` — dungeon missing.

---

### Untrust a member

`DELETE /game/dungeons/{dungeonUuid}/members/{playerUuid}`

Removes a trusted player. Returns the updated member list.

**Path parameters**

| Name | Type | Description |
|---|---|---|
| `dungeonUuid` | uuid | The dungeon |
| `playerUuid` | uuid | Player to untrust |

**Responses**

`200 OK` — array of member UUIDs · `400 Bad Request` — owner is never a member · `404 Not Found` — player wasn't trusted.

---

### Create a room

`POST /game/dungeons/{dungeonUuid}/rooms`

Creates a room (always at level 1).

**Path parameters**

| Name | Type | Description |
|---|---|---|
| `dungeonUuid` | uuid | The dungeon |

**Request body** — [`CreateRoomRequest`](#createroomrequest)

```json
{ "name": "armory" }
```

**Responses**

`200 OK` — [`DungeonRoomDto`](#dungeonroomdto)

```json
{ "name": "armory", "level": 1 }
```

`409 Conflict` — a room with that name already exists · `404 Not Found` — dungeon missing.

---

### Update a room

`PATCH /game/dungeons/{dungeonUuid}/rooms/{room}`

Updates a room (e.g. level-up). The room name is immutable.

**Path parameters**

| Name | Type | Description |
|---|---|---|
| `dungeonUuid` | uuid | The dungeon |
| `room` | string | Room name |

**Request body** — [`UpdateRoomRequest`](#updateroomrequest)

```json
{ "level": 3 }
```

**Responses**

`200 OK` — [`DungeonRoomDto`](#dungeonroomdto) · `404 Not Found` — room missing.

---

### Open a gate

`POST /game/dungeons/{dungeonUuid}/gate/open`

Starts a gate: sets the gate, replaces its modifiers, and opens the door.

**Path parameters**

| Name | Type | Description |
|---|---|---|
| `dungeonUuid` | uuid | The dungeon |

**Request body** — [`OpenGateRequest`](#opengaterequest)

```json
{ "gate": "fire", "modifiers": ["double-spawn", "no-rest"] }
```

**Responses**

`204 No Content` · `404 Not Found` — dungeon missing.

---

### Complete a gate

`POST /game/dungeons/{dungeonUuid}/gate/complete`

Completes the active gate: records the completion (increments the count, sets first-completed on first time), clears the gate and modifiers, and closes the door.

**Path parameters**

| Name | Type | Description |
|---|---|---|
| `dungeonUuid` | uuid | The dungeon |

**Responses**

`200 OK` — [`DungeonCompletedGateDto`](#dungeoncompletedgatedto)

```json
{ "gate": "fire", "completionCount": 2, "firstCompletedAt": "2026-06-28T10:05:00Z" }
```

`400 Bad Request` — no active gate · `404 Not Found` — dungeon missing.

---

### Open / close the door

`POST /game/dungeons/{dungeonUuid}/door/open`
`POST /game/dungeons/{dungeonUuid}/door/close`

Toggles the door, keeping the current gate and modifiers.

**Path parameters**

| Name | Type | Description |
|---|---|---|
| `dungeonUuid` | uuid | The dungeon |

**Responses**

`204 No Content` · `404 Not Found` — dungeon missing.

---

## Friends

### Send a friend request

`POST /game/players/{uuid}/friend-requests`

Sends a request **by name**. If the target already has a pending request to you, this auto-accepts into a friendship. Publishes `friend.request.sent` or `friend.request.accepted`.

**Path parameters**

| Name | Type | Description |
|---|---|---|
| `uuid` | uuid | Acting player |

**Request body** — [`SendFriendRequest`](#sendfriendrequest)

```json
{ "targetName": "Alex" }
```

**Responses**

`200 OK` — [`SendFriendResponse`](#sendfriendresponse)

```json
{ "result": "REQUESTED" }
```

`result` is `REQUESTED` (request created) or `FRIENDED` (auto-accepted).

`400 Bad Request` — friending yourself · `404 Not Found` — target name unknown · `409 Conflict` — already friends, or a request was already sent.

---

### List outgoing requests

`GET /game/players/{uuid}/friend-requests/outgoing`

Requests you've sent, including each target's name.

**Path parameters**

| Name | Type | Description |
|---|---|---|
| `uuid` | uuid | Acting player |

**Responses**

`200 OK` — array of [`FriendRequestDto`](#friendrequestdto)

```json
[{ "playerUuid": "33333333-3333-3333-3333-333333333333", "playerName": "Alex" }]
```

---

### List incoming requests

`GET /game/players/{uuid}/friend-requests/incoming`

Requests you've received, including each sender's name.

**Path parameters**

| Name | Type | Description |
|---|---|---|
| `uuid` | uuid | Acting player |

**Responses**

`200 OK` — array of [`FriendRequestDto`](#friendrequestdto).

---

### Cancel a request

`DELETE /game/players/{uuid}/friend-requests/outgoing/{targetUuid}`

Cancels a request you sent. Publishes `friend.request.cancelled`.

**Path parameters**

| Name | Type | Description |
|---|---|---|
| `uuid` | uuid | Acting player (sender) |
| `targetUuid` | uuid | The request's receiver |

**Responses**

`204 No Content` · `404 Not Found` — no such outgoing request.

---

### Accept a request

`POST /game/players/{uuid}/friend-requests/incoming/{senderUuid}/accept`

Accepts an incoming request, creating the friendship. Publishes `friend.request.accepted`.

**Path parameters**

| Name | Type | Description |
|---|---|---|
| `uuid` | uuid | Acting player (receiver) |
| `senderUuid` | uuid | The request's sender |

**Responses**

`204 No Content` · `404 Not Found` — no such incoming request.

---

### Deny a request

`POST /game/players/{uuid}/friend-requests/incoming/{senderUuid}/deny`

Denies an incoming request. Publishes `friend.request.denied`.

**Path parameters**

| Name | Type | Description |
|---|---|---|
| `uuid` | uuid | Acting player (receiver) |
| `senderUuid` | uuid | The request's sender |

**Responses**

`204 No Content` · `404 Not Found` — no such incoming request.

---

### List friends

`GET /game/players/{uuid}/friends`

Your friends, with the time each friendship was established.

**Path parameters**

| Name | Type | Description |
|---|---|---|
| `uuid` | uuid | Acting player |

**Responses**

`200 OK` — array of [`FriendDto`](#frienddto)

```json
[{ "uuid": "33333333-3333-3333-3333-333333333333", "name": "Alex", "friendsSince": "2026-06-28T10:10:00Z" }]
```

---

### Unfriend

`DELETE /game/players/{uuid}/friends/{friendUuid}`

Removes a friendship. Publishes `friendship.removed`.

**Path parameters**

| Name | Type | Description |
|---|---|---|
| `uuid` | uuid | Acting player |
| `friendUuid` | uuid | The friend to remove |

**Responses**

`204 No Content` · `404 Not Found` — not friends.

---

## Schemas

### PlayerJoinRequest
| Field | Type | Nullable | Description |
|---|---|---|---|
| `name` | string | no | Current Minecraft name |

### PlayerDto
| Field | Type | Nullable | Description |
|---|---|---|---|
| `uuid` | uuid | no | |
| `name` | string | yes | Cached current name |
| `firstSeen` | date-time | no | |
| `lastSeen` | date-time | no | |
| `playtime` | int64 | no | Milliseconds |

### PlayerJoinResponse
| Field | Type | Nullable | Description |
|---|---|---|---|
| `player` | [PlayerDto](#playerdto) | no | |
| `activePunishments` | [ActivePunishmentDto](#activepunishmentdto)[] | no | Empty if none |

### ActivePunishmentDto
| Field | Type | Nullable | Description |
|---|---|---|---|
| `type` | enum `BAN` \| `MUTE` | no | |
| `reason` | string | no | |
| `expiresAt` | date-time | yes | `null` = permanent |

### PlaytimeResponse
| Field | Type | Nullable | Description |
|---|---|---|---|
| `playtime` | int64 | no | Milliseconds, accurate as of now |

### DungeonDto
| Field | Type | Nullable | Description |
|---|---|---|---|
| `uuid` | uuid | no | |
| `ownerUuid` | uuid | no | |
| `doorOpen` | boolean | no | |
| `gate` | string | yes | Active gate, `null` if none |
| `gateModifiers` | string[] | no | Modifiers on the active gate |
| `members` | uuid[] | no | Trusted players |
| `rooms` | [DungeonRoomDto](#dungeonroomdto)[] | no | |
| `citizens` | [DungeonCitizenDto](#dungeoncitizendto)[] | no | |
| `completedGates` | [DungeonCompletedGateDto](#dungeoncompletedgatedto)[] | no | |

### DungeonSummaryDto
| Field | Type | Nullable | Description |
|---|---|---|---|
| `uuid` | uuid | no | |
| `ownerUuid` | uuid | no | |
| `ownerName` | string | yes | |

### DungeonRoomDto
| Field | Type | Nullable | Description |
|---|---|---|---|
| `name` | string | no | |
| `level` | int | no | |

### DungeonCitizenDto
| Field | Type | Nullable | Description |
|---|---|---|---|
| `name` | string | no | NPC identifier (will expand) |

### DungeonCompletedGateDto
| Field | Type | Nullable | Description |
|---|---|---|---|
| `gate` | string | no | |
| `completionCount` | int64 | no | Times completed |
| `firstCompletedAt` | date-time | no | First completion |

### CreateRoomRequest
| Field | Type | Nullable | Description |
|---|---|---|---|
| `name` | string | no | Room name (unique per dungeon) |

### UpdateRoomRequest
| Field | Type | Nullable | Description |
|---|---|---|---|
| `level` | int | no | New level |

### OpenGateRequest
| Field | Type | Nullable | Description |
|---|---|---|---|
| `gate` | string | no | Gate identifier |
| `modifiers` | string[] | no | 0..n modifiers (defaults to empty) |

### SendFriendRequest
| Field | Type | Nullable | Description |
|---|---|---|---|
| `targetName` | string | no | Name of the player to friend |

### SendFriendResponse
| Field | Type | Nullable | Description |
|---|---|---|---|
| `result` | enum `REQUESTED` \| `FRIENDED` | no | Whether a request was created or auto-accepted |

### FriendRequestDto
| Field | Type | Nullable | Description |
|---|---|---|---|
| `playerUuid` | uuid | no | The other party |
| `playerName` | string | yes | |

### FriendDto
| Field | Type | Nullable | Description |
|---|---|---|---|
| `uuid` | uuid | no | |
| `name` | string | yes | |
| `friendsSince` | date-time | no | |

### ApiError
| Field | Type | Nullable | Description |
|---|---|---|---|
| `status` | int | no | HTTP status code |
| `message` | string | yes | Human-readable detail |

---

## Published events

Friend mutations publish JSON to the `btg.events` topic exchange **after commit**. Each event carries both players (uuid + name); `accepted` also carries `friendsSince`.

| Routing key | Event |
|---|---|
| `friend.request.sent` | `FriendRequestSentEvent` |
| `friend.request.cancelled` | `FriendRequestCancelledEvent` |
| `friend.request.accepted` | `FriendRequestAcceptedEvent` |
| `friend.request.denied` | `FriendRequestDeniedEvent` |
| `friendship.removed` | `FriendshipRemovedEvent` |
