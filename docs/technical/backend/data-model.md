# Data Model

PostgreSQL, owned by Flyway.

## :material-sitemap: Entity relationships

[![Entity relationship diagram](../../assets/data-model.png){ loading=lazy }](../../assets/data-model.png)

## :material-database: Migrations

| Version | Name | Description |
|---|---|---|
| **V1** | `init` | Baseline schema: `player`, `web_account`, `verification_code`, `dungeon` (+ `dungeon_member`, `gate_modifier`, `dungeon_room`, `dungeon_citizen`), `player_collection`, `friend_request`, `friendship`, `active_punishment`, `moderation_log`. |
| **V2** | `player_name_nullable` | Makes `player.name` nullable so a name can be temporarily unknown when reclaimed by another player. |
| **V3** | `dungeon_completed_gate` | Adds `dungeon_completed_gate` — per-dungeon gate completion tracking with count and first-completed timestamp. |
| **V4** | `collection_catalog` | Adds the static catalog (`catalog_collection_category`, `catalog_collection`, `catalog_collection_level`) and rebuilds `player_collection` to reference it (drops `category`, renames `type` → `collection_key`). |
| **V5** | `spawn_and_gate_data` | Adds `player.current_dungeon_uuid`, `dungeon.world_initialized`, and `dungeon.gate_data` (JSONB) for spawn resolution and opaque per-gate state. |
| **V6** | `collection_claimed_level` | Adds `player_collection.claimed_level` — the claim ledger high-water mark for exactly-once level rewards. |

> Migrations are forward-only and additive; each is applied automatically by Flyway on startup. File names follow `V<n>__<name>.sql` in `backend/src/main/resources/db/migration`.
