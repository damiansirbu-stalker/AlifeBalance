AlifeBalance: A-Life balance layer for STALKER Anomaly, by Damian
Version: 1.1.2 (xlibs 1.8.2, demonized 20250908)
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

AlifeBalance is a balance layer for vanilla A-Life:
  - Smart Balance: shortens respawn cooldowns only where combat actually happened.

It runs alongside the engine without rewriting it.
Inventory bounding (formerly Inventory Balance) now lives in AlifeGuard as Inventory Guard.

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


Compatibility:
  Requires xlibs.
  Runs on themrdemonized modded exes 2025.9.10 or newer, or AOEngine v0.55 or newer.
  The full feature set needs the latest demonized build. A feature that needs a newer build stays inactive on older exes.

  Tested with vanilla Anomaly 1.5.3, GAMMA, ZCP, Redone, Warfare, AlifeGuard, AlifePlus.
  Also tested with Night Mutants, Nocturnal Mutants, GAMMA Dynamic Despawner, Guards Spawner.

  Conflicts (critical): none. Vanilla and ZCP share the respawn-cooldown field Smart Balance writes; they compose.

  Affects / coexists (Smart Balance):
  - Vanilla, ZCP: same cooldown field; compose.
  - Redone, GAMMA NPC Spawns: pure config; Smart Balance writes runtime state on a different layer.
  - Night Mutants: engine spawn path; squads register against their origin smart.
  - Nocturnal Mutants: spawn outside smart terrains; no interaction.
  - ReSpawn Mutant Collection: composes if squads register against an origin smart.
  - Dynamic Despawner, AlifeGuard: despawn without firing death; never trigger advances.


MCM:
  Smart Balance tab: enable, advance count, minimum cooldown remaining.
  Development tab: log level, map markers.

  Map markers (Development): green PDA spots appear on every smart terrain that received an advance.
  They linger 5 real-time minutes. Right-click any marker to teleport to that smart or display its full advance history.
  Independent of log level.


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
Runs on xlibs over the X-Ray engine, using runtime callbacks only and leaving base scripts and the engine binary untouched.
See doc/architecture.md for the full design and the multi-stage validation pipeline (luacheck, selene, AST analysis, contract rules, integration tests).


FAQ:
Do I need modded exes?
  Yes. AlifeBalance needs themrdemonized modded exes (2025.9.10 or newer) or AOEngine (v0.55 or newer). Vanilla Anomaly does not expose the APIs it relies on.

Credits:
Altogolik - support, ideas, source materials.


Usage and License:
  Modpacks: allowed and encouraged. Keep the readme and license files.
  Addons, patches, integrations: allowed. Credit "AlifeBalance by Damian Sirbu" visibly on your mod page.
  Reproducing the implementation in other software: not allowed, even with credit.
  Full license in LICENSE file and on GitHub.


Keep the Zone alive while letting vanilla A-Life remain vanilla.

Reporting issues and suggestions
Open a bug report or a suggestion at https://github.com/damiansirbu-stalker/AlifeBalance/issues/new/choose.
Also discussed on the GAMMA, EFP, Anomaly, and Zona Discord servers.

Before posting, read this readme and the MCM options.

Include:
- Exact steps to reproduce, from a new game or a named save, with expected and actual result.
- xray.log and the mod debug log (MCM log level DEBUG), plus engine build, modlist, load order.
- Describe the behavior. With hundreds of mods and overrides, only the log shows whether this mod was involved and what caused it.
