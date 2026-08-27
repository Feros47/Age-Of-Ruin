# Age of Ruin v3.9 / Resource Pack v3.4 — Functional Audit

Target: Minecraft Java 26.2  
Datapack format: 107.1  
Resource-pack format: 88.0

## Outcome

Pass. The purple/black boss model and missing gene/affix presentation were reproduced and traced to separate display-layer problems. Both are repaired in datapack v3.9 and resource pack v3.4.

Release artifacts:

- `Age_of_Ruin_Datapack_26.2_v3.9.zip` — SHA-256 `96DF27CC428E716EDDE2A140C3A3F5B3B48F520A4E0E1B45C04A3003F8645792`
- `Age_of_Ruin_ResourcePack_26.2_v3.4.zip` — SHA-256 `54CCB73214CD45BC3927185876888EB773AB4BE8E35E60D5A02C9DA2536E7732`; SHA-1 `ED77850DAA8D456C6B1099EFAD4D2589A7780552`
- Datapack package verification: 1,387 source/archive files, 0 mismatches.
- Resource-pack verification: 130 source/archive files, 0 mismatches.

## Boss texture diagnosis

The Java client log loaded all ten `ruin:boss/*` models but reported `ruin:boss/boss_atlas_v3_3` as missing for every model. The v3.3 PNG existed, but it was under `textures/boss`, which is not registered by Java 26.2's vanilla item atlas.

The vanilla 26.2 `assets/minecraft/atlases/items.json` was inspected directly. Its directory source is `textures/item` with the `item/` resource prefix. Resource pack v3.4 therefore:

- moves the atlas to `assets/ruin/textures/item/boss_atlas_v3_4.png`;
- changes all ten models to `ruin:item/boss_atlas_v3_4`;
- adds the same valid texture as each model's particle reference;
- removes the obsolete unregistered atlas path.

All ten corrected model references and the packaged PNG were validated.

## Gene, affix, and health-label repair

Official-server inspection proved the underlying `CustomName` data was intact:

- genetic test: `Genes • Vigor • Swift • Lucky` with preserved component colors;
- affix test: `TEST ZOMBIE • Elite • Armored • Swift` with preserved tier/affix colors.

The health display could visually overlap/depth-hide the separate vanilla nameplate. v3.9 mirrors an intentionally visible `CustomName` into the same centered text display:

1. colored `❤ current / max HP`;
2. exact original name component, including genes, tiers, affixes, colors, and bold formatting.

The underlying entity name is preserved. Its vanilla visibility is temporarily disabled only while mirrored, then restored when health tracking ends.

## Validation results

- 1,010 functions, 2,433 internal function calls, and 52 declared objectives.
- 371 datapack JSON documents and 464 JSON/meta documents across both sources.
- 45 item-model references, 45 item definitions, 45 matching models, and 35 textures.
- Static audit: 0 warnings, 0 errors.
- Fresh official Java 26.2 server: 1,595 recipes and 1,928 advancements, with no pack-loader or command-parser errors.
- Gene two-line display: passed.
- Affix two-line display: passed.
- Health color update without losing affixes: passed.
- Name visibility restoration after cleanup: passed.
- v3.8→v3.9 migration preserved Day 42, Corruption 17, and milestone state; existing genetic entities were marked for refresh.

## Deployment configuration

After uploading the v3.4 ZIP to the repository's `main` branch, use:

```properties
resource-pack=https\://raw.githubusercontent.com/Feros47/Age-Of-Ruin/main/Age_of_Ruin_ResourcePack_26.2_v3.4.zip
resource-pack-id=
resource-pack-prompt={"text"\:"Age of Ruin requires the resource pack. Accept it to join the server.","color"\:"gold"}
resource-pack-sha1=ED77850DAA8D456C6B1099EFAD4D2589A7780552
require-resource-pack=true
```

The changed filename and SHA-1 force clients to download the corrected resource pack instead of reusing v3.3.

## Client note

The resource location is validated against the actual vanilla 26.2 atlas definition, and the server-side display/component behavior is fully tested. Final pixels require one reconnect after v3.4 is uploaded and configured because the dedicated server does not render client assets.
