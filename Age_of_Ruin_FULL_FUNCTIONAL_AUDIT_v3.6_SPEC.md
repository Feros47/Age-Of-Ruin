# AGE OF RUIN — FULL FUNCTIONAL AUDIT & ACCEPTANCE SPECIFICATION

**Target:** Minecraft Java Edition 26.2  
**Datapack target format:** 107.1  
**Resource-pack target format:** 88.0  
**Purpose:** This is the canonical specification for how Age of Ruin is supposed to behave. It is not a player guide. It is an engineering audit, implementation contract, and QA checklist covering the complete datapack, resource pack, progression loop, hostile mobs, neutral/friendly mobs, genetics, bosses, loot, enchantments, fishing, siege behavior, world events, admin tooling, migration, and acceptance testing.

---

# 0. CANONICAL DESIGN RULES

Age of Ruin must obey all of the following rules everywhere in the implementation.

1. **The world becomes more dangerous over time.** Difficulty is driven by Minecraft world age and permanent Corruption gained from player progression.
2. **Difficulty must not be only HP inflation.** Higher Threat must add rarity, affixes, evolved variants, events, siege behavior, stronger enemy groups, mini-bosses, and boss mechanics.
3. **Rewards must scale with danger.** Harder rarity tiers and bosses must provide better loot and progression currency.
4. **Vanilla bosses must never be hard-locked by equipment checks.** The Ender Dragon and Wither can always be fought when vanilla Minecraft allows it. They should simply be so difficult that attempting them too early is impractical.
5. **The Ender Dragon should be balanced around approximately full Protection VIII gear plus a Power X-level bow for a four-player group.** This is a balance target, not a permission gate.
6. **The Wither must be substantially harder than the Dragon.** It should feel like the final vanilla boss rather than an easier post-End encounter.
7. **Long-term progression must continue to level 255 enchantments.** Very late-game Threat must make ordinary vanilla enchantments inadequate.
8. **Every hostile or potentially hostile mob should either scale, evolve, gain special variants, or participate in another Age of Ruin system.** No large hostile family should remain untouched.
9. **The Nether must be independently more dangerous than the Overworld at the same Threat.** It must have its own variant pressure, warbands, events, and multipliers.
10. **Bases cannot become permanently safe through trivial walls or elevated platforms.** Siege enemies must be able to breach selected blocks and Architect enemies must be able to construct routes toward players.
11. **Friendly and neutral mobs are part of progression.** They can spawn with special traits, genes, utility abilities, unusual loot, and selective-breeding potential.
12. **Genes are combinable.** Offspring can inherit different genes from both parents and create multi-gene bloodlines.
13. **Invincible is mutation-only.** It cannot be inherited directly, and Invincible animals are sterile once adult.
14. **Invincible animals do not need to die to provide resources.** A successful hit causes species-appropriate resource drops.
15. **Rarity, affixes, evolved variant names, and genes must be visible.** Players should be able to understand why an enemy or animal is special.
16. **Normal mobs remain visually uncluttered.** Ordinary mobs without Age of Ruin traits do not need nametags.
17. **All major systems must have a complete runtime chain:** initialization → recurring update → effect → reward/cleanup.
18. **No feature is considered implemented merely because a function exists.** QA must demonstrate that the feature is reachable from the runtime and works in live gameplay.
19. **The resource pack must be compatible with the datapack version and every referenced `ruin:` item model must resolve correctly.**
20. **World upgrades must preserve player/world progression.** Updating the datapack must not reset Day, Corruption, boss progression, or established genetic animals unless explicitly intended.

---

# 1. VERSION, INSTALLATION, AND PACK RELATIONSHIP

## 1.1 Datapack

The datapack belongs in:

`<world>/datapacks/Age_of_Ruin_Datapack_26.2_<version>.zip`

It must remain a valid datapack ZIP with `pack.mcmeta` at the root.

## 1.2 Resource pack

The resource pack is a separate ZIP delivered to clients through the server's resource-pack settings. It is not placed inside the datapacks folder.

Server configuration should use:

- `require-resource-pack=true`
- a direct-download `resource-pack=` URL
- the correct SHA-1 for the exact resource-pack ZIP
- a clear prompt that the pack is required

## 1.3 Compatibility contract

The datapack may reference custom items by `minecraft:item_model='ruin:<id>'`. Every referenced ID must have:

1. `assets/ruin/items/<id>.json`
2. a valid referenced model under `assets/ruin/models/item/`
3. the referenced texture under `assets/ruin/textures/item/`
4. valid JSON and PNG data

The resource pack may remain on an earlier semantic version only if its asset manifest is still fully compatible with the newer datapack. Any new visual asset requires a resource-pack version bump.

---

# 2. RUNTIME ARCHITECTURE

Age of Ruin must have a reliable global runtime independent of player actions.

## 2.1 On load

The datapack must:

- create every required scoreboard objective if missing;
- create/initialize every global fake-player score if missing;
- initialize Day epoch correctly;
- initialize Corruption to 0 on a genuinely fresh world;
- preserve Corruption and progression on migration;
- initialize event state, Blood Moon state, timers, progression flags, rarity probability values, and runtime flags;
- schedule/start the recurring heartbeat;
- mark the currently installed version;
- display a short one-time version message to players who have not seen that version.

## 2.2 Heartbeat

A recurring server heartbeat should run approximately once per second for medium-frequency systems. It must reschedule itself reliably and expose a heartbeat counter for diagnostics.

It must update:

- real world day;
- daytime position;
- Threat and probabilities;
- new-day detection;
- hostile initialization scans;
- neutral/friendly special-mob scans;
- genetics maintenance;
- player systems;
- event state;
- Nether event systems;
- progress/corruption triggers;
- boss maintenance where one-second precision is acceptable;
- cleanup of stale encounter state.

## 2.3 Per-tick runtime

Systems that require 20 Hz precision must use the tick runtime rather than the one-second heartbeat. This includes at minimum:

- special projectile collision handling;
- relic hit/projectile behavior;
- Phoenix Plate lethal-damage approximation or equivalent protection logic;
- boss mechanics that need precise phase/attack cadence;
- siege logic if one-second response creates obvious pathing failures;
- any short-duration damage-immunity mechanic.

## 2.4 Runtime observability

An admin diagnostic must expose:

- heartbeat counter;
- runtime-ready flag;
- raw Minecraft day repetition count;
- Age of Ruin Day;
- Corruption;
- Threat;
- Era;
- current event and event-live state;
- Blood Moon scheduled/live state;
- current daytime value;
- siege state;
- relevant boss runtime state.

The heartbeat must visibly increase between repeated checks.

---

# 3. DAY, THREAT, CORRUPTION, AND ERAS

## 3.1 World day

Age of Ruin Day is based on completed Minecraft day repetitions, not the tick position inside the current day.

Sleeping through the night must correctly advance Age of Ruin Day and trigger the new-day report.

## 3.2 Threat

**Threat = Age of Ruin Day + permanent Corruption**

Day is the baseline. Corruption allows aggressive player progression to accelerate the world.

## 3.3 Corruption sources

At minimum:

| Progression event | Permanent Corruption |
|---|---:|
| First diamond acquired | +1 |
| First Nether entry | +3 |
| First netherite acquisition | +3 |
| Major custom boss defeated | +5 |
| Ender Dragon defeated | +15 |

The progression trigger must fire only once per intended milestone unless a boss is intentionally designed to add Corruption on every repeat kill.

The server should announce major corruption milestones in chat so players understand why Threat increased.

## 3.4 Era structure

| Threat/Day band | Era | Expected gameplay change |
|---|---|---|
| 1–5 | Calm | Mostly vanilla; first rare enhanced enemies |
| 6–15 | Awakening | Elite pressure begins; more equipment/abilities |
| 16–30 | Corruption | Mini-bosses and evolved variants become meaningful |
| 31–50 | Blood Age | Dangerous nights, hordes, Nether escalation |
| 51–75 | Cataclysm | Strong mini-bosses, world events, dangerous caves |
| 76–100 | Ascension | Endgame gearing and legendary pressure |
| 101–150 | Doom | Dragon/Wither-level progression and major bosses |
| 151+ | Endless Ruin | Infinite long-tail scaling and increasingly extreme enemies |

Era boundaries are presentation/progression milestones. Threat itself remains continuous.

---

# 4. DAILY WORLD REPORT

A world report must be automatically sent to all online players whenever Age of Ruin Day increases, including when the day advances by sleeping.

