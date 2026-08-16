# Better_villager

Review workspace for `TutorialVillager-1.0.1-Spigot-1.21.1`.

The supplied JAR contains compiled Java bytecode but no `.java` source files. The current specification below was recovered from `plugin.yml`, `config.yml`, and Java 21 bytecode. No behavior-changing patch has been applied in this review pass.

## Runtime

- Minecraft / Spigot target: 1.21.x (`api-version: 1.21`)
- Java bytecode: Java 21 (class major version 65)
- Plugin name: `TutorialVillager`
- Main class: `com.sagakenichi.tutorialvillager.TutorialVillagerPlugin`
- Version: `1.0.1`

## Purpose

Creates persistent-looking tutorial villagers at configured locations. A managed villager is non-AI, invulnerable, silent, non-collidable, has a visible custom name, and does not naturally despawn when far from players. Right-clicking a currently tracked tutorial villager cancels the normal interaction and sends configured tutorial text to the clicking player.

## Commands

Main command: `/tutorialnpc`
Alias: `/tnpc`
Permission: `tutorialvillager.admin` (default: OP)

The command handler only accepts in-game players; console use is rejected.

- `/tutorialnpc create <ID> <display_name> <message...>`
  - Creates an NPC at the executing player's location.
  - The supplied ID is lower-cased and all characters except `a-z`, `0-9`, `_`, and `-` are removed.
  - In this command only, underscores in `display_name` are converted to spaces.
- `/tutorialnpc createauto <display_name> <message...>`
  - Creates an NPC with the first free automatic ID `npc1`, `npc2`, ...
  - Underscores in `display_name` are converted to spaces.
- `/tutorialnpc message <ID> <message...>`
  - Updates only the stored interaction message.
- `/tutorialnpc name <ID> <display name...>`
  - Updates the custom display name, removes the currently tracked entity, saves, and respawns it.
- `/tutorialnpc move <ID>`
  - Moves the NPC definition to the executing player's current world, position, yaw, and pitch; then removes and respawns the entity.
- `/tutorialnpc remove <ID>`
  - Removes the NPC definition and the currently tracked entity.
- `/tutorialnpc list`
  - Lists stored NPC definitions, including ID, display name, and world.
- `/tutorialnpc reload`
  - Removes currently tracked entities, reloads `config.yml`, loads NPC definitions, and respawns them.

## Interaction text

When a tracked NPC is right-clicked:

1. The normal entity interaction is cancelled.
2. A header containing the NPC display name is sent.
3. Literal `\\n` in the stored message is converted to new lines.
4. `%player%` is replaced with the clicking player's name.
5. `&` color codes are translated with Bukkit `ChatColor`.

## Storage

The default configuration is:

```yaml
npcs: []
```

Each NPC is stored as one pipe-delimited string with nine fields:

`id | base64url(displayName) | base64url(message) | world | x | y | z | yaw | pitch`

Display name and message are Base64 URL encoded without padding. IDs and world names are stored as plain text.

## Entity lifecycle

At enable, definitions are loaded and each NPC is spawned if its configured world is currently available. At normal plugin disable, entities that are still discoverable through the plugin's in-memory UUID map are removed. The config definitions remain and are respawned at the next enable.

See `REVIEW.md` for the source/bytecode review and risks found.