Version: 1.1.5-snapshot (xlibs 1.10.1, demonized latest)
Changelog: https://github.com/damiansirbu-stalker/AlifeBalance/blob/main/doc/changelog
Health: https://damiansirbu-stalker.github.io/AlifeBalance/health/
JitProfiler: https://damiansirbu-stalker.github.io/AlifeBalance/jitprofiler/
Bugs: https://github.com/damiansirbu-stalker/AlifeBalance/issues
Russian / На русском: https://github.com/damiansirbu-stalker/AlifeBalance/blob/main/doc/readme_ru.txt

---
Alife mods:
  AlifePlus: https://www.moddb.com/mods/stalker-anomaly/addons/alifeplus-v1-0-01
  AlifeTactics: https://www.moddb.com/mods/stalker-anomaly/addons/alifetactics
  AlifeBalance: https://www.moddb.com/mods/stalker-anomaly/addons/alifebalance
  AlifeGuard: https://www.moddb.com/mods/stalker-anomaly/addons/alifeguard-1001
Diegetic mods:
  DiegeticControl: https://www.moddb.com/mods/stalker-anomaly/addons/diegeticcontrol
  DiegeticAmbience
  DiegeticDread
Tools:
  JitProfiler: https://www.moddb.com/mods/stalker-anomaly/addons/jitprofiler
Libraries:
  xlibs: https://www.moddb.com/mods/stalker-anomaly/addons/xlibs-1001
Engines:
  X-Ray Monolith: https://github.com/themrdemonized/xray-monolith/pulls?q=is%3Apr+author%3Adamiansirbu+is%3Amerged
  OpenXRay: https://github.com/OpenXRay/xray-16/pulls?q=is%3Apr+author%3Adamiansirbu+is%3Amerged
  IX-Ray: https://github.com/ixray-team/ixray-1.6-stcop/commits/develop?author=damiansirbu
Dependents:
  COMPASS: https://github.com/Crimento/COMPASS
  Nemesis Spawn System: https://github.com/SirEndet/Nemesis-Spawn-System
  Word of Mouth: https://www.moddb.com/mods/stalker-anomaly/addons/word-of-mouth
Integrations:
  Warfare (erepb): https://www.moddb.com/mods/stalker-anomaly/addons/warfare-alife-overhaul-new
  Homestead: https://www.moddb.com/mods/stalker-anomaly/addons/homestead-stalker-settlement-builder
  Stealth Overhaul: https://github.com/Alex-leon1594/Stealth_Overhaul_Reworked
  TALKER: https://www.moddb.com/mods/stalker-anomaly/addons/talker-ai-powered-npc-dialogue-beta
  Factions of the Zone: https://github.com/devinhorowitz/factionsofthezone
  Anomaly Radios Extended: https://www.moddb.com/mods/stalker-anomaly/addons/anomaly-radios-extended-full-remaster-152
Collaborations:
  xAGNA: https://www.moddb.com/mods/stalker-anomaly/addons/xagna

[ Hero image: alifebalance-hero.gif - each map recovers to its design ]

My work is free and open source, ships in most modpacks, and integrates with established projects.
An organized group copies my work, defames, impersonates and review-bombs me across platforms.
You can help with a review, a rating, a proper bug report, or a coffee on Ko-fi: https://ko-fi.com/damiansirbu

Every release is stable, and a new one can add whole systems or move their settings.
After every update, open Development in MCM and reset every setting to its default.

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
Modded exes: themrdemonized 20250908 or newer, or AOEngine v0.5.2 or newer. The full feature set needs the latest demonized build. A feature needing a newer one is inactive or reduced on older exes.
xlibs 1.9.0 or newer (https://www.moddb.com/mods/stalker-anomaly/addons/xlibs-1001)
MCM

Update check:
On the first save load of each game launch, the mod asks GitHub during the loading screen whether a newer release is out. It needs xlibs 1.10.1 or newer.
A newer version shows at the foot of its MCM page, and in a PDA message once a day. Check for update in the Development tab asks right away, whenever you want.
GitHub sees your IP address and the names of the mods it checks. Each request carries only the mod's name and the word xlibs, with no save data, settings or player name.

Compatibility:
Depends only on xlibs. Install and uninstall mid-save work. Tested: Anomaly 1.5.3, GAMMA, EFP, Zona, Forgotten Zone.
Avoid:
- Squad Filler - it tops up the same squads Squad Refill repairs, to a flat size of 2, and ignores the size each squad is configured with in its own section.
- Warfare, old or new - Smart Balance steers population back to each map's declared design, while Warfare drives attrition and territory swings, so the two pull it in opposite directions.
Coexists:
- ZCP - shares the cooldown field and spawner records, so the two compose: ZCP picks the species and squad sizes, Smart Balance counts declared slots against them.
- Night Mutants, Nocturnal Mutants - their squads spawn outside the spawner ledger, so Smart Balance neither boosts nor suppresses the squads they add at night.
Everything else coexists, as long as it extends X-Ray and Anomaly and never overrides them.

How It's Built:

The code and patterns are original, built on best practices from the best STALKER modders and on hands-on reverse-engineering of X-Ray and the Anomaly scripts.
The design stays engine-native and minimal, with event-native pub/sub not polling, work spread across frames via deferred queues and rate limiters, and per-level caches replacing world scans.
Raycasting and range math are hand-written and load-tested live, on X-Ray's standards and flags.
Where scripting hits an engine limit, the fix is made in X-Ray itself, in the modded exes.
Performance is the first invariant. Every flow stays under 2ms or the build rewrites or drops it, profiled continuously with JitProfiler and hand-tested on unoptimized, single-threaded exes.
Each mod carries OpenTelemetry-style tracing and performance monitoring, spanning world events and every flow, gated by the log level, which skips the tracing work when it is off.
Every commit runs the full pipeline locally and in CI, with luacheck, a custom STALKER selene build, and a load test that runs every script on engine stubs.
Rule layers then check Lua practice, engine truth, conventions, contracts, release, security and docs, and a grade gate fails the build on any code or security finding.
Every rate, threshold and toggle sits in MCM or LTX with nothing hard-coded, and any engine value a mod writes stays within engine bounds, which keeps every save loadable.
It needs only X-Ray and the shared xlibs rulebook beneath it, which gives every mod in the family the same protection, distances, faction logic and combat reads.
It depends on no other mod, the author's other mods included.
See the Health and JitProfiler links up top for every test and smoke result, and for the mod's real CPU and allocation cost, measured in a live game session.

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
First rule this mod out: reproduce with it off, then on.
The cleanest test is this mod alone on vanilla and xlibs.
Send the traces to my Anomaly Discord thread: https://discord.com/channels/456765861953536020/1512539453791670402
Join the server first through the Anomaly Discord invite: https://discord.gg/c4RuJNs
Or file a defect with the same information on GitHub: https://github.com/damiansirbu-stalker/AlifeBalance/issues
Attach xray.log, the mod log, the engine build, the modlist, and the load order.
For deep technical details and mechanisms, check the architecture docs on GitHub.

Tags: alife, population-balance, dynamic-spawning, semi-elastic-recovery, engine-canon, engine-native, homeostasis, feedback-control, performance, equilibrium, realistic, save-safe, reverse-engineering
