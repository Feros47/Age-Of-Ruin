# Age of Ruin v3.7 — Boss Model Repair Results

Target: Minecraft Java 26.2  
Datapack: v3.7 / format 107.1  
Required resource pack: v3.3 / format 88.0

## Outcome

The ten custom bosses now have ten distinct vanilla-compatible visual models. Invisible carrier mobs retain the original AI, mechanics, health, hitboxes, phases and rewards; `item_display` visuals follow them at 20 Hz and clean themselves when an encounter ends.

Models were added for Grave King, Infernal Behemoth, Abyss Watcher, Leviathan, Ancient Colossus, Lich King, Infernal Titan, Warden King, Storm Herald and Void Emperor.

## Validation

- Clean official Java 26.2 startup: 1,595 recipes and 1,928 advancements, zero datapack-loader errors.
- 996 functions, 2,411 internal function references and 464 JSON/metadata documents validated.
- 45/45 datapack item-model references resolve through 45 definitions, 45 models and 35 textures.
- All ten test carriers produced exactly ten distinct displays with dedicated model IDs, scales and vertical billboarding.
- Carrier invisibility, automatic display recreation, 20-block movement following and ten-to-zero orphan cleanup passed live.
- A real Grave King encounter created and cleaned its dedicated visual correctly.
- v3.6→v3.7 migration preserved Day, Corruption and milestone state.
- Package verification found zero mismatches across 1,373 datapack files and 129 resource-pack files.

## Artifacts and hashes

| Artifact | SHA-256 |
|---|---|
| `Age_of_Ruin_Datapack_26.2_v3.7.zip` | `31DFE0BB002AEFFB6445F6C24588DED084DCC167C5ADD44CD262AEF261B1F10E` |
| `Age_of_Ruin_ResourcePack_26.2_v3.3.zip` | `57F82BA8B56EE6221FEEA81123F59DB56FF9D8D4DB6D9A211D7C570B0A897F19` |

Resource-pack SHA-1 for `server.properties`: `341D420704153A2F450629BD7466BC0CA592C814`

## Installation requirement

Both new ZIPs are required. Resource pack v3.2 and older do not contain the boss models. Replace the old server/client resource-pack file, update its SHA-1, require the resource pack, and reconnect or reload client resources.

The dedicated server validates display NBT and asset dependency chains but cannot render client graphics. Final pixel appearance should be checked once in-game with v3.3 active.