The report should be concise but must show the meaningful current state:

- Day;
- Threat;
- Corruption;
- Era;
- health scaling;
- damage scaling;
- Elite chance;
- Greater Elite chance;
- mini-boss chance;
- Blood Moon chance/forecast;
- world event forecast if one is scheduled;
- special warning if a Blood Moon is scheduled;
- optional Nether/escalation notice at relevant milestones.

The report must never display the tick-within-day value as the world day.

Admin must be able to invoke the report manually for testing.

---

# 5. BASE HOSTILE SCALING

## 5.1 Core formula

Before rarity, variant, Nether, Blood Moon, and other modifiers:

**Base max-health bonus = +1.5% per Threat**  
**Base melee-damage bonus = +1.0% per Threat**

The implementation may bucket Threat in small intervals for command efficiency, but the resulting values should closely track the formula.

Examples:

| Threat | Health multiplier | Damage multiplier |
|---:|---:|---:|
| 1 | ~1.015× | ~1.01× |
| 20 | 1.30× | 1.20× |
| 50 | 1.75× | 1.50× |
| 100 | 2.50× | 2.00× |
| 150 | 3.25× | 2.50× |
| 300 | 5.50× | 4.00× |
| 600 | 10.00× | 7.00× |

## 5.2 Endless durability

Minecraft's native health attribute has practical/engine limits. Once ordinary max-health scaling becomes insufficient, Age of Ruin must add effective durability through supported systems such as:

- absorption;
- Resistance tiers;
- phase mechanics;
- additional lives/refills where explicitly designed.

Long-tail resistance target:

| Threat | Additional global Resistance |
|---:|---|
| 150–249 | Resistance I |
| 250–399 | Resistance II |
| 400–599 | Resistance III |
| 600+ | Resistance IV |

The exact effective-health implementation may change, but the balance goal does not: eventually vanilla Sharpness V–X should feel inadequate, while extreme enchanted weapons become necessary.

## 5.3 Initialization rules

Every naturally spawned scalable hostile must be processed exactly once for:

1. base Threat scaling;
2. dimension modifier;
3. current event modifier;
4. evolved variant roll;
5. mini-boss conversion roll;
6. rarity roll;
7. affix application;
8. loot/currency setup;
9. final name and health refill;
10. initialized marker.

Boss/event-spawned adds must either go through the same pipeline or explicitly receive equivalent scaling. They must not accidentally bypass Threat because they were spawned already tagged initialized.

---

# 6. RARITY SYSTEM

## 6.1 Rarity tiers

| Tier | Name presentation | Core role |
|---|---|---|
| Normal | no special nametag | baseline scaled enemy |
| Elite | green | 1–3 affixes, Soul Shard progression |
| Greater Elite | blue | 3–5 affixes, stronger stats, Greater Soul |
| Epic | dark purple | upgraded Greater Elite; major bonus stats and loot |
| Legendary | gold | extreme random enemy; high stats, 5–6 affixes, legendary loot |
| Mythic | dark red / boss presentation | reserved primarily for bosses and extraordinary rewards |

## 6.2 Rarity probability anchors

The probability curve should follow these design anchors:

| Threat | Elite | Greater Elite | Natural mini-boss |
|---:|---:|---:|---:|
| 1 | 1% | 0% | 0% |
| 10 | 5% | 0.5% | ~0% |
| 25 | 10% | 2% | 0.2% |
| 50 | 16% | 4% | 0.5% |
| 75 | 22% | 6% | 1.0% |
| 100 | 27% | 8% | 1.5% |
| 150+ | 35% | 10% | ~2.5% |
| 200+ | 35% baseline | 10% baseline | 3% baseline |

Event and dimension modifiers can increase the live roll.

## 6.3 Epic/Legendary promotion

Greater Elites may promote upward once the relevant Threat threshold is reached.

Intended rule:

- Epic becomes possible from approximately Threat 50.
- Legendary becomes possible from approximately Threat 100.
- Legendary remains much rarer than Epic.

Current design target is approximately 12.5% of Greater Elites promoting to Epic and 2.5% promoting directly to Legendary at the relevant thresholds.

## 6.4 Nametags

The final visible name must combine rarity/variant identity and affixes, for example:

- `Elite • Swift • Vampiric`
- `Greater Elite • Juggernaut • Commander • Regenerating`
- `Legendary • Phasewalker • Reflective • Executioner`
- `Dread Knight • Armored • Brutal`

Affix naming must remain correct if an enemy is evolved first and promoted later.

---

# 7. COMPLETE AFFIX SYSTEM

Every affix must have its own independent clock/state. One affix may never reset or block another affix on the same mob.

| Affix | Required behavior |
|---|---|
| **Armored** | +12 armor and +4 armor toughness or equivalent durable armor increase |
| **Brutal** | approximately +60% attack damage |
| **Swift** | approximately +45% movement speed; attack pressure should feel faster |
| **Vampiric** | periodically damages a nearby player and heals itself; visible heart effect |
| **Regenerating** | periodically restores health independent of Vampiric |
| **Berserker** | below roughly 30% HP gains major Strength and Speed |
| **Juggernaut** | roughly doubles health, adds large absorption, very high knockback resistance, slightly slower movement |
| **Stormborn** | periodically strikes a nearby player/location with lightning; electric particles |
| **Venomous** | applies strong Poison to nearby players |
| **Withering** | applies Wither to nearby players |
| **Frostbound** | Slowness aura plus freeze damage at close range |
| **Infernal** | Fire Resistance, close-range fire damage, visible flames, safe/controlled fire-trail behavior |
| **Explosive** | periodic telegraphed non-griefing explosion damage; should not randomly demolish important terrain |
| **Necromancer** | periodically summons weaker undead/thralls that are appropriately scaled |
| **Commander** | periodically buffs nearby hostiles with Strength, Speed, Resistance; killing the commander removes future buff pulses |
| **Blinking** | short-range teleports toward/repositions around the player |
| **Reflective** | periodically intercepts/reflects projectiles; must not create infinite projectile loops |
| **Leeching** | damages nearby player, removes/steals absorption, grants itself absorption |
| **Hexed** | randomly applies debuffs such as Weakness, Slowness, Darkness, Mining Fatigue, Hunger |
| **Phasewalker** | short windows of very high damage resistance plus teleport/repositioning |
| **Executioner** | extra damage against players at low HP; must calculate player-health threshold correctly |
| **Architect** | marks mob as a siege builder; can bridge/pillar/create routes toward inaccessible players |

### Affix QA

For a forced Elite with multiple affixes, each timed affix must visibly activate at least once during a controlled 60–90 second test. No affix timer may starve another.

---

# 8. HOSTILE MOB EVOLUTION — COMPLETE COVERAGE

Generic families begin evolved-variant rolls around Threat 20. For most non-core families, approximate variant frequency should scale from ~10% at Threat 20 to ~20% at 50, ~30% at 100, and ~40% at 150+.

Core Zombie/Skeleton/Creeper families use bespoke thresholds.

## 8.1 Zombie family

Applies to Zombie, Husk, Drowned, Zombie Villager where appropriate.

- **Armored Zombie** — early armor-focused variant.
- **Plague Zombie** — poison/plague pressure and enhanced prevalence during Plague events.
- **Berserker Zombie** — stronger/faster at low health.
- **Dread Knight** — late-game heavily equipped, high knockback resistance, siege-capable melee threat.

## 8.2 Skeleton family

Applies to Skeleton, Stray, Bogged, Wither Skeleton where appropriate.

- **Ranger** — mobile/rapid ranged attacker.
- **Sniper** — slower but high-damage shots.
- **Necrotic Archer** — projectiles apply Wither/necrotic pressure.
- **Storm Archer** — special arrows call lightning once on valid impact.
- **Phantom Archer** — reposition/teleport behavior after ranged attacks.

Skeleton weapon enchantment strength should rise with Threat, eventually reaching Power X+ and far beyond in Endless Ruin.

## 8.3 Creepers

- **Charged Creeper** — natural charged variant.
- **Fragmentation Creeper** — secondary explosion/fragmentation behavior.
- **Necrotic Creeper** — leaves Wither/necrotic area pressure.
- **Void Creeper** — teleports/repositions before detonation.
- **Titan Creeper** — huge durability and long, dangerous telegraphed detonation.

## 8.4 Spiders / Cave Spiders

