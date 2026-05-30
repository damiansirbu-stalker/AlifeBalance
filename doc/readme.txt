AlifeBalance: A-Life balance layer for STALKER Anomaly, by Damian
Version: 1.0.2 (xlibs 1.7.0)
GitHub: https://github.com/damiansirbu-stalker/AlifeBalance
Changelog: https://github.com/damiansirbu-stalker/AlifeBalance/blob/main/doc/changelog
Architecture: https://github.com/damiansirbu-stalker/AlifeBalance/blob/main/doc/architecture.md
Russian / Na russkom: https://github.com/damiansirbu-stalker/AlifeBalance/blob/main/doc/readme_ru.txt
Bugs, suggestions: https://github.com/damiansirbu-stalker/AlifeBalance/issues

! Reset MCM settings to defaults after updating !

AlifeBalance is a mod composed of several systems, that tries to achieve a balanced and seamless gameplay.

Smart Balance:
  Vanilla respawn cooldowns run on a fixed schedule that ignores combat pressure. Regions you fight through empty out while quieter parts of the map stay full.
  Smart Balance counts deaths per region and faction, and once enough have piled up it accelerates the respawn cooldown at one matching smart terrain on that map, choosing the one farthest from you.
  Heavy combat brings refills back faster, on the level where it happened, off-screen.

Inventory Balance:
  Vanilla looting works fine in isolation, but long-lived NPCs keep picking up corpse gear and never stop, so their inventories grow into hoards that affect game balance, performance, and save size.
  A lightweight scanner walks online stalkers across frames and trims each one at most once per game-day. Traders are skipped because their inventory is their stock.
  The result is that vanilla loot system works fully, NPCs have items to use and trade, and the risk of over accumulation is gone.

MCM (General):
  Smart Balance: enabled, advances to floor, min minutes.
  Inventory Balance: enabled, NPCs per frame, NPCs per cycle, rescan cooldown.
  Development: log level, map markers, show status, reset counters.

Inventory Balance policy:
  Per-category max ceilings live in configs/alifebalance/ab_inventory_policy.ltx. Anything above the max is released. Defaults cover equipped ammo (rounds), grenades, consumables (medkit / bandage / antirad / stim / pill / food / drink), and gear (weapons / outfits / helmets / artefacts / crafting / device). Quest items, runtime-story_id items, companion-gifted items, and player-strapped weapons are protected by xinventory and never touched. DLTX-overridable for modpack integrators.

Compatibility:
  Tested with vanilla Anomaly 1.5.3, GAMMA, ZCP, Redone, Warfare, AlifeGuard, AlifePlus, Night Mutants, Nocturnal Mutants, GAMMA Dynamic Despawner, Guards Spawner.
  Inventory Balance requires vanilla NPC corpse looting to be enabled. Disable any mod that blocks looting (such as "NPC Stop Looting Dead Bodies") for the full effect.

Companion mods:

AlifePlus (reactive A-Life framework) -- https://www.moddb.com/mods/stalker-anomaly/addons/alifeplus-v1-0-01 | https://www.nexusmods.com/stalkeranomaly/mods/105
AlifeGuard (entity hygiene) -- https://www.moddb.com/mods/stalker-anomaly/addons/alifeguard-1001 | https://www.nexusmods.com/stalkeranomaly/mods/104

Requirements:
Anomaly 1.5.3
xlibs 1.7.0+ (https://www.moddb.com/mods/stalker-anomaly/addons/xlibs-1001)
MCM

Install (MO2):
1. Install xlibs
2. Install AlifeBalance
3. Load order does not matter
4. Configure via MCM

Uninstall (MO2):
Disable or remove in MO2.

Architecture and validation:
Runs on xlibs over the X-Ray engine, using runtime callbacks only and leaving base scripts and the engine binary untouched. See doc/architecture.md for the full design and the multi-stage validation pipeline (luacheck, selene, AST analysis, contract rules, integration tests).

Credits:
Altogolik - support, ideas, source materials.

Usage and License:
  Modpacks: allowed and encouraged. Keep the readme and license files.
  Addons, patches, integrations: allowed. Credit "AlifeBalance by Damian Sirbu" visibly on your mod page.
  Reproducing the implementation in other software: not allowed, even with credit.
  Full license in LICENSE file and on GitHub.

Keep the Zone alive while letting vanilla A-Life remain vanilla.
