Version: 1.1.5-snapshot (xlibs 1.9.0, demonized 20250908)
Changelog: https://github.com/damiansirbu-stalker/AlifeBalance/blob/main/doc/changelog
Health: https://damiansirbu-stalker.github.io/AlifeBalance/health/
JitProfiler: https://damiansirbu-stalker.github.io/AlifeBalance/jitprofiler/
Bugs: https://github.com/damiansirbu-stalker/AlifeBalance/issues
Russian / На русском: https://github.com/damiansirbu-stalker/AlifeBalance/blob/main/doc/readme_ru.txt

My work:
GitHub: https://github.com/orgs/damiansirbu-stalker/repositories
ModDB: https://www.moddb.com/members/damian-sirbu/addons
Nexus: https://www.nexusmods.com/profile/damiansirbu/mods

My contributions:
X-Ray Monolith: https://github.com/themrdemonized/xray-monolith

[ Hero image: alifebalance-hero.gif - each map recovers to its design ]

Thank you for the support, I do not need donations. Reviews, ratings, and proper bug reports help.
An organized group plagiarizes my work, posts daily lies and mass-downvotes my mods everywhere.
Most modpacks use my work, established projects integrate with it, and downloads near 1 million.

! Reset MCM settings to defaults after updating !

AlifeBalance is a balance layer for vanilla A-Life:
  - Smart Balance: keeps each map's population near what the map's own spawn configs declare.

It runs alongside the engine without rewriting it.
Inventory bounding (formerly Inventory Balance) now lives in AlifeGuard as Inventory Guard.

It works alongside AlifePlus, which adds reactive A-Life behavior across the Zone.
More activity means more losses, and AlifeBalance steers recovery back to each map's designed population.
Both mods are independent and can be used separately.


Smart Balance:
  Vanilla respawn cooldowns run on a fixed schedule that ignores the state of the world.
  A faction you wipe out waits its full turn while an untouched faction keeps cycling, and maps drift away from how they were designed.

  Smart Balance reads the game's own spawner records. Every smart terrain already tracks how many squads it is allowed and how many are alive right now.
  Summed per faction per map, with all mutants as one group, that is the declared population and the actual one, in the engine's own numbers.
  A faction below its declared count gets its respawn cooldowns advanced in proportion to how depleted it is: wiped means full speed, dented means a nudge.
  A faction losing across the whole Zone also spawns fuller squads, up to their own configured maximum (Spawn Size).
  Squads that fell below their own minimum size regain one member per pass, far from the player (Squad Refill).
  A faction above its declared count, and any spawner sitting in an already-crowded area, gets its cooldowns delayed, never past a configurable ceiling (default 12 game-hours).
  One full vanilla cooldown binds when it is shorter, and spawns are never blocked.
  Corrections only go to spawners the game will actually fire, never into crowded areas, so recovery flows toward the empty parts of a map.
  Factions near their declared count are left completely alone.

  What you'll notice:
    Massacre a faction and it returns first, while overgrown factions idle.
    Squads ground down to lone survivors regain their members and stop wandering the map as one-man ghosts.
    Mutant maps stay mutant-heavy and war zones stay contested. Each map drifts back to its designed character.
    A depleted map as a whole recovers faster. A healthy map runs exactly like vanilla.
    Vanilla A-Life still owns every spawn.

  Important:
    Smart Balance never blocks a spawn and never removes an NPC.
    The pacing shifts a timestamp the engine was always going to read on its next alife update.
    Spawn Size and Squad Refill only add members inside a squad's own configured range, never beyond it.
    Populations the spawners do not own are neither boosted nor suppressed. Those are the starting population, and event and mod spawns.
    The engine still owns spawning, recipes, squad selection, and budget caps.

  Example:
    A firefight kills every bandit on Cordon.
    The next pass sees Cordon's bandit spawner slots freed and far below their declared count.
    Every Cordon smart terrain that can spawn bandits gets its cooldown advanced at full strength.
    Surviving bandit squads that dropped below their minimum size regain a member each pass.
    If bandits are collapsing Zone-wide, the replacements arrive at fuller squad sizes.
    Recovery arrives over the next game-hours as the engine reaches each shortened wait. Once bandits are back near their declared count, Smart Balance goes silent.
    If instead a faction holds more spawner-born squads than currently allowed, those cooldowns are delayed up to the configured ceiling until the excess clears.

  Settings (MCM, Respawn Pacing tab):
    Correction passes (2-8, default 6): how many passes carry one smart terrain from full cooldown to the floor at maximum depletion.
    Lower corrects harder per pass. Higher corrects more gradually. Partial depletion scales each push down. Delays use the same step size.
    Minimum cooldown remaining (30-360 game-minutes, default 120): the floor the advance direction never pushes below.
    The engine ages the final wait out on its own clock.
    Maximum cooldown remaining (120-1440 game-minutes, default 720): the ceiling the delay direction never pushes past.
    One full vanilla cooldown stays the bound when it is shorter than the ceiling.
    Spawn Size (own tab, default on): Zone-wide depleted factions spawn fuller squads. The strength slider scales the response.
    Squad Refill (own tab, default on): squads below their own minimum size regain one member per pass.
    Crowded area threshold (General tab): how many off-screen NPCs make an area count as crowded.

  Presets:
    Aggressive: correction passes 2, minimum cooldown 30, spawn size strength 100.
    Default (mild): correction passes 6, minimum cooldown 120 (2 game-hours), maximum 720 (12 game-hours), spawn size strength 50.
    Conservative: correction passes 8, minimum cooldown 360, spawn size strength 25.