- **Webspinner** — controls movement/web pressure.
- **Broodmother** — summons/supports spider adds.
- **Leaping Hunter** — aggressive leap/mobility behavior.
- **Venomweaver** — stronger poison/venom pressure.

## 8.5 Pillagers

- **Bombardier**
- **Hexshot**
- **War Captain**
- **Sapper**

These must provide distinct ranged/explosive/debuff/commander/siege behavior rather than being name-only variants.

## 8.6 Vindicators

- **Executioner**
- **Blood Axe**
- **Doorbreaker**

Doorbreaker must participate in siege/breach logic.

## 8.7 Evokers / Illusioners

- **Riftcaller**
- **Storm Mage**
- **Hexmaster**

## 8.8 Ravagers

- **Siege Beast**
- **Ironhide**
- **Quake Beast**

## 8.9 Vexes

- **Soulblade**
- **Leeching Vex**
- **Blinkblade**

## 8.10 Silverfish

- **Swarmcaller**
- **Phase Burrower**
- **Corrosive Silverfish**

## 8.11 Endermites

- **Void Leech**
- **Blinkmite**
- **Swarm Seed**

## 8.12 Endermen

- **Void Stalker**
- **Phase Enderman**
- **Abyssal Grasp**

## 8.13 Witches

- **Plague Witch**
- **Storm Witch**
- **Coven Mother**

## 8.14 Blaze

- **Inferno Lord**
- **Solar Flare**
- **Phoenix Blaze**

## 8.15 Ghast

- **Hellstorm Ghast**
- **Soul Screamer**
- **Siege Ghast**

## 8.16 Magma Cube

- **Molten Juggernaut**
- **Volcanic Splitter**
- **Ash Leech**

## 8.17 Slime

- **Acidic Slime**
- **Splitting Horror**
- **Vault Slime**

## 8.18 Piglin

- **Warleader**
- **Gold-Cursed**
- **Spear Charger**

## 8.19 Piglin Brute

- **Butcher**
- **Juggernaut Brute**
- **Brute Executioner**

## 8.20 Zombified Piglin

- **Ash Berserker**
- **Spear Reaver**
- **Soulbound**

## 8.21 Hoglin / Zoglin

- **Gorebeast**
- **Bristleback**
- **Siege Boar**

## 8.22 Breeze

- **Tempest**
- **Stormborn Breeze**
- **Phase Breeze**

## 8.23 Phantom

- **Night Terror**
- **Vampiric Wing**
- **Hexwing**

## 8.24 Shulker

- **Void Artillery**
- **Reflective Shell**
- **Phase Shell**

## 8.25 Guardian

- **Shock Guardian**
- **Tidecaller**
- **Armored Guardian**

## 8.26 Elder Guardian

- **Abyssal Elder**
- **Storm Elder**
- **Tide Tyrant**

## 8.27 Warden

- **Echo Tyrant**
- **Phase Warden**
- **Warden Executioner**

## 8.28 Creaking

- **Root Reaper**
- **Phase Creaking**
- **Dreadwood**

## 8.29 Zombie Nautilus

- **Abyssal Charger**
- **Drowned Bulwark**
- **Tide Leech**

## 8.30 Sulfur Cube

Use the actual 26.2 Sulfur Cube entity type and size/content mechanics; do not reference a nonexistent separate `small_sulfur_cube` entity ID.

- **Volatile Sulfur Cube**
- **Noxious Cube**
- **Sticky Hunter**

## 8.31 Parched

- **Sunscorched Sniper**
- **Dust Hexer**
- **Sandstorm Archer**

Parched evolution must use the same rarity/variant probability philosophy as other families; it must not become 100% evolved merely because Threat passed the unlock threshold.

---

# 9. NETHER-SPECIFIC DIFFICULTY

The Nether receives normal global Threat scaling **plus** additional dimension pressure.

Target modifiers:

- approximately +25% additional effective durability;
- approximately +25% additional damage;
- Elite odds ×1.25;
- Greater Elite odds ×1.5;
- mini-boss odds ×1.5;
- greater prevalence of siege/Infernal variants at high Threat.

## 9.1 Infernal Warbands

Warbands are coordinated groups spawned around **one selected Nether player**. The whole warband must use the same target anchor.

Typical members include:

- Piglins;
- Piglin Brutes;
- Blazes;
- Hoglins;
- at least one properly initialized high-rarity enemy.

Forced Greater/Elite members must go through the real rarity pipeline before final health/name initialization.

## 9.2 Soul Storms

At higher Threat, Soul Storms can create Wither Skeleton/Ghast pressure around **one selected Nether target**.

No individual event spawn should independently select a different player.

---

# 10. HORDES AND UNDEAD PATROLS

From approximately Threat 30 onward, ordinary nights can create tactical patrols independent of full invasions.

Canonical Undead Patrol:

- 1 Commander Zombie;
- 3 Armored Zombies;
- 2 Skeletons;
- 1 true Elite enemy.

Commander pulses Strength, Speed, and Resistance to nearby hostiles. The patrol must spawn around one target area and remain coherent as a group.

---

# 11. SIEGE / BASE-BREACH SYSTEM

Late-game mobs must not be stopped indefinitely by trivial architecture.

## 11.1 Siege enemies

Greater Elites, Legendary enemies, Dread Knights, Doorbreakers, Siege Beasts, Siege Ghasts, Siege Boars, Architects, and selected boss/add mobs can receive siege behavior.

## 11.2 Breaching

Siege mobs may destroy a controlled whitelist of common defensive/building blocks such as:

- doors;
- trapdoors where appropriate;
- glass;
- planks/log-derived simple walls;
- cobblestone/stone-brick-type basic fortifications;
- selected fences/walls if required for pathing.

The whitelist must be conservative. Valuable/special blocks should not be casually erased.

**Obsidian is intentionally excluded** so serious endgame fortification retains value.

## 11.3 Architect construction

Architects must be able to:

- bridge short gaps;
- place cobblestone or another fixed temporary building material;
- pillar upward toward elevated players;
- create a practical route when the target is above or across a gap.

The platform-detection selector must use a genuine 3D search volume/range around the mob, not a narrow malformed selection column.

## 11.4 Cleanup philosophy

Age of Ruin should not become permanent random cobblestone spam. If feasible, temporary Architect blocks should either be limited in count/location or optionally cleaned after encounters.

---

# 12. BLOOD MOONS

Blood Moons are central high-risk/high-reward nights.

## 12.1 Scheduling

- Minimum separation: about 7 days.
- Probability increases with Threat.
- If no Blood Moon has occurred for about 12 days, force one.
- The new-day report must warn players in advance when one is scheduled.

## 12.2 Night behavior

During a Blood Moon:

- hostile spawn pressure approximately doubles;
- Elite chance ×2;
- mini-boss chance ×3;
- enemy effective HP +25%;
- enemy damage +20%;
- custom rare/Soul reward quantity or chance ×2;
- sleeping is disabled for the entire relevant night;
- Blood Moon mobs/events should be visually obvious through titles/particles/atmosphere where possible.

Sleeping must be disabled **before** players can skip the scheduled Blood Moon.

## 12.3 Gamerule preservation

If the datapack modifies `playersSleepingPercentage`, it should preserve the server's prior value and restore that value afterward rather than blindly setting it to 100.

---

# 13. WORLD EVENTS

World events are rolled every few days.

Scheduling target:

- at least ~3 days between events;
- roughly 35% roll when eligible;
- force an event after a long drought of about 7 days.

Nine event types:

| Event | Required behavior |
|---|---|
| **Undead Invasion** | Three waves attack one selected player/base; final mini-boss; completion reward |
| **The Hunt** | Elite odds significantly increased |
| **Nether Incursion** | Nether mobs pressure the Overworld / Nether-themed pulses |
| **Black Night** | Darkness for players and stronger hostile mobs |
| **Storm of Wrath** | Thunderstorm plus Stormborn/lightning pressure |
| **Plague** | Large share of Zombie-family mobs become Plague variants; regeneration/plague pressure |
| **Endstorm** | sustained Enderman/End pressure |
| **Bounty** | mini-boss odds significantly increased |
| **Blessing** | rare positive event; Luck and enhanced custom rewards |

Only one normal world event should be live at a time unless deliberately designed to stack with a Blood Moon.

---

# 14. UNDEAD INVASIONS

An invasion must attack **one selected target** for the whole encounter.

Recommended timing within one night:

- Wave I: immediately at event start;
- Wave II: ~2 minutes later;
- Wave III: ~5 minutes later.

