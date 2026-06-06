AlifeBalance: A-Life balance layer for STALKER Anomaly, by Damian
Version: 1.1.0 (xlibs 1.7.0, demonized 20260601)
GitHub: https://github.com/damiansirbu-stalker/AlifeBalance
Changelog: https://github.com/damiansirbu-stalker/AlifeBalance/blob/main/doc/changelog
Architecture: https://github.com/damiansirbu-stalker/AlifeBalance/blob/main/doc/architecture.md
Russian / Na russkom: https://github.com/damiansirbu-stalker/AlifeBalance/blob/main/doc/readme_ru.txt
Bugs, suggestions: https://github.com/damiansirbu-stalker/AlifeBalance/issues

Alife Collection:
AlifePlus: https://www.moddb.com/mods/stalker-anomaly/addons/alifeplus-v1-0-01
AlifeBalance: https://www.moddb.com/mods/stalker-anomaly/addons/alifebalance
AlifeGuard: https://www.moddb.com/mods/stalker-anomaly/addons/alifeguard-1001
AlifeTactics: https://www.moddb.com/mods/stalker-anomaly/addons/alifetactics

! Reset MCM settings to defaults after updating !

AlifeBalance is a balance layer for vanilla A-Life. It contains multiple systems:
  - Smart Balance: shortens respawn cooldowns only where combat actually happened.
  - Inventory Balance: bounds NPC inventory hoarding so anti-loot addons are no longer needed.

Both run alongside the engine without rewriting it. Each can be toggled independently.

Built as a companion to AlifePlus, which adds reactive A-Life behavior across the Zone.
More activity means more deaths, and AlifeBalance scales refill to match.
Both mods are independent and can be used separately.


Smart Balance:
  Vanilla respawn cooldowns run on a fixed schedule that ignores combat pressure.
  Regions you fight through empty out while quieter parts of the map drift toward overcrowding.

  Smart Balance counts deaths by level and faction.
  Once enough have accumulated, it shortens the cooldown at one matching smart terrain on that level.
  The threshold is the largest possible squad size for that faction's spawn pool.
  The next respawn there gets pulled closer in time, but never all the way to immediate.
  The picker chooses the smart terrain farthest from you so the refill lands off-screen.

  Why not just shorten the global cooldown to 1h or 2h?
    The engine's respawn_idle is one number shared by every smart terrain in the entire Zone.
    Shorten it and the world cycles faster everywhere at once, including maps you will never visit this session.
    The actor-distance gate blocks the spawn only while you stand inside a smart's radius.
    The moment you walk out, the overdue cooldown fires and the squad lands at your back.
    Smart Balance instead applies one targeted advance per threshold of deaths, on the level where the kills happened.
    The picker chooses the eligible smart farthest from you, so refills land away from your current combat, one squad at a time.

  What you'll notice:
    Active regions recover faster after heavy firefights.
    Inactive regions stop drifting toward overcrowding.
    Mutant lairs and stalker bases repopulate at a pace closer to the kill rate.
    Refills land at the far end of the level, off-screen, one squad at a time.
    Vanilla A-Life still owns every spawn.

  Important:
    Smart Balance never creates NPCs.
    It shifts a timestamp the engine was always going to read on its next alife tick.
    The engine still owns spawning, recipes, squad selection, and budget caps.

  Example:
    A firefight kills 30 bandits on Cordon in one minute.
    Cordon bandit squads max out at 3, so 10 separate adjustments are available.
    Smart Balance picks up to 10 different bandit-spawning Cordon smart terrains and shortens each one's wait.
    The refills land over the next couple of game-hours as the engine reaches each smart's now-shorter wait.
    If every eligible smart is already at population cap or already at the floor, the adjustments are held and the kills carry over to the next check.

  Settings (MCM, Smart Balance tab):
    Advance count (1-8, default 4): how many combat bursts it takes to fully accelerate one smart terrain.
    Lower means each burst matters more; higher paces acceleration across longer combat.
    Minimum cooldown remaining (10-360 game-minutes, default 120): the floor Smart Balance never pushes below.
    The engine ages the final wait out on its own clock.
    Higher leaves more delay between combat and refill.

  Presets:
    Aggressive (one burst = full advancement): advance count 1, minimum cooldown 60.
    Default (one burst = 25% advancement): advance count 4, minimum cooldown 120.
    Conservative (one burst = 12.5% advancement): advance count 8, minimum cooldown 360.


