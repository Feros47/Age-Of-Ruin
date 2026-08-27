# Age of Ruin v3.10 / Resource Pack v3.5 — Functional Audit Results

Target: Minecraft Java 26.2  
Datapack format: 107.1  
Resource-pack format: 88.0  
Audit date: 2026-08-27

## Result

PASS. The repaired source completed static reference/asset validation and a clean official Java 26.2 dedicated-server regression. No datapack-loader, command-parser, missing-function, missing-objective, registry, or runtime-command errors were found.

## Repairs covered by this audit

- Health displays are summoned at the tracked entity, eliminating the first-seen movement from player to mob.
- Current and maximum health are sampled at 1/100 precision every game tick; text is rebuilt only after a value or displayed name changes.
- Visible names, genes and affixes remain preserved as the second line of the health display.
- All ten boss assets are solid nine-cuboid models with six textured faces per cuboid and no camera-facing billboard behavior.
- Boss carriers use a real infinite invisibility effect because Java 26.2 discards the former living-entity `Invisible` merge.
- Carrier hands and armor are cleared once so vanilla textures/equipment do not show through the replacement model.
- Boss displays follow carrier position and rotation every tick.
- Custom boss rewards remain direct defeat rewards; Leviathan awards both Leviathan Plate and Tidebreaker.

## Static validation

| Check | Result |
|---|---:|
| Functions | 1,017 |
| Advancements | 240 |
| Recipes | 10 |
| Loot tables | 57 |
| Item modifiers | 49 |
| Predicates | 3 |
| Function references | 2,447 |
| Macro functions | 6 |
| Scoreboard objectives | 56 |
| Datapack item-model references | 45 |
| Resource item definitions | 45 |
| Resource models | 45 |
| Textures | 35 |
| PNG files | 36 |
| JSON/metadata documents | 464 |
| Warnings | 0 |
| Errors | 0 |

Additional boss-model checks: 10 models, 90 cuboid elements, all required faces present, matching `boss_atlas_v3_5.png`, and zero vertical billboards.

## Official Java 26.2 server regression

- Fresh world startup loaded 1,595 recipes and 1,928 advancements.
- A cow tracked from an execution origin at `(0,100,0)` received its display at the cow position `(10.5,80,10.5)`.
- Health `9.75` cached as `975`; one fast tick updated both the cache and label.
- A later health change to `4.25` cached as `425` and changed the label to yellow on the next fast tick.
- A Grave King test carrier received infinite invisibility with particles and icon hidden.
- Main hand, offhand and armor equipment were absent after visual processing.
- The display used `billboard: "fixed"` and `ruin:boss/grave_king`.
- Carrier yaw changed from 90° to 180°; one visual tick changed display yaw from 90° to 180°.
- Simulated v3.9 → v3.10 migration preserved Day 42, Corruption 17 and the Grave King milestone, set version 310, and recomputed Threat 59.

The Windows test host emitted its unrelated Perflib counter warning, plus expected warnings for intentionally disabled online authentication. Neither concerns the packs.

## Deployment note

Both new ZIPs must replace the older versions. Clients must reconnect or reload resources after the v3.5 resource pack is installed. These are vanilla solid JSON models; fully skeletal/animated entity models would require a client-side mod or custom entity-model system.

## Final packages

- `Age_of_Ruin_Datapack_26.2_v3.10.zip` — SHA-256 `E54804624509AD2266DF3CF847BDC3D39A3A35DC525A37C0014898DC8B2B5EB0`
- `Age_of_Ruin_ResourcePack_26.2_v3.5.zip` — SHA-256 `DACB194E33E116773FB7EAFC3BC8FA520743DD4FE58C8AF6156C082418B257D4`
- Resource-pack SHA-1 for `server.properties`: `ca65200a8fbaacc6700196e47ff34de2619de880`

Archive extraction was byte-compared with the release sources: datapack 1,394/1,394 files and resource pack 130/130 files, with zero missing, extra or changed files. Both archives contain `pack.mcmeta` at the ZIP root.