This ensures all three waves can happen within a normal Minecraft night.

Wave structure:

- **Wave I:** large normal/variant mob pressure;
- **Wave II:** true Elites + Commander + siege pressure;
- **Wave III:** mini-boss / Dread Marshal or equivalent encounter finisher.

Wave-II enemies advertised as Elite must pass through actual Elite initialization: stats, 1–3 affixes, Soul drop, nametag, bonus loot.

Completing Wave III grants a meaningful reward.

---

# 15. NATURAL MINI-BOSSES

Mini-bosses are rare natural conversions/spawns. They are not merely large-HP normal mobs; each must have recurring mechanics.

Current 12-template roster:

1. **Grave Knight**
2. **Plaguebringer**
3. **Stormcaller**
4. **Abyssal Stalker**
5. **Dread Marshal**
6. **Frost Warden**
7. **Infernal Huntmaster**
8. **Rift Stalker**
9. **Plague Matron**
10. **Ashen Executioner**
11. **Bone Colossus**
12. **Void Reaver**

## 15.1 Grave Knight

Required mechanics:

- Wither Skeleton base;
- strong sword/shield-style identity;
- high projectile resistance/shield phase;
- telegraphed charge;
- undead summons;
- Execution effect against low-health players;
- Enrage below ~30% HP;
- guaranteed progression reward plus chance of high Protection book / unique weapon / netherite scrap.

## 15.2 Plaguebringer

- Zombie/Husk base;
- poison clouds;
- summons Plague Zombies;
- regenerates while plague adds survive;
- rewards **Plague Heart**.

## 15.3 Stormcaller

- Evoker base;
- lightning attacks;
- Vex summons;
- shockwave;
- teleportation;
- telegraphed major lightning attack;
- rewards **Storm Core**.

## 15.4 Abyssal Stalker

- Deep Dark encounter;
- Darkness;
- teleporting;
- sonic-style damage;
- misleading Warden/footstep audio;
- temporary light-disruption fantasy where practical;
- rewards **Abyssal Essence**.

## 15.5 Remaining mini-bosses

Each of the remaining eight templates must have at least one recurring attack mechanic plus one distinguishing passive/mobility/add mechanic. Their guaranteed progression reward must be delivered reliably independent of fragile offhand-equipment drop behavior.

---

# 16. MAJOR CUSTOM BOSSES

All custom bosses must:

- be summonable legitimately through their Sigil/altar flow;
- be summonable through admin testing functions;
- create a boss bar;
- detect nearby player count;
- scale durability using the multiplayer formula;
- track encounter participants while alive;
- execute recurring boss pulses/phases;
- scale adds to current Threat;
- clean up local encounter objects/add markers on defeat;
- grant guaranteed rewards directly to tracked participants;
- add the intended Corruption/Threat increase;
- remove boss bar and participant tags on completion.

## 16.1 Player-count scaling

Target formula:

**Boss effective HP = base × (1 + 0.65 × additional players)**

Approximation is acceptable when Minecraft health/absorption/Resistance constraints require it, but resulting effective durability must closely follow the target.

Damage should scale more conservatively than HP so multiplayer does not become disproportionately harder than solo.

## 16.2 Pre-Dragon major bosses

### Grave King

- Overworld undead sovereign.
- Wither Skeleton base.
- Charge/close-range pressure.
- Summons undead.
- Siege capable.
- Rewards Ancient Soul / Grave progression and **Gravecaller** relic line.

### Infernal Behemoth

- Nether-oriented heavy boss.
- Ravager-style base.
- Blaze/Magma Cube adds.
- fire AoE and environmental pressure.
- Reward must be an **Infernal** progression item, not Plague Heart.

### Abyss Watcher

- Deep Dark/Warden-style boss.
- Darkness and sonic attacks.
- Echo/Warden pressure.
- rewards **Abyssal Essence**.

### Leviathan

- Ocean/underwater boss.
- designed to make Respiration, Depth Strider, Tridents, Conduits, potions useful.
- tide/guardian pressure.
- rewards a Tide/Leviathan progression item and/or **Leviathan Plate** line.

## 16.3 Post-Dragon / post-Wither bosses

### Ancient Colossus

- huge armored construct fantasy;
- slow devastating attacks;
- four encounter-local weak points must be destroyed before the core can be damaged normally;
- weak points from another encounter can never interfere;
- rewards **Colossus Core**.

### Lich King

- necromancer encounter;
- multiple phases;
- summons undead hordes;
- encounter-local phylacteries must be destroyed;
- no global stale phylactery checks;
- rewards **Lich Phylactery** / Lich progression.

### Infernal Titan

- advanced Nether boss;
- large fire/lava attacks;
- Blaze/Magma pressure;
- environmental hazard focus;
- unique Titan reward delivered directly to participants.

### Warden King

- Deep Dark superboss;
- Darkness + sonic attacks;
- cannot be sensibly facetanked by only Prot X;
- unique **Warden Crown** reward delivered directly to participants.

### Storm Herald

- lightning/storm boss;
- Vex/Breeze adds;
- major lightning attacks;
- rewards **Storm Core** and/or **Stormbreaker** relic line.

### Void Emperor

- final custom superboss;
- End/Void theme;
- Darkness, teleporting, Shulker/Endermite pressure;
- designed for near-maximum endgame equipment;
- rewards **Voidpiercer** / final Void progression.

---

# 17. ENDER DRAGON — THE WORLD EATER

The Dragon must never check whether a player possesses Protection VIII, Power X, Netherite, or any boss key before taking damage.

Those are **balance expectations only**.

Target effective durability for four players: roughly **3,000–5,000+ HP equivalent**, with ~5,000 as the central target.

## Phase I — 100–75%

- normal End crystals remain relevant;
- stronger breath pressure;
- hostile Endermen pressure;
- baseline enhanced attacks.

## Phase II — 75–50%

- announce **THE END AWAKENS**;
- summon End guardians/adds;
- teleport/reposition mechanics;
- dangerous Void zones;
- crystals may respawn;
- reduced ranged effectiveness during appropriate windows.

## Phase III — 50–25%

- increased aggression;
- Elite Endermen;
- Shulkers;
- Endermites;
- projectile/barrage pressure.

## Phase IV — 25–0%

- announce **THE WORLD EATER ENRAGES**;
- no trivial regeneration loop;
- high damage;
- arena progressively less safe;
- large telegraphed breath attack;
- constant movement required.

## Dragon reward

- **Heart of the End**;
- unlocks/feeds the next enchantment and mythic progression tier;
- Dragon defeat adds major Corruption (~+15);
- reward must be delivered reliably to eligible participants/killer without relying on vanished entity position.

---

# 18. WITHER — HARBINGER OF EXTINCTION

The Wither is intentionally harder than the Dragon and also has **no gear permission gate**.

Expected practical gear is around Protection IX, Power XI, Sharpness X+ or stronger, but players are free to attempt it earlier.

Target effective durability for four players: roughly **8,000–9,000+ HP equivalent** across phases.

## Phase I

- enhanced airborne Wither;
- increased projectile pressure;
- properly scaled Wither Skeleton adds.

## Phase II — ~70%

- shield/ranged-resistance behavior;
- heads should create differentiated pressure where datapack mechanics permit;
- stronger Wither status pressure.

## Phase III — ~40%

- announce **DEATH HAS COME**;
- ground/crash shockwave;
- true Elite Wither Skeletons initialized through the actual Elite pipeline;
- dangerous Wither zones.

## Phase IV — ~15%

- no endless summons;
- no regeneration safety net;
- hyper-aggressive final phase;
- self-decaying or timed DPS-race pressure;
- massively increased threat.

## Completion

Only the tracked Age of Ruin Wither encounter should determine its completion. An unrelated Wither elsewhere in the world must not block or falsely trigger the reward.

---

# 19. LOOT PROGRESSION AND SOUL ECONOMY

Difficulty must produce meaningful reward progression.

## 19.1 Currency

### Soul Shard

Primary Elite currency.

### Greater Soul

Greater Elite/high-rarity currency.

### Ancient Soul

Boss/late-game currency.

Blood Moon and Blessing modifiers can increase Soul rewards.

## 19.2 Rarity bonus loot

### Elite

Typical bonus roll includes:

- gold;
- emeralds;
- lapis;
- low diamond chance;
- low high-Unbreaking book chance;
- XP;
- guaranteed Soul Shard progression.