Requirements:
Anomaly 1.5.3
Modded exes: themrdemonized 20250908 or newer, or AOEngine v0.55 or newer. The full feature set needs the latest demonized build. A feature that needs a newer one stays inactive on older exes.
xlibs 1.9.0 or newer (https://www.moddb.com/mods/stalker-anomaly/addons/xlibs-1001)
MCM

Compatibility:
Depends only on xlibs. Install and uninstall mid-save work. Tested: Anomaly 1.5.3, GAMMA, EFP, Zona, Forgotten Zone.
Disable (conflict, superseded, problematic):
- Squad Filler - tops up the same squads Squad Refill repairs, to a flat size of 2, ignoring each squad's own configured size.
- Warfare, old or new - Smart Balance steers population back to each map's declared design, while Warfare drives attrition and territory swings, so the two pull it in opposite directions.
Coexists:
- ZCP - shares the cooldown field and spawner records, so the two compose: ZCP picks the species and squad sizes, Smart Balance counts declared slots against them.
- Night Mutants, Nocturnal Mutants - spawn outside the spawner ledger, so Smart Balance neither boosts nor suppresses them.
It coexists with everything else.

How It's Built:

The code and patterns are original, built on best practices from the best STALKER modders and hands-on reverse-engineering of X-Ray.
The design stays engine-native and minimal, with event-native pub/sub not polling, work spread across frames via deferred queues and rate limiters, and per-level caches replacing world scans.
The raycasting and range math are hand-written and load-tested live, following the engine's own standards and flags.
Where scripting hits an engine limit, the fix is made in X-Ray itself, in the modded exes.
Performance is the first invariant, so every flow stays under 2ms or the build rewrites or drops it, profiled continuously with JitProfiler and hand-tested on unoptimized, single-threaded exes.
Every mod carries OpenTelemetry-style tracing and performance monitoring, spanning world events and every flow, gated by the log level so it costs nothing when off.
Every commit runs the full pipeline locally and in CI, with luacheck, a custom STALKER selene build, and a load test on engine stubs.
Rule layers then check Lua practice, engine truth, conventions, contracts, release, security, and docs.
Every rate, threshold, and toggle is exposed through MCM or LTX with nothing left hard-coded, and it writes no engine values, keeping its state within engine bounds so a save can never corrupt.
It runs on one xlibs rulebook shared across the whole mod family, the same protection, distances, faction logic, and combat reads in every mod.
It depends on no other mod, not even the author's own, and needs only X-Ray and xlibs beneath it.
See the Health and JitProfiler links up top for every test and smoke result, and the mod's real CPU and allocation cost.

Credits:
Altogolik provided support, ideas, and source materials.

Usage and License:
  Modpacks: allowed and encouraged. Keep the readme and license files.
  Addons, patches, integrations: allowed. Credit "AlifeBalance by Damian Sirbu" visibly on your mod page.
  Reproducing the implementation in other software: not allowed, even with credit.
  The full license is in the LICENSE file and on GitHub.

Diagnostics and reporting:
Every release goes through careful engineering and testing, but bugs can still slip through.
To report one, reproduce with debug logging on, and the world log where the mod has one.
First rule this mod out: reproduce with it off, then on. The cleanest test is this mod alone on vanilla and xlibs.
Send the traces on the Anomaly Discord, or file a defect on GitHub with the same information.
Attach xray.log, the mod log, the engine build, the modlist, and the load order.
For deep technical details and mechanisms, check the architecture docs on GitHub.

Tags: alife, population-balance, dynamic-spawning, semi-elastic-recovery, engine-canon, engine-native, homeostasis, feedback-control, performance, equilibrium, realistic, save-safe, reverse-engineering
