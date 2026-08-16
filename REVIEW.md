# TutorialVillager 1.0.1 review

## Review basis

The supplied JAR does not contain Java source files. This review was performed against:

- `plugin.yml`
- `config.yml`
- `TutorialVillagerPlugin.class`
- `TutorialVillagerPlugin$TutorialNpc.class`
- Java 21 bytecode / constant-pool inspection

JAR SHA-256 reviewed: `cb0de5f30834696b788a7407d79c839b9f4e4bf6cc6d1229ffd955e7d40378fc`.

No behavioral source patch is included in this review pass.

## Findings

### HIGH — NPC identity exists only in memory

The plugin stores the relation between spawned entity UUID and logical NPC ID in `Map<UUID, String> entityToNpc`. The spawned villager itself is not marked using `PersistentDataContainer`, scoreboard tags, or another persistent identifier, and the entity UUID is not saved in the config.

Consequences:

- after an abnormal server stop, an already-saved villager can remain in the world while the in-memory map is lost;
- on the next plugin enable, the config definition is spawned again, potentially creating a duplicate/orphan tutorial villager;
- if an NPC entity cannot be found when `removeEntity` or plugin disable runs, the mapping is still discarded, which can also leave an orphan entity behind.

Recommended fix: give every managed entity a `PersistentDataContainer` key containing the logical NPC ID, scan/reconcile tagged entities on chunk/world load, and make spawn idempotent.

### HIGH — Missing worlds are skipped with no later retry

`spawn(TutorialNpc)` calls `Server#getWorld(name)`. If the world is not available, it logs a warning and returns. There is no `WorldLoadEvent` listener and no queued retry.

This is especially relevant when worlds are created or loaded later by a multi-world plugin. The NPC stays absent until a manual `/tutorialnpc reload` or another full reload/restart occurs after the world exists.

Recommended fix: keep pending NPC IDs by world and spawn them on `WorldLoadEvent`; also reconcile on chunk load.

### HIGH — Config parse failures are silently swallowed

`loadNpcs()` catches `RuntimeException` for each stored row and continues without logging the invalid row or reason. A malformed Base64 field or number therefore causes an NPC to disappear silently from the loaded set.

Recommended fix: log NPC ID/row index and exception reason, keep the malformed row untouched, and avoid overwriting config until the administrator explicitly repairs or migrates it.

### MEDIUM — ID normalization is inconsistent between create and edit commands

`create` normalizes IDs by lower-casing and removing everything outside `[a-z0-9_-]`. However, `message`, `name`, `move`, and `remove` only lower-case the provided ID; they do not apply the same normalization.

Example: creating ID `Shop.1` stores it as `shop1`, but later `/tutorialnpc remove Shop.1` looks for `shop.1` and fails.

Recommended fix: apply one canonical `normalizeId` path to every command that accepts an ID, and reject a changed ID explicitly rather than silently deleting characters.

### MEDIUM — Interaction does not filter the hand

`PlayerInteractEntityEvent` exposes the equipment hand used for the interaction, but the handler does not check it. This can produce duplicate/undesired processing in hand-sensitive interaction paths and makes behavior less deterministic with off-hand items.

Recommended fix: process only `EquipmentSlot.HAND` unless off-hand interaction is deliberately supported.

### MEDIUM — Removal is O(n) and only removes the first matching entity mapping

`removeEntity(id)` scans `entityToNpc.entrySet()` until the first mapping with the matching logical ID, then removes at most that one entity. If duplicate mappings exist, additional managed entities remain.

Recommended fix: maintain both `npcId -> entity UUID` and `entity UUID -> npcId`, or remove all mappings for the logical ID during reconciliation.

### MEDIUM — Startup/disable entity lifecycle can orphan entities in unloaded locations

The plugin depends on `Server#getEntity(UUID)` to find the entity before removal. If the entity cannot be found at that moment, the map entry is cleared anyway. Since no persistent marker/reconciliation exists, a later-loaded entity cannot be identified as managed.

Recommended fix: persistent entity tags plus `ChunkLoadEvent` reconciliation; do not treat `getEntity(uuid) == null` as proof that the entity is gone forever.

### LOW — All functionality is concentrated in one plugin class

Command parsing, persistence, entity lifecycle, interaction rendering, and repository state are all implemented in `TutorialVillagerPlugin`. This makes regression testing and future feature work harder.

Recommended split:

- `TutorialNpc` immutable model
- `NpcRepository` YAML persistence and migration
- `NpcEntityService` spawn/reconcile/remove
- `NpcCommand` command parsing
- `NpcInteractionListener` interaction/damage handling

### LOW — Custom pipe/Base64 storage is difficult to maintain

The `npcs` list uses an undocumented positional string format. Although Base64 protects display/message from the `|` separator, there is no schema version and malformed rows are hard to edit manually.

Recommended fix: structured YAML sections keyed by NPC ID and a `data-version` field, with one-time migration from the current nine-field format.

### LOW — No tab completion and console administration

The plugin registers only a `CommandExecutor`, not a `TabCompleter`. It also rejects every non-player command, including `list`, `reload`, and `remove`, even though some operations do not inherently require a player location.

Recommended fix: tab completion for subcommands/IDs; allow console for non-location commands.

### LOW — Interaction has no anti-spam/cooldown

Every accepted right-click sends the full configured message. Players can rapidly spam chat by clicking repeatedly.

Recommended fix: optional short per-player/per-NPC cooldown.

## Things that are implemented well

- Java 21 / Spigot 1.21 API metadata is internally consistent.
- NPC data is represented as an immutable record-like class.
- Messages support `%player%`, multiple lines, and `&` color codes.
- Managed villagers have AI disabled, are invulnerable, silent, non-collidable, and have visible custom names.
- Updates save immediately after create/message/name/move/remove.
- `createauto` avoids ID collision by selecting the first unused `npcN` identifier.
- `Objects.requireNonNull(getCommand(...))` fails fast if `plugin.yml` and code drift apart.

## Priority remediation order

1. Persistent NPC identity + reconciliation (PDC / world/chunk lifecycle).
2. World-load retry.
3. Explicit config parse errors and safe migration/storage.
4. Canonical ID handling for every command.
5. Main-hand filtering and optional click cooldown.
6. Refactor into services and add tests.

## Verification limitations

This review is static. No live Spigot server was started in this pass, so runtime behavior under chunk unload/reload, Multiverse-style delayed world loading, crash recovery, and concurrent plugin interactions remains to be verified with integration tests.