### Greater Elite

Typical bonus roll includes:

- diamonds;
- emeralds;
- Sharpness ~VII book;
- Protection ~VI book;
- netherite scrap;
- Greater Soul.

### Epic

Typical rewards include:

- multiple diamonds;
- Sharpness XII book;
- Power XII book;
- Protection X book;
- netherite scrap/ingot.

### Legendary

Guaranteed high-value base reward plus one legendary roll, such as:

- `Legendary Reaver` sword (Sharpness ~25, Looting ~12, Unbreaking ~20);
- `Legendary Longbow` (Power ~25, Punch ~8, Unbreaking ~20);
- `Legendary Bulwark` chestplate (Protection ~20, Thorns ~10, Unbreaking ~20);
- Fortune XX book;
- Luck of the Sea XX book;
- netherite ingots.

### Mythic

Mythic equipment should generally not be a normal random mob drop. It should come from major bosses, exceptional fishing, or equivalent endgame systems.

---

# 20. ENCHANTMENT PROGRESSION TO LEVEL 255

Every vanilla enchantment in the targeted 26.2 build must have an Age of Ruin upgrade route to **level 255**.

The current canonical list is 43 enchantments:

1. Aqua Affinity
2. Bane of Arthropods
3. Binding Curse
4. Blast Protection
5. Breach
6. Channeling
7. Density
8. Depth Strider
9. Efficiency
10. Feather Falling
11. Fire Aspect
12. Fire Protection
13. Flame
14. Fortune
15. Frost Walker
16. Impaling
17. Infinity
18. Knockback
19. Looting
20. Loyalty
21. Luck of the Sea
22. Lunge
23. Lure
24. Mending
25. Multishot
26. Piercing
27. Power
28. Projectile Protection
29. Protection
30. Punch
31. Quick Charge
32. Respiration
33. Riptide
34. Sharpness
35. Silk Touch
36. Smite
37. Soul Speed
38. Sweeping Edge
39. Swift Sneak
40. Thorns
41. Unbreaking
42. Vanishing Curse
43. Wind Burst

## 20.1 Compatibility

The forge must respect item compatibility. A level upgrade may not put Sharpness on a helmet simply because a command can technically do so.

## 20.2 Enchanted books

Upgrade functions must support both:

- `minecraft:enchantments` on active equipment/items;
- `minecraft:stored_enchantments` on enchanted books.

## 20.3 Upgrade cost tiers

Each +1 level becomes more expensive:

| Current level | Cost for +1 |
|---:|---|
| 0–9 | 8 Soul Shards |
| 10–24 | 16 Soul Shards + 1 Greater Soul |
| 25–49 | 32 Soul Shards + 2 Greater Souls |
| 50–99 | 48 Soul Shards + 4 Greater Souls + 1 Ancient Soul |
| 100–199 | 64 Soul Shards + 8 Greater Souls + 2 Ancient Souls |
| 200–254 | 96 Soul Shards + 16 Greater Souls + 4 Ancient Souls |

No upgrade can exceed 255.

## 20.4 Villagers

Ordinary villager rerolling must not trivialize Age of Ruin progression. Villagers should remain effectively vanilla-capped for ordinary trades; extreme levels come from Age of Ruin systems.

## 20.5 Mechanical caveat

Some vanilla enchantments are binary or have mechanics that do not meaningfully scale with every numerical level. The pack may still store/display up to 255, but the audit must distinguish **stored level** from **actual vanilla effect scaling**.

---

# 21. PROTECTION / FORTIFICATION ENDGAME DEFENSE

Vanilla Protection has diminishing/capped practical behavior, so Age of Ruin needs a second defensive layer.

Combined Protection across armor should produce additional Resistance breakpoints, approximately:

| Combined Protection | Age of Ruin bonus |
|---:|---|
| 32–79 | Resistance II equivalent |
| 80–159 | Resistance III equivalent |
| 160+ | Resistance IV equivalent |

This makes full Protection VIII (32 combined) a meaningful early-endgame breakpoint.

## Fortification

Age of Ruin also defines a custom **Fortification** enchantment. Combined Fortification can increase max health in tiers, approximately:

- total 1–4: +10% max health;
- total 5–9: +20%;
- total 10–14: +30%;
- total 15+: +40%.

The player must not receive stale/stacking duplicate modifiers when armor changes.

---

# 22. ARCANE FORGE

The Arcane Forge is the conceptual workstation/progression interface for extreme enchantments.

Visual concept:

```text
     AMETHYST
        ↓
OBSIDIAN → ANVIL ← OBSIDIAN
        ↑
 ENCHANTING TABLE
```

The exact detection interface may use triggers/functions rather than literal item-entity crafting, but the player-facing concept is:

1. bring the compatible enchanted item/book;
2. possess required Soul currencies;
3. perform the forge upgrade;
4. consume exactly one tier cost;
5. increase the selected enchantment by exactly +1;
6. play clear success feedback;
7. refuse cleanly if incompatible, insufficient currency, absent enchantment, or already level 255.

---

# 23. LEGENDARY / MYTHIC FISHING

Fishing becomes an alternative extreme endgame progression route.

A qualifying fishing reward roll requires **Luck of the Sea 10+**.

## 23.1 Bonus reward chances

| Luck of the Sea | Legendary bonus chance | Mythic bonus chance |
|---:|---:|---:|
| 10–24 | 0.05% | 0% |
| 25–49 | 0.15% | 0.005% |
| 50–99 | 0.40% | 0.020% |
| 100–149 | 1.00% | 0.080% |
| 150–199 | 2.00% | 0.200% |
| 200–254 | 4.00% | 0.500% |
| 255 | 7.50% | 1.250% |

The Mythic roll is checked first; Legendary is then checked as the next bonus tier so one catch does not award both.

## 23.2 Legendary fishing pool

1. **Abyssal Harpoon** — Netherite Spear; Sharpness ~45, Lunge ~25, Unbreaking ~60, Mending.
2. **Tideglass Bow** — Power ~55, Punch ~15, Infinity, Unbreaking ~60.
3. **Leviathan Plate** — Netherite chestplate; Protection ~45, Thorns ~20, Unbreaking ~60, Mending.
4. **Pearl Rod** — Luck of the Sea ~70, Lure ~60, Unbreaking ~70, Mending.
5. **Stormhook** — Trident; Impaling ~60, Loyalty ~25, Channeling, Unbreaking ~60, Mending.
6. **Deepwalker Boots** — Protection ~40, Depth Strider ~40, Feather Falling ~30, Soul Speed ~20, Unbreaking ~60.
7. Luck of the Sea ~80 enchanted book.
8. Mending ~40 enchanted book.

## 23.3 Mythic fishing pool

1. **Rod of the Drowned King** — Luck 255, Lure 200, Unbreaking 255, Mending 255.
2. **Poseidon's Wrath** — Trident; Impaling 255, Loyalty 100, Channeling 255, Unbreaking 200, Mending 100.
3. **Ocean's End** — Netherite sword; Sharpness 180, Looting 120, Sweeping Edge 100, Unbreaking 200, Mending 100.
4. **Abyss Crown** — Netherite helmet; Protection 150, Respiration 120, Aqua Affinity 100, Thorns 80, Unbreaking 180, Mending 100.
5. **The Final Catch** — Netherite spear; Sharpness 255, Lunge 120, Fire Aspect 80, Looting 100, Unbreaking 220, Mending 100.
6. Luck of the Sea 255 enchanted book.

A Mythic catch must create a server-wide announcement and distinctive sound/visual feedback.

---

# 24. LEGENDARY RELICS AND UNIQUE PASSIVES

Boss/relic equipment should do something that raw enchantment levels cannot reproduce.

## Gravecaller

- Netherite sword identity.
- Killing an enemy has about a 10% chance to summon a temporary allied Gravebound Skeleton.
- Gravebound entity must be temporary and cleaned up.

## Stormbreaker

- Mace/axe-style storm relic.
- Successful hits have a chance to call lightning or trigger a storm strike.
- One physical hit may trigger the effect at most once.

## Voidpiercer

- Bow.
- High Power.
- Arrows can pierce/bonus-hit multiple enemies through custom logic.
- A single arrow collision may not apply the bonus every game tick; processed targets/projectiles need cooldown/tagging.

## Bloodletter

- Sword.
- Deals increased damage while wielder is below ~30% HP.

## Phoenix Plate

