# API Layer

## Conventions

- **Surface:** `/game/**` for Minecraft servers (the only surface implemented today).
- **Format:** JSON in/out.
- **Acting player:** identified by `{uuid}` in the path (will become the JWT subject on `/web/**` later).
- **Errors:** a single `@RestControllerAdvice` returns `ApiError { status, message }` with the matching HTTP code:

| Exception | Status |
|---|---|
| `NotFoundException` | `404` |
| `ConflictException` | `409` |
| `BadRequestException` | `400` |

## Players

| Method | Path | Description |
|---|---|---|
| `POST` | `/game/players/{uuid}/join` | Upsert player; returns player + active punishments |
| `POST` | `/game/players/{uuid}/quit` | Accumulate session playtime (`204`) |
| `GET` | `/game/players/{uuid}/playtime?online={bool}` | Current playtime (ms), online-aware |
| `GET` | `/game/players/by-name/{name}/playtime?online={bool}` | Same, by name |

## Dungeons

| Method | Path | Description |
|---|---|---|
| `GET` | `/game/dungeons/{dungeonUuid}` | Full dungeon aggregate |
| `POST` | `/game/players/{playerUuid}/dungeon` | Get-or-create the player's dungeon |
| `GET` | `/game/players/{playerUuid}/dungeons` | Dungeons the player owns or is trusted in |
| `POST` | `/game/dungeons/{dungeonUuid}/members/{playerUuid}` | Trust a player |
| `DELETE` | `/game/dungeons/{dungeonUuid}/members/{playerUuid}` | Untrust a player |
| `POST` | `/game/dungeons/{dungeonUuid}/rooms` | Create a room (always level 1) |
| `PATCH` | `/game/dungeons/{dungeonUuid}/rooms/{room}` | Update a room (e.g. level) |
| `POST` | `/game/dungeons/{dungeonUuid}/gate/open` | Start a gate (set gate + modifiers, open door) |
| `POST` | `/game/dungeons/{dungeonUuid}/gate/complete` | Complete the gate (record completion, clear, close) |
| `POST` | `/game/dungeons/{dungeonUuid}/door/open` | Open the door |
| `POST` | `/game/dungeons/{dungeonUuid}/door/close` | Close the door |

> Door/gate mutations return `204`, except `gate/complete`, which returns the updated completion entry.

## Friends

| Method | Path | Description |
|---|---|---|
| `POST` | `/game/players/{uuid}/friend-requests` | Send by name; mutual requests auto-accept. Returns `REQUESTED` or `FRIENDED` |
| `GET` | `/game/players/{uuid}/friend-requests/outgoing` | Sent requests (with target name) |
| `GET` | `/game/players/{uuid}/friend-requests/incoming` | Received requests (with sender name) |
| `DELETE` | `/game/players/{uuid}/friend-requests/outgoing/{targetUuid}` | Cancel a request |
| `POST` | `/game/players/{uuid}/friend-requests/incoming/{senderUuid}/accept` | Accept |
| `POST` | `/game/players/{uuid}/friend-requests/incoming/{senderUuid}/deny` | Deny |
| `GET` | `/game/players/{uuid}/friends` | Friends (uuid, name, `friendsSince`) |
| `DELETE` | `/game/players/{uuid}/friends/{friendUuid}` | Unfriend |

## Published events

Every friend mutation publishes to the `btg.events` topic exchange **after commit**. Each event carries both players (uuid + name); `accepted` also carries `friendsSince`.

| Routing key | Event |
|---|---|
| `friend.request.sent` | `FriendRequestSentEvent` |
| `friend.request.cancelled` | `FriendRequestCancelledEvent` |
| `friend.request.accepted` | `FriendRequestAcceptedEvent` |
| `friend.request.denied` | `FriendRequestDeniedEvent` |
| `friendship.removed` | `FriendshipRemovedEvent` |

!!! note "Not yet implemented"
    Moderation write endpoints, the `/web/**` surface, and the web auth flow are planned.
