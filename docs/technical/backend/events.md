# Events

All events publish to the **`btg.events`** topic exchange **after the DB commit** — a rolled-back or no-op action (e.g. unbanning a clean player) emits nothing. Payloads are JSON and self-contained: every referenced player is a [`PlayerRef`](#playerref) (uuid + name), so consumers can render without a lookup.

Bind a queue with a routing-key pattern to receive a category:

| Pattern | Receives |
|---|---|
| `friend.#` | all friend events |
| `moderation.#` | all moderation events |
| `verification.code` | the in-game verification code delivery |
| `#` | everything |

---

## :material-account-group: Friends

| Routing key | Payload |
|---|---|
| `friend.request.sent` | [`FriendRequestSentEvent`](#friendrequestsentevent) |
| `friend.request.cancelled` | [`FriendRequestCancelledEvent`](#friendrequestcancelledevent) |
| `friend.request.accepted` | [`FriendRequestAcceptedEvent`](#friendrequestacceptedevent) |
| `friend.request.denied` | [`FriendRequestDeniedEvent`](#friendrequestdeniedevent) |
| `friendship.removed` | [`FriendshipRemovedEvent`](#friendshipremovedevent) |

---

## :material-gavel: Moderation

| Routing key | Payload |
|---|---|
| `moderation.ban` | [`ModerationBanEvent`](#moderationbanevent) |
| `moderation.mute` | [`ModerationMuteEvent`](#moderationmuteevent) |
| `moderation.unban` | [`ModerationUnbanEvent`](#moderationunbanevent) |
| `moderation.unmute` | [`ModerationUnmuteEvent`](#moderationunmuteevent) |
| `moderation.kick` | [`ModerationKickEvent`](#moderationkickevent) |

---

## :material-shield-key: Verification codes

Published when a web account needs an in-game code (registration, resend, or password reset). Consumed by the **proxy**, which messages the player if they're online. The code is **plaintext here only** — the database stores just a hash.

| Routing key | Payload |
|---|---|
| `verification.code` | [`DeliverCodeEvent`](#delivercodeevent) |

---

## :material-code-braces: Payloads

### PlayerRef
| Field | Type | Notes |
|---|---|---|
| `uuid` | UUID | |
| `name` | string? | snapshot at emit time; may be unknown |

### FriendRequestSentEvent
| Field | Type | Notes |
|---|---|---|
| `sender` | [PlayerRef](#playerref) | |
| `receiver` | [PlayerRef](#playerref) | |

### FriendRequestCancelledEvent
| Field | Type | Notes |
|---|---|---|
| `sender` | [PlayerRef](#playerref) | |
| `receiver` | [PlayerRef](#playerref) | |

### FriendRequestAcceptedEvent
| Field | Type | Notes |
|---|---|---|
| `sender` | [PlayerRef](#playerref) | original requester |
| `receiver` | [PlayerRef](#playerref) | accepter |
| `friendsSince` | timestamp | |

### FriendRequestDeniedEvent
| Field | Type | Notes |
|---|---|---|
| `sender` | [PlayerRef](#playerref) | |
| `receiver` | [PlayerRef](#playerref) | |

### FriendshipRemovedEvent
| Field | Type | Notes |
|---|---|---|
| `remover` | [PlayerRef](#playerref) | who unfriended |
| `removed` | [PlayerRef](#playerref) | |

### ModerationBanEvent
| Field | Type | Notes |
|---|---|---|
| `target` | [PlayerRef](#playerref) | banned player |
| `moderator` | [PlayerRef](#playerref)? | `null` = console/system |
| `reason` | string | |
| `startsAt` | timestamp | |
| `endsAt` | timestamp? | `null` = permanent |

### ModerationMuteEvent
| Field | Type | Notes |
|---|---|---|
| `target` | [PlayerRef](#playerref) | muted player |
| `moderator` | [PlayerRef](#playerref)? | `null` = console/system |
| `reason` | string | |
| `startsAt` | timestamp | |
| `endsAt` | timestamp? | `null` = permanent |

### ModerationUnbanEvent
| Field | Type | Notes |
|---|---|---|
| `target` | [PlayerRef](#playerref) | |
| `moderator` | [PlayerRef](#playerref)? | `null` = console/system |
| `at` | timestamp | when lifted |

### ModerationUnmuteEvent
| Field | Type | Notes |
|---|---|---|
| `target` | [PlayerRef](#playerref) | |
| `moderator` | [PlayerRef](#playerref)? | `null` = console/system |
| `at` | timestamp | when lifted |

### ModerationKickEvent
| Field | Type | Notes |
|---|---|---|
| `target` | [PlayerRef](#playerref) | |
| `moderator` | [PlayerRef](#playerref)? | `null` = console/system |
| `reason` | string? | optional |
| `at` | timestamp | when kicked |

### DeliverCodeEvent
| Field | Type | Notes |
|---|---|---|
| `playerUuid` | UUID | recipient |
| `code` | string | 6-digit, plaintext, 10-minute validity |
| `purpose` | enum | `ACCOUNT_ACTIVATION` \| `PASSWORD_RESET` |