- Chestplate.
- Intended fantasy: once per ~30 minutes, otherwise lethal damage leaves the wearer alive at ~1 HP and ignites/damages nearby enemies.
- This must be implemented as reliably as vanilla datapack mechanics permit.
- If exact pre-death interception is impossible, the implementation/audit must explicitly document the approximation rather than claiming perfect lethal-hit interception.

---

# 25. SOUL FRACTURE DEATH PENALTY

Player death applies a temporary stacking Soul Fracture.

- each death: +1 stack;
- each stack: −5% max health;
- maximum 5 stacks / −25% max health;
- duration: approximately 10 minutes;
- killing Elites reduces the remaining recovery time;
- once the timer ends, the modifier is removed completely.

Repeated deaths should therefore discourage suicide-rushing bosses without turning the server into hardcore mode.

---

# 26. FRIENDLY / NEUTRAL SPECIAL MOBS

Potentially aggressive and neutral entities can independently roll special non-genetic variants. Current design uses roughly an 18% chance when first initialized for eligible neutral/special mobs.

## Iron Golem

- **Sentinel** — higher attack damage / active protection.
- **Bastion** — much higher HP and armor.
- **Arcane Golem** — nearby Haste support.

## Snow Golem

- **Frost Golem** — higher HP/movement and slows nearby hostiles.

## Copper Golem

- **Arcane Golem** — higher movement/health; utility/support fantasy.

## Bee

- **Royal Bee** — more HP/damage and nearby Regeneration support.

## Polar Bear

- stronger health, damage, knockback resistance.

## Pufferfish

- **Toxic Puffer** — stronger survivability / toxic identity.

## Camel Husk

- **Boneback** — durable mount/enemy.
- **Dunecharger** — faster.
- **Grave Caravan** — buffs nearby Zombie-family mobs.

## Zombie Horse

- **Dreadsteed** — durable.
- **Gravecharger** — faster.
- **Soulsteed** — buffs nearby Zombie-family speed.

## Allay

- **Arcane Allay** — faster and provides nearby Regeneration.

## Bat

- **Echo Bat** — much more durable / special cave identity.

## Parrot

- **Lucky Parrot** — higher HP and nearby Luck support.

## Squid

- **Ink Squid** — more durable / special aquatic identity.

## Glow Squid

- **Arcane Glow Squid** — more durable and nearby Water Breathing support.

## Skeleton Horse

- **Grave Horse** — faster and more durable.

## Villager

- **Blessed Villager** — more HP and nearby Hero of the Village support.

## Wandering Trader

- **Arcane Trader** — more HP and special identity; future trade integration may be added but must not be claimed unless implemented.

## Dolphin

- **Tide Dolphin** — faster and stronger Dolphin's Grace support.

---

# 27. GENETIC ANIMAL SYSTEM

Genetics is separate from the hostile-affix system.

## 27.1 Genetic species pool

The full current genetic pool contains:

1. Cow
2. Pig
3. Sheep
4. Chicken
5. Wolf
6. Horse
7. Donkey
8. Mule
9. Camel
10. Goat
11. Bee
12. Rabbit
13. Axolotl
14. Turtle
15. Frog
16. Cat
17. Ocelot
18. Fox
19. Llama
20. Trader Llama
21. Panda
22. Mooshroom
23. Sniffer
24. Armadillo
25. Nautilus
26. Strider

## 27.2 Standard genes

Nine ordinary combinable genes:

| Gene | Primary phenotype |
|---|---|
| **Vigor** | +50% max health, immediate healing/regen feedback |
| **Swift** | +30% movement speed |
| **Guardian** | +knockback resistance and approximately +60% attack damage where entity has an attack attribute |
| **Primal** | +25% health, +15% scale, +25% attack damage where applicable; enhanced normal loot chance |
| **Gourmet** | improved food-resource output |
| **Golden** | gold-related resource output |
| **Arcane** | magical materials or support effects |
| **Lucky** | emerald/diamond/rare reward potential or Luck support |
| **Alchemist** | potion/brewing resource output and special milking behavior on supported species |

## 27.3 Natural genes

New eligible animals can naturally roll genetic traits. The current design uses roughly an 18% chance to receive a natural gene when first initialized, with a small chance to gain an additional mutation.

## 27.4 Breeding inheritance

Genes are inherited independently.

For each gene eligible for that species:

- if one parent carries the gene: **55% inheritance chance**;
- if both parents carry the gene: **85% inheritance chance**;
- after inheritance: **3% normal mutation chance** for an extra species-valid gene.

This allows real multi-generation combinations such as:

`Golden Cow × Alchemist Cow → Golden + Alchemist descendants`

then:

`Golden + Alchemist × Vigor → Golden + Alchemist + Vigor descendants`

## 27.5 Hatch/egg species

Turtle, Frog, and Sniffer reproduction must not use a generic “nearest newborn” assumption at breeding time because they involve egg/spawn stages. Parent gene state must be stored and transferred safely to the eventual offspring without accidentally modifying an unrelated baby nearby.

## 27.6 Mule inheritance

Horse × Donkey breeding must correctly provide inheritance to Mule offspring.

---

# 28. SPECIES-VALID GENE SETS

Genes should only mutate/inherit where they make thematic/mechanical sense.

| Species | Valid ordinary genes |
|---|---|
| Armadillo | Golden, Lucky, Primal, Vigor |
| Axolotl | Arcane, Guardian, Lucky, Swift, Vigor |
| Bee | Arcane, Guardian, Lucky, Swift, Vigor |
| Camel | Lucky, Primal, Swift, Vigor |
| Cat | Arcane, Guardian, Lucky, Swift |
| Chicken | Arcane, Golden, Gourmet, Lucky, Swift |
| Cow | Alchemist, Arcane, Golden, Gourmet, Lucky, Vigor |
| Donkey | Lucky, Primal, Swift, Vigor |
| Fox | Arcane, Golden, Guardian, Lucky, Swift |
| Frog | Vigor, Arcane, Lucky, Primal |
| Goat | Alchemist, Guardian, Lucky, Primal, Vigor |
| Horse | Lucky, Primal, Swift, Vigor |
| Llama | Golden, Guardian, Lucky, Primal, Vigor |
| Mooshroom | Alchemist, Arcane, Gourmet, Lucky, Vigor |
| Mule | Lucky, Primal, Swift, Vigor |
| Nautilus | Arcane, Guardian, Lucky, Swift, Vigor |
| Ocelot | Arcane, Guardian, Lucky, Swift |
| Panda | Arcane, Lucky, Primal, Vigor |
| Pig | Arcane, Golden, Gourmet, Lucky, Swift, Vigor |
| Rabbit | Arcane, Golden, Gourmet, Lucky, Swift |
| Sheep | Arcane, Golden, Lucky, Swift, Vigor |
| Sniffer | Vigor, Lucky, Arcane, Primal |
| Strider | Golden, Lucky, Primal, Swift, Vigor |
| Trader Llama | Vigor, Guardian, Golden, Lucky, Primal |
| Turtle | Vigor, Golden, Arcane, Lucky |
| Wolf | Arcane, Guardian, Lucky, Primal, Swift, Vigor |

---

# 29. INVINCIBLE GENE

Invincible is the tenth special gene but follows completely different inheritance rules.

## 29.1 Acquisition

- cannot be inherited from either parent;
- both parents being Invincible gives no inheritance bonus;
- mutation-only;
- every bred eligible offspring independently rolls **1 in 400 / 0.25%** for Invincible.

## 29.2 Sterility

Once an Invincible animal reaches adulthood, it cannot breed. This prevents exponential immortal bloodlines.

## 29.3 Survivability

Invincible animals should be functionally impossible to kill through ordinary survival combat/environmental damage by combining:

- extremely high supported max health;
- extremely high absorption;
- Resistance IV-equivalent protection;
- Fire Resistance;
- continuous restoration/refill;
- visible Totem particles.

Administrative `/kill`, void/admin intervention, or engine-level exceptions do not need to be defeated.

## 29.4 Hit harvesting

When a player successfully hits an Invincible animal:

1. the animal survives;
2. a visible/sound feedback plays;
3. the animal drops a species-specific renewable resource package at its feet;
4. Gourmet/Golden/Arcane/Lucky/Alchemist genes add their extra reward packages if present;
5. one hit should produce one harvest event, not one event per game tick of HurtTime.

Examples:

