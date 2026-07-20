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
  - Smart Balance: keeps each map's population near what the map's own spawn configs declare.

It runs alongside the engine without rewriting it.
Inventory bounding (formerly Inventory Balance) now lives in AlifeGuard as Inventory Guard.

Built as a companion to AlifePlus, which adds reactive A-Life behavior across the Zone.
More activity means more losses, and AlifeBalance steers recovery back to each map's designed population.
Both mods are independent and can be used separately.


Smart Balance:
  Vanilla respawn cooldowns run on a fixed schedule that ignores the state of the world.
  A faction you wipe out waits its full turn while an untouched faction keeps cycling, and maps drift away from how they were designed.

  Smart Balance runs a rolling census: it counts live NPCs per map for every faction the map can spawn, with all mutants counted as one group.
  Each map's spawn configs already declare how many of each group it is meant to hold; that declared population is the target.
  A group below its target gets its respawn cooldowns advanced, and its squads that fell below their own minimum size
  are gradually refilled, one member at a time, far from the player and never in crowded areas.
  Corrections pick their smart terrains: only ones that can spawn the depleted group and still have spawn budget,
  never ones in already-crowded areas, so recovery flows toward the empty parts of a map.
  A group above its target gets its cooldowns delayed, never past a configurable ceiling (default 12 game-hours; one full vanilla cooldown binds when shorter), and spawns are never blocked.
  Groups near their target are left completely alone.

  What you'll notice:
    Massacre a faction and it returns first, while overgrown factions idle.
    Squads ground down to lone survivors regain their members instead of wandering the map forever as one-man ghosts.
    Mutant maps stay mutant-heavy and war zones stay contested; each map drifts back to its designed character.
    A depleted map as a whole recovers faster; a healthy map behaves exactly like vanilla.
    Vanilla A-Life still owns every spawn.

  Important:
    Smart Balance never blocks a spawn and never removes an NPC.
    The timing lever shifts a timestamp the engine was always going to read on its next alife tick.
    The refill only adds members to depleted squads, up to the squad's own configured minimum, never beyond it,
    and only while the squad is offline and its group is below the map's declared population.
    The engine still owns spawning, recipes, squad selection, and budget caps.

  Example:
    A firefight wipes the bandits on Cordon.
    The next census pass sees bandits far below Cordon's declared bandit population.
    Every Cordon smart terrain that can spawn bandits gets its cooldown advanced one step per pass,
    and surviving bandit squads that dropped below their minimum size regain a member each pass.
    Recovery arrives over the next game-hours as the engine reaches each shortened wait; once bandits are back near target, Smart Balance goes silent.
    If instead the map is overrun (say a mod flooded it), those cooldowns are delayed up to the configured ceiling until the census clears.

  Settings (MCM, Smart Balance tab):
    Correction steps to floor (1-8, default 4): how many census passes carry one smart terrain from full cooldown to the floor.
    Lower corrects harder per pass; higher corrects more gradually. Delays use the same step size.
    Minimum cooldown remaining (10-360 game-minutes, default 120): the floor the advance direction never pushes below.
    The engine ages the final wait out on its own clock.
    Maximum cooldown remaining (60-1440 game-minutes, default 720): the ceiling the delay direction never pushes past.
    One full vanilla cooldown stays the bound when it is shorter than the ceiling.
    Refill depleted squads (default on): squads below their own minimum size regain one member per census pass while their group is under target.

  Presets:
    Aggressive (one pass = full advancement): correction steps 1, minimum cooldown 60.
    Default (one pass = 25% advancement): correction steps 4, minimum cooldown 120.
    Conservative (one pass = 12.5% advancement): correction steps 8, minimum cooldown 360.


Compatibility:
  Requires xlibs.
  Runs on themrdemonized modded exes 2025.9.10 or newer, or AOEngine v0.55 or newer.
  The full feature set needs the latest demonized build. A feature that needs a newer build stays inactive on older exes.

  Tested with vanilla Anomaly 1.5.3, GAMMA, ZCP, Redone, AlifeGuard, AlifePlus.
  Also tested with Night Mutants, Nocturnal Mutants, GAMMA Dynamic Despawner, Guards Spawner.

  Conflicts (critical): Warfare. Its population model fights any external balancing; disable AlifeBalance when running Warfare.

  Superseded: Squad Filler. It tops offline stalker squads up to a flat size once per game-day, regardless of how crowded or populated the map is.
  Smart Balance's refill covers the same squads gated by the population census, skips crowded areas, works for mutants too,
  and respects each squad's own configured size. Disable Squad Filler when running AlifeBalance.

  Affects / coexists (Smart Balance):
  - Vanilla, ZCP: same cooldown field and gate; compose. ZCP keeps deciding which species or faction actually spawns and how squad sizes scale;
    the population target accounts for that scaling, and Smart Balance only times recovery and repairs depleted squads.
  - Redone, GAMMA NPC Spawns: pure config; that config IS the declared population Smart Balance steers toward.
  - AlifePlus: territory conquest and infestation change what a smart spawns; the census target follows those changes automatically.
  - Night Mutants: engine spawn path; squads are counted by the census.
  - Nocturnal Mutants: spawn outside smart terrains; no interaction.
  - Dynamic Despawner, AlifeGuard: despawns lower the census like any other loss; recovery follows, and the refill skips crowded areas so it never fights a density cull.


MCM:
  Smart Balance tab: enable, correction steps, minimum cooldown remaining, maximum cooldown remaining, refill depleted squads.
  Development tab: log level, map markers.

  Map markers (Development): green PDA spots appear on every smart terrain that received a cooldown advance or delay.
  They linger 5 real-time minutes. Right-click any marker to teleport to that smart or display its full correction history.
  Independent of log level.


Requirements:
Anomaly 1.5.3
xlibs 1.8.2+ (https://www.moddb.com/mods/stalker-anomaly/addons/xlibs-1001)
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