Inventory Balance:
  Vanilla NPCs loot corpses, and over a long session this builds up two costs.
  A long-lived stalker carries a trader run of gear, so killing them becomes a jackpot.
  Every looted item is also a permanent alife server object, and the engine tracks at most around 65000 of those.
  Long saves drift toward the cap and the game eventually slows, then crashes.

  Why not just disable NPC looting?
    Mods like NPC Stop Looting Dead Bodies and BoltBeGone sidestep the problem by blocking the engine's loot path.
    That fixes the symptom but stalkers no longer loot their kills, which is one of the things that makes A-Life feel alive.
    Inventory Balance keeps vanilla looting on and bounds the cause instead.
    A lightweight scanner walks online stalkers in small batches and releases anything above each category's limit.
    Killing a stalker still yields what they actually need to carry, not what they have accumulated over 50 game-days.

  What you'll notice:
    Long-lived stalkers carry a believable loadout instead of a trader-sized hoard.
    Killing a random stalker yields reasonable loot, not a vendor run.
    Saves stay performant across long sessions.
    Companions, story characters, traders, and named NPCs are never touched.
    Vanilla looting still works. Stalkers loot their kills.

  Important:
    Inventory Balance never spawns items.
    It releases what NPCs already accumulated above the per-category limits.
    Quest items, equipped gear, story_id items, companion gifts, and player-strapped weapons are always preserved.

  Example:
    An online stalker on Cordon has been alive for 50 game-days.
    They have picked up 47 medkits, 23 bandages, 18 grenades, and 600 rounds of mismatched ammo.
    The scanner's next visit releases 42 medkits (cap 5), 18 bandages (cap 5), 15 grenades (cap 3), and the 600 mismatched rounds (cap 0).
    The NPC walks around with a believable load: a few medkits, a stack of bandages, three grenades, and ammo for the gun they actually carry.

  Settings (MCM, Inventory Balance tab):
    NPCs per frame (1-10, default 1): how many NPCs the scanner trims in a single frame.
    The default keeps every frame cheap. Raise only if the scanner cannot keep up on very large saves.
    Rescan cooldown (1-72 game-hours, default 24): an NPC is not rescanned until this many game-hours have passed.
    Lower trims more aggressively against fast hoarders. Higher reduces overall work.

  Policy:
    Per-category limits live in configs/alifebalance/ab_inventory_policy.ltx (DLTX-overridable).
    Defaults cover equipped ammo (in rounds, per tier), grenades, and consumables: medkit, bandage, antirad, stim, pill, food, drink.
    Gear coverage: weapons, outfits, helmets, artefacts, crafting items, devices.
    Quest items, equipped gear, story_id items, companion gifts, and player-strapped weapons are protected by xlibs and never touched.


Compatibility:
  Tested with vanilla Anomaly 1.5.3, GAMMA, ZCP, Redone, Warfare, AlifeGuard, AlifePlus.
  Also tested with Night Mutants, Nocturnal Mutants, GAMMA Dynamic Despawner, Guards Spawner.

  Smart Balance:
    Vanilla and ZCP read the same cooldown field Smart Balance writes.
    Redone and GAMMA NPC Spawns ship pure config. Smart Balance writes runtime state on a different layer.
    Night Mutants spawns through the engine's own path. Squads still register against their origin smart.
    Nocturnal Mutants spawns outside smart terrains. No interaction.
    Dynamic Despawner and AlifeGuard despawn without firing the death event. Never trigger advances.

  Inventory Balance:
    Inventory Balance manages vanilla NPC looting at the source, so anti-loot addons are no longer needed.
    Disable any (NPC Stop Looting Dead Bodies, BoltBeGone) for the full effect.
    Weapons Drop on Bodies is unrelated and compatible. It only changes where a dying NPC's active weapon ends up (corpse inventory vs floor) and does not block NPC looting.


MCM:
  Smart Balance tab: enable, advance count, minimum cooldown remaining.
  Inventory Balance tab: enable, NPCs per frame, rescan cooldown.
  Development tab: log level, map markers.

  Map markers (Development): green PDA spots appear on every smart terrain that received an advance.
  They linger 5 real-time minutes. Right-click any marker to teleport to that smart or display its full advance history.
  Independent of log level.


Requirements:
Anomaly 1.5.3
demonized 20260601+ (https://github.com/themrdemonized/xray-monolith)
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
Runs on xlibs over the X-Ray engine, using runtime callbacks only and leaving base scripts and the engine binary untouched.
See doc/architecture.md for the full design and the multi-stage validation pipeline (luacheck, selene, AST analysis, contract rules, integration tests).


Credits:
Altogolik - support, ideas, source materials.


Usage and License:
  Modpacks: allowed and encouraged. Keep the readme and license files.
  Addons, patches, integrations: allowed. Credit "AlifeBalance by Damian Sirbu" visibly on your mod page.
  Reproducing the implementation in other software: not allowed, even with credit.
  Full license in LICENSE file and on GitHub.


Keep the Zone alive while letting vanilla A-Life remain vanilla.