- Cow → beef/leather package;
- Pig → pork package;
- Sheep → wool/mutton;
- Chicken → chicken/feathers;
- Horse/Donkey/Mule → leather;
- Bee → honeycomb;
- Turtle → scute;
- Armadillo → armadillo scute;
- Sniffer → ancient plant/seed resource;
- Strider → string;
- other species receive logical renewable packages.

## 29.5 Server announcement

A newly mutated Invincible offspring should produce a nearby/server message and Totem-style sound so the event feels exceptionally rare.

---

# 30. GENETIC DEATH LOOT

Every genetic animal must use a custom Age of Ruin death loot table that:

1. preserves the animal's normal vanilla loot;
2. adds gene-specific pools based on its tags;
3. stacks multiple genes independently;
4. does not require the killer to be a player unless deliberately specified.

Gene reward philosophy:

- **Gourmet:** additional cooked/species food.
- **Golden:** gold nuggets with smaller ingot chance.
- **Arcane:** lapis, amethyst, XP bottles, Soul/magic materials depending species.
- **Lucky:** emeralds and rare diamond/valuable rolls.
- **Alchemist:** redstone, glowstone, Nether Wart, Blaze Powder, potion-related resources.
- **Primal:** chance to roll normal vanilla loot a second time.
- physical genes (Vigor/Swift/Guardian) primarily affect the living animal rather than directly printing extra loot.

A multi-gene animal must produce all applicable additional pools.

---

# 31. ALCHEMIST MILKING / SPECIAL PRODUCTION

Supported Alchemist cows/mooshrooms or equivalent milkable animals can occasionally provide an infused potion in addition to normal milking behavior.

Potion pool can include:

- strong Regeneration;
- strong Strength;
- long Swiftness;
- Fire Resistance;
- Water Breathing;
- Night Vision.

The feature must not permanently break ordinary bucket/milk functionality.

---

# 32. NAMETAGS AND VISUAL READABILITY

## Hostiles

Only special Age of Ruin hostiles need persistent visible names.

Name should contain:

1. evolved variant or rarity identity;
2. every affix, separated clearly;
3. rarity color.

## Genetic animals

Name format should clearly list genes, e.g.:

- `Genes • Vigor • Golden`
- `Genes • Golden • Alchemist • Vigor`
- `Genes • Golden • Arcane • INVINCIBLE`

Invincible should be visually emphasized in gold or another exceptional color.

Normal unmodified animals should not be given noisy permanent nametags.

---

# 33. RESOURCE PACK — COMPLETE FUNCTIONAL SCOPE

The resource pack is not decorative-only. It is part of Age of Ruin's readability and item identity.

## 33.1 Required root files

- valid `pack.mcmeta` for resource-pack format 88.0;
- `pack.png`;
- README/version text;
- language file(s).

## 33.2 Custom item assets

Every custom datapack item-model reference must have a valid item definition, item model, and texture.

Current custom asset manifest:

1. Abyss Crown
2. Abyss Sigil
3. Abyssal Essence
4. Abyssal Harpoon
5. Ancient Soul
6. Bloodletter
7. Colossus Core
8. Colossus Sigil
9. Rod of the Drowned King
10. Grave Sigil
11. Gravecaller
12. Greater Soul
13. Heart of the End
14. Heart of Extinction
15. Infernal Sigil
16. Leviathan Plate
17. Lich Phylactery
18. Lich Sigil
19. Ocean's End
20. Pearl Rod
21. Phoenix Plate
22. Plague Heart
23. Poseidon's Wrath
24. Soul Shard
25. Storm Core
26. Storm Sigil
27. Stormbreaker
28. Tide Sigil
29. Tideglass Bow
30. Titan Sigil
31. Void Sigil
32. Voidpiercer
33. Warden Crown
34. Warden Sigil

## 33.3 Language entries

All custom currencies, sigils, boss components, relics, and custom enchantment names should have language entries where relevant.

## 33.4 Entity/boss visuals

A vanilla resource pack cannot simply attach arbitrary new entity models to NBT-tagged mobs in the same way a mod can. Therefore boss uniqueness should come from a combination of:

- visible custom names;
- custom carried/worn items;
- unique custom-item textures;
- particles;
- sounds;
- boss bars;
- display entities/model items where practical;
- vanilla entity silhouettes selected to match boss fantasy.

The pack must not claim to provide a completely new animated creature model unless an actual supported resource-pack/display implementation exists.

## 33.5 Resource-pack acceptance tests

For every `item_model='ruin:<id>'` in the datapack:

- item renders with the intended custom appearance;
- no missing-model purple/black texture;
- model JSON resolves;
- texture file resolves;
- PNG opens correctly;
- pack has no resource reload errors in client log.

---

# 34. BOSS SIGILS AND SUMMONING

Ten custom boss Sigils are craftable.

| Boss | Sigil | Key ingredients |
|---|---|---|
| Grave King | Grave Sigil | Bone + Soul Sand + Diamond |
| Infernal Behemoth | Infernal Sigil | Blaze Rod + Magma Cream + Diamond |
| Abyss Watcher | Abyss Sigil | Echo Shard + Sculk Catalyst + Diamond |
| Leviathan | Tide Sigil | Prismarine Crystals + Heart of the Sea + Diamond |
| Ancient Colossus | Colossus Sigil | Iron Block + Obsidian + Nether Star |
| Lich King | Lich Sigil | Bone Block + Fermented Spider Eye + Nether Star |
| Infernal Titan | Titan Sigil | Blaze Rod + Magma Block + Nether Star |
| Warden King | Warden Sigil | Echo Shard + Sculk Catalyst + Nether Star |
| Storm Herald | Storm Sigil | Lightning Rod + Breeze Rod + Nether Star |
| Void Emperor | Void Sigil | Ender Eye + Popped Chorus Fruit + Nether Star |

The legitimate summon flow must consume the Sigil only when a valid altar/summon action succeeds.

Admin functions must bypass crafting for testing but use the same underlying boss summon implementation.

---

# 35. ADMIN / QA TOOLING

At minimum administrators should have functions for:

- status;
- runtime status;
- daily message;
- give test currency;
- toggle siege;
- force/test Elite;
- test scaling;
- test genetic animal;
- test genetic loot;
- inspect assigned gene loot table;
- self-test bundle;
- summon each of the ten custom bosses.

Boss admin aliases:

- Grave King
- Infernal Behemoth
- Abyss Watcher
- Leviathan
- Ancient Colossus
- Lich King
- Infernal Titan
- Warden King
- Storm Herald
- Void Emperor

Admin testing must not create a parallel mechanic. Tests should call the same functions/pipelines used by natural gameplay wherever possible.

---

# 36. MIGRATION AND WORLD-SAFETY REQUIREMENTS

Every new release must support two situations:

## Fresh world

All objectives and global values are created from zero without requiring any previous version.

## Existing Age of Ruin world

Preserve:

- Age Day epoch/current progression;
- Corruption;
- first-diamond/Nether/netherite flags;
- relevant boss progression;
- genetic tags on existing animals;
- current custom loot assignments should be migratable/reassigned safely;
- existing special items remain valid where components/models are unchanged.

Version migration must use a new version marker so setup repairs are not skipped merely because an older release already initialized the world.

---

# 37. PERFORMANCE AND CLEANUP

The server is intended for four players, but the datapack must avoid pathological entity-wide work.

Requirements:

- hostile scans should target only uninitialized eligible entities;
- neutral scans should target only uninitialized eligible entities;
- timed affixes should not run expensive logic on every entity every tick when one-second pulses are sufficient;
- projectile logic must remove/mark processed projectiles to avoid repeated effects;
- temporary thralls/adds should despawn or be cleaned up;
- boss bars must be removed after encounters;
- participant tags must be cleared;
- weak points/phylacteries must be local and cleaned;
- stale event target tags must be removed;
- temporary siege construction should be bounded;
- no scheduled function should accidentally schedule duplicate permanent heartbeat chains after `/reload` or migration.

---

# 38. FULL END-TO-END ACCEPTANCE TEST MATRIX

The following tests define what “works” means.

## A. Pack loading

- Datapack appears enabled in `/datapack list`.
- No red datapack errors at server startup.
- Resource pack downloads and client accepts it.
- No client resource reload errors.

## B. Runtime

- Heartbeat increases over time.
- Day repetition value is correct.
- Day increments by exactly one after sleeping through one night.
- Daily message fires after sleep.

## C. Scaling

