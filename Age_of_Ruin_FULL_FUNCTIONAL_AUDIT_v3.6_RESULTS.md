# Age of Ruin v3.6 — Functional Audit Results

Target: Minecraft Java 26.2  
Datapack: v3.6, format 107.1  
Resource pack: v3.2, format 88.0  
Acceptance source: `Age_of_Ruin_FULL_FUNCTIONAL_AUDIT_v3.6_SPEC.md`

## Result

The consolidated repair is complete. No known parser, registry, internal-reference, migration, reward-chain, or asset-chain defect remains in the packaged source. The release was validated statically, on a fresh official Java 26.2 dedicated server, through targeted runtime tests, and by extracting and hashing every packaged file.

## Release artifacts

| Artifact | SHA-256 |
|---|---|
| `Age_of_Ruin_Datapack_26.2_v3.6.zip` | `4A5716EB55280F409C2F4FEF4CD46273391FE391EF2E5E91FB6B80269DFCADD9` |
| `Age_of_Ruin_ResourcePack_26.2_v3.2.zip` | `F2F995455FED205770A435925F344402E0EA16081BE194CEEF7394B8471F45F7` |

Resource-pack SHA-1 for `server.properties`: `118ABDEFEEF35E71B6AD48DFE378D46CA3BAC506`

## Inventory and static validation

- 982 functions, 240 advancements, 57 loot tables, 49 item modifiers, 10 recipes, 3 predicates, 9 custom tags, 2 vanilla function tags, and 1 custom enchantment.
- 49 scoreboard objectives; every referenced objective is declared by setup/migration.
- 2,393 internal function references; all resolve, including all macro calls.
- All referenced predicates, loot tables, and item modifiers resolve.
- All 444 JSON/metadata documents across both sources parse.
- 35/35 datapack item-model IDs resolve through resource-pack item definition → model → texture.
- 35 item definitions, 35 models, 34 unique item textures, one pack icon, and 36 English language entries.
- All PNG files have valid PNG/IHDR data and expected dimensions.
- No known invalid 26.2 patterns remain, including bare `revoke`, removed day-repetition syntax, camelCase gamerules, invalid entity/tag IDs, malformed selector ranges, or multi-target `damage` commands.
- Package extraction comparison: datapack 1,359/1,359 files and resource pack 108/108 files matched their release sources byte-for-byte; both have `pack.mcmeta` at ZIP root.

## Official Java 26.2 server validation

- Fresh startup automatically enabled `file/Age_of_Ruin`.
- Loaded 1,595 recipes and 1,928 advancements with zero datapack-loader errors.
- Heartbeat scheduling and Threat recomputation ran normally.
- Simulated v3.5→v3.6 migration preserved Day 42, Corruption 17, milestone state, and custom-boss progression; version became 36, Threat recomputed to 59, and heartbeat advanced from 0 to 8.
- Threat 100 changed a Zombie from 20→50 max health and 3→6 attack damage, matching the exact formulas.
- Enchanted-book modifier raised Sharpness 5→6 using the Java 26.2 stored-enchantment representation.
- Altar test consumed exactly one Sigil from a stack of three.
- Evolved identity, rarity, and affix name components serialized without warnings.
- Custom-boss anchor creation and environmental-death cleanup reset active state and removed the anchor.
- A two-Wither test claimed/scaled one encounter while leaving the unrelated Wither untouched.
- Blood Moon restore macro returned `players_sleeping_percentage` to an arbitrary saved value of 37.

## Repaired functional areas

- Fresh setup, recovery-safe migration, global heartbeat, sleep-safe day rollover, daily reports, Threat/Corruption/Era state, and cross-dimension processing.
- Exact hostile scaling, long-tail durability, rarity probability separation, tier stats/loot, evolved-name preservation, 22 affixes, and scaled encounter adds.
- One-shot special projectiles, bounded Necromancer/boss-add entities, hostile variants, hordes, Nether escalation, Architect construction, and siege cleanup.
- Blood Moon gamerule preservation, nine world events, coherent event targeting, participant-only invasion rewards, and cross-dimension event cleanup.
- Twelve mini-bosses, ten Sigil bosses, active-encounter altar guards, participant rewards, environmental completion anchors, Dragon phases, and isolated Wither completion.
- All 43 enchantment upgrade paths, Soul economy, Protection/Fortification, Soul Fracture, fishing, and five relic passives.
- Twenty-six genetic species, species-valid inheritance, hatch-safe mutation, Mule crosses, Invincible behavior/harvesting, species death loot, and seventeen neutral/friendly variants.
- Resource-pack versioning, missing language names, Infernal Heart asset compatibility, model chains, texture references, and documentation.

## Control result and explicit limits

- `/reload` stopped after recipe/advancement loading on both the Age of Ruin test server and an otherwise vanilla 26.2 control server. Fresh startups are clean; this is not an Age of Ruin-only regression.
- Phoenix Plate is a proactive rescue at or below 30% health with a 30-minute cooldown. Vanilla datapacks do not provide reliable pre-lethal interception.
- The headless server cannot visually render client assets. Asset chains and PNGs are valid, but final appearance still needs an in-game client visual check.
- A real four-player balance session was not simulated. Full-duration affix cadence, every boss phase, statistical fishing/genetics, and subjective combat balance remain multiplayer acceptance work rather than falsely reported automated passes.
