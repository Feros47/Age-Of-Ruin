# Age of Ruin v3.8 — Functional Audit Results

Target: Minecraft Java 26.2  
Datapack format: 107.1  
Companion resource pack: v3.3, format 88.0

## Outcome

Pass. The v3.8 datapack adds bounded, colored health labels for loaded living mobs without changing the v3.3 resource-pack asset chain. Static validation and a fresh official Java 26.2 dedicated-server run completed with no datapack-loader, function-parser, registry, or runtime-command errors.

Release artifacts:

- `Age_of_Ruin_Datapack_26.2_v3.8.zip` — SHA-256 `B4617961A50F2942C116AE3FA2C57E97891BDF41041166DF54A424345690A85E`
- `Age_of_Ruin_ResourcePack_26.2_v3.3.zip` — SHA-256 `57F82BA8B56EE6221FEEA81123F59DB56FF9D8D4DB6D9A211D7C570B0A897F19`; SHA-1 `341D420704153A2F450629BD7466BC0CA592C814`
- Package verification compared all 1,384 datapack source/archive files byte-for-byte with 0 missing, extra, or changed files.

## Static audit

- 1,007 functions
- 371 datapack JSON files
- 240 advancements
- 57 loot tables
- 49 item modifiers
- 10 recipes
- 3 predicates
- 52 declared scoreboard objectives
- 2,426 internal function references
- 45 datapack item-model references, definitions, and matching resource models
- 35 gameplay/resource textures plus pack icon
- 464 parsed JSON/metadata documents across both packs
- 0 warnings and 0 errors from the repository audit script

## Health-label acceptance

- [x] A living entity receives exactly one centered `text_display` passenger.
- [x] The label renders `❤ current / max HP` using integer health values.
- [x] Green rendering verified at 100% health.
- [x] Yellow rendering verified at 50% health.
- [x] Red rendering verified at 20% health.
- [x] Repeated attachment/repair checks do not create duplicates.
- [x] A label follows its parent after teleportation without a per-tick teleport command.
- [x] Detached displays are identified and removed while the valid replacement remains.
- [x] Miniboss and boss offsets were verified at 1.15 and 1.75 blocks.
- [x] Out-of-range cleanup removes the display, tracking tag, and temporary scores.
- [x] Tracking is limited to living non-player entities within 48 blocks of players and removed beyond 56 blocks.

## Server acceptance

- Fresh official Minecraft Java 26.2 server startup succeeded.
- The server loaded 1,595 recipes and 1,928 advancements.
- Version marker initialized to 38.
- Simulated v3.7→v3.8 migration preserved Day 42, Corruption 17, and milestone state.
- The server shut down cleanly after testing.
- The log contained no datapack-loader or command errors. The Windows Perflib counter warning and intentional offline-mode warning are host/test-environment messages, not pack defects.

## Scope

Health labels cover passive mobs, villagers, hostile mobs, genetic entities, variants, minibosses, and custom bosses. Players, armor stands, items, projectiles, and utility display entities are intentionally excluded. Labels update once per second; riding the parent keeps motion continuous. Resource pack v3.3 remains current because vanilla text displays need no new texture or model.

## Client note

The dedicated server validates label data, colors, passenger linkage, transformations, and cleanup behavior, but it cannot render client pixels. An in-game check is recommended only for subjective GUI-scale/height preference.