- Spawn a normal Zombie at known Threat.
- Inspect max HP and compare with +1.5%/Threat target.
- Verify melee damage modifier at +1%/Threat.
- Repeat in Nether and verify additional multiplier.

## D. Rarity

- Forced Elite shows green rarity and 1–3 affixes.
- Forced Greater shows blue rarity and 3–5 affixes.
- Forced Epic/Legendary have correct bonus stats and names.
- Killing each tier gives the correct currency and bonus loot.

## E. Affixes

Force a mob with all timed affixes under controlled conditions. Over 90 seconds verify Vampiric, Regenerating, Berserker, Stormborn, Venomous, Withering, Frostbound, Infernal, Explosive, Necromancer, Commander, Blinking, Reflective, Leeching, Hexed, Phasewalker, Executioner all independently fire.

## F. Variants

For every evolved family, force each variant once and verify at least one defining mechanic plus correct nametag.

## G. Siege

- place player behind door/basic wall; Siege mob eventually breaches.
- place player on platform; Architect constructs a viable upward route.
- confirm obsidian is not broken.

## H. Blood Moon

- force schedule;
- try sleeping before night and at night: skip must be prevented;
- verify Elite odds increase;
- verify mini-boss odds increase;
- verify HP/damage buffs;
- verify additional spawn pressure;
- verify doubled custom rewards;
- verify prior sleeping gamerule restored after event.

## I. World events

Force all nine events individually and verify each event's advertised effect.

## J. Invasion

- one player selected;
- all waves around same player;
- Wave II arrives ~2 min;
- Wave III ~5 min;
- true Elites in Wave II;
- mini-boss in Wave III;
- reward on completion.

## K. Nether

- warband stays around one target;
- forced Greater Elite has proper final HP/name/loot;
- Soul Storm stays around one target;
- Nether variants spawn/operate.

## L. Mini-bosses

Force all 12; each must show boss identity, recurring mechanic, correct reward, and cleanup.

## M. Custom bosses

For each of 10:

- legit Sigil craft;
- valid summon;
- boss bar;
- 1/2/3/4-player scaling;
- participant tracking;
- mechanics fire;
- adds scale;
- guaranteed reward delivered to participants;
- +5 Corruption if intended;
- cleanup complete.

## N. Dragon

- no gear gate;
- phase thresholds occur;
- adds scale;
- Heart of End reward;
- +15 Corruption;
- unrelated entities do not corrupt completion logic.

## O. Wither

- no gear gate;
- harder than Dragon in practical test;
- four phase transitions;
- Elite adds are true Elites;
- another unrelated Wither does not block reward;
- completion reward correct.

## P. Enchantments

For all 43 enchantments:

- valid item upgrade succeeds;
- invalid item refuses;
- enchanted book upgrade succeeds using stored enchantments;
- exact cost tier consumed;
- +1 level only;
- 255 refuses further upgrade.

## Q. Protection/Fortification

Equip known total levels and verify correct Resistance/max-health breakpoint; remove armor and verify bonuses disappear/adjust immediately.

## R. Fishing

At controlled Luck levels test reward thresholds statistically or by forced RNG harness; verify all 8 Legendary and all 6 Mythic rewards render correctly.

## S. Relics

- Gravecaller can summon temporary ally on kill.
- Stormbreaker procs once per hit.
- Voidpiercer never multi-procs each tick on same collision.
- Bloodletter damage changes below threshold.
- Phoenix Plate cooldown initializes and can proc on first eligible event.

## T. Soul Fracture

Die five times; verify −25% cap. Kill Elites and verify time reduction. Wait expiry and verify full max-health restoration.

## U. Genetics

- naturally spawn/initialize each eligible species;
- verify species-valid genes only;
- one-parent 55% inheritance statistically;
- two-parent 85% statistically;
- 3% mutation statistically;
- genes combine independently;
- Turtle/Frog/Sniffer hatch inheritance targets correct child;
- Mule inheritance works.

## V. Invincible

- 1/400 breeding mutation through forced RNG/test hook;
- never inherited directly;
- adult cannot breed;
- survival attacks cannot kill it;
- each hit causes one species drop event;
- extra economic genes add extra drops;
- nametag shows INVINCIBLE.

## W. Genetic loot

For all 26 species, kill a multi-gene test animal and verify vanilla loot plus all applicable extra gene pools.

## X. Neutral specials

Force every neutral special variant and verify stats/aura/behavior.

## Y. Resource pack

Generate every custom item and visually inspect all 34 model/texture pairs in game.

---

# 39. CURRENT v3.5 / RESOURCE-PACK v3.1 KNOWN DIVERGENCES TO FIX BEFORE “FINAL” STATUS

This section records issues discovered during static audit that violate the specification above. A future consolidated build should clear all of them before being called production-ready.

1. `minecraft:small_sulfur_cube` still appears in the scalable-hostiles tag and must be removed.
2. Infernal fire-trail logic references `#minecraft:fire_igniter` as a block tag; replace with a validated block condition/whitelist.
3. Soul Storm event currently risks choosing separate random players for individual spawns; use one target anchor.
4. Infernal Warband promotion/finalization order can produce incorrect Greater-Elite final HP/name; promote before finalizing.
5. Architect upward-player detection needs a genuine surrounding 3D range instead of malformed `dy`-style volume logic.
6. Storm Archer projectiles must be marked after impact so one arrow cannot call lightning repeatedly every tick.
7. Necrotic Archer impact must also be one-shot per projectile/target collision.
8. Voidpiercer must not apply repeated bonus damage every tick while overlapping a target.
9. Parched CustomName format must match the modern component format used elsewhere.
10. Parched variant probability must not become effectively 100% after its threshold.
11. Dragon/Wither/custom-boss adds spawned already as `ruin.initialized` can bypass current Threat scaling; all intended adds must be scaled explicitly or passed through the pipeline.
12. Wither phase adds labeled Elite must be real initialized Elites, not a tag-only approximation.
13. Custom boss guaranteed progression rewards should not rely solely on offhand-equipment drops, especially on entities where equipment behavior is fragile.
14. Infernal Behemoth currently has a mismatched Plague Heart reward; replace with an Infernal progression reward.
15. Ancient Colossus, Leviathan, Infernal Titan, Warden King, Lich King, and Void Emperor need guaranteed participant-delivered signature rewards rather than relying on equipment drops.
16. Mini-boss guaranteed Ancient Soul/progression rewards must likewise be independent of fragile equipment slots.
17. Phoenix Plate exact lethal-hit interception is constrained by vanilla datapack timing; either improve implementation or explicitly retain/document the approximation.
18. Blood Moon must preserve and restore the server's prior sleeping-percentage gamerule instead of forcing 100 afterward.
19. Boss/miniboss/event minions must have a consistent, documented Threat-scaling rule rather than arbitrary pre-initialized status.
20. Resource-pack README/version string should be synchronized with the actual distributed pack version.
21. Every new datapack custom item added after resource-pack v3.1 must be checked against the 34-item resource asset manifest before release.

---

# 40. DEFINITION OF DONE

Age of Ruin is only “fully implemented” when all of the following are true:

- the complete runtime works after a fresh install;
- migration works from the previous release;
- daily messages work through natural day passage and sleeping;
- all eligible natural hostiles actually receive scaling;
- all rarity tiers and all 22 affixes work live;
- all hostile families listed above have working special behavior;
- Nether-specific systems work;
- siege/building behavior works;
- all nine world events work;
- all 12 mini-boss templates work;
- all 10 custom bosses work end-to-end;
- Dragon and Wither work through every phase without gear locks;
- rarity/boss rewards are reliable;
- Soul Fracture works;
- all 43 enchantments upgrade both equipment and books to 255;
- high Protection/Fortification creates meaningful defensive progression;
- Legendary/Mythic fishing works with every listed reward;
- all relic passives work without repeat-proc bugs;
- all 26 genetic species work;
- inheritance, mutation, and Invincible rules work;
- genetic death loot works for every species;
- neutral special mobs work;
- every datapack `ruin:` item model exists and renders through the resource pack;
- no unresolved function/model/texture references exist;
- no invalid 26.2 entity IDs/predicates/components remain;
- the datapack boots on an actual Java 26.2 server with zero pack-loader errors;
- a controlled multiplayer gameplay test confirms the major systems above.

The governing design principle remains:

**Difficulty ↑ → Reward ↑ → Player Power ↑ → New Difficulty ↑**

The datapack must never collapse into merely giving normal mobs huge HP while the rest of the game remains unchanged.
