AlifeBalance: A-Life balance layer for STALKER Anomaly, by Damian
Version: 1.0.0 (xlibs 1.5.2)
GitHub: https://github.com/damiansirbu-stalker/AlifeBalance
Changelog: https://github.com/damiansirbu-stalker/AlifeBalance/blob/main/doc/changelog
Architecture: https://github.com/damiansirbu-stalker/AlifeBalance/blob/main/doc/architecture.md
Russian / Na russkom: https://github.com/damiansirbu-stalker/AlifeBalance/blob/main/doc/readme_ru.txt
Bugs, suggestions: https://github.com/damiansirbu-stalker/AlifeBalance/issues

! Please use the RESET button in MCM when updating to a new version !

The Anomaly A-Life simulation runs respawn cooldowns on a fixed schedule.
The schedule does not know about combat pressure or quiet periods, so heavily fought regions empty out while ignored ones drift toward overcrowding.
AlifeBalance is a balance layer that corrects this gap without rewriting the engine.

Built as a companion to AlifePlus, which increases NPC activity and combat across the Zone. More activity means more deaths, and AlifeBalance scales refill to match. Both mods are independent and can be used separately.

Why not just shorten the global cooldown to 1h or 2h?

The engine's respawn_idle is one number shared by every smart terrain in the entire Zone: every level, every faction, every empty corner. Shorten it and the world cycles faster everywhere at once. The level you are fighting in and the quiet maps you will never visit this session both refill at the new accelerated rate. The engine has no concept of where combat is happening; it accelerates uniformly. Tactically the actor-distance gate blocks the spawn only while you stand inside a smart's radius, so the moment you walk out the overdue cooldown fires immediately and the squad lands at your back. AlifeBalance applies one targeted advance per threshold of deaths, and the picker chooses the eligible smart farthest from you. Refills happen on the level where the kills happened, away from your current combat, one squad at a time.

What you'll notice:

- Active regions recover faster after heavy firefights.
- Inactive regions stop drifting toward overcrowding.
- Mutant lairs and stalker bases repopulate at a pace closer to the kill rate.
- Refills land at the far end of the level, off-screen, one squad at a time.
- Vanilla A-Life still owns every spawn.

Important:

AlifeBalance never creates NPCs. It shifts a timestamp the engine was always going to read on its next alife tick. The engine still owns spawning, recipes, squad selection, and budget caps.

Features:

Smart Pacing

The core feature. It speeds up vanilla respawns where combat has actually happened, and only there.

In plain words: every smart terrain has a built-in waiting period before it can spawn again. AlifeBalance counts deaths per level and per faction. Once enough deaths have piled up for a faction (a number tied to the largest squad size for that faction's spawn pool), it shortens the wait at one matching smart terrain. The next respawn there gets pulled closer in time, but never all the way to immediate. AlifeBalance picks the smart terrain farthest from the player so the refill happens off-screen.

The engine still handles the spawn itself on its own clock. AlifeBalance only nudges that clock forward, and always leaves at least 2 game-hours of natural waiting before the engine fires. That margin is intentional: combat produces a delayed wave of refills, not instant arrivals.

Heavy combat moves several smart terrains forward in the same minute. Quiet zones see no movement.

Two settings in MCM:
- Advance count (default 4): how aggressively each combat burst shortens the wait. Lower = each burst matters more; higher = needs sustained combat.
- Minimum waiting time (default 2 game-hours): the floor AlifeBalance never pushes below. Higher leaves more delay between combat and refill.

Example:

A firefight kills 30 bandits on Cordon in one minute. Cordon bandit squads max out at 3, so 10 separate adjustments are available. AlifeBalance picks up to 10 different bandit-spawning Cordon smart terrains and shortens each one's wait. The refills land over the next couple of game-hours as the engine reaches each smart's now-shorter wait. If every Cordon bandit smart terrain is already at population cap or already at the minimum wait, the adjustments are held and the kills carry over to the next check.

Compatibility:

Compatible with vanilla Anomaly 1.5.3, GAMMA, ZCP, Redone, Warfare, AlifeGuard, AlifePlus, Night Mutants, GAMMA Dynamic Despawner, Guards Spawner, Nocturnal Mutants.

- Vanilla and ZCP read the same cooldown field AlifeBalance writes.
- Redone and GAMMA NPC Spawns ship pure config. AlifeBalance writes runtime state on a different layer.
- Night Mutants spawns through the engine's own path. Squads still register against their origin smart.
- Nocturnal Mutants spawns outside smart terrains. No interaction.
- Dynamic Despawner and AlifeGuard despawn without firing the death event. Never trigger advances.

MCM:

- Smart Pacing: enable, advance count, minimum cooldown remaining.
- Development: log level, map markers, show status, reset counters.
- Map markers are green PDA spots on every smart receiving an advance. Right-click to teleport or show advance stats.

Configuration tips:

Two settings: advance count (1-8, default 4) and minimum cooldown remaining (10-360 game minutes, default 120).

Advance count is the number of advances needed to drive one smart from full cooldown to the Min Minutes floor. One advance subtracts 1/advances of the cooldown cycle from the picked smart. AlifeBalance does not spawn anything: once a smart's cooldown is at the floor, the engine's own clock ages out the rest and the engine's try_respawn fires its normal spawn. At advances=4 (default), four thresholds of deaths drive one smart to the floor, 25% per advance. Lower advance count = larger advance per threshold = faster acceleration of one smart. Higher advance count = smaller advances = more sustained combat before any smart reaches the floor.

Min Minutes controls how long the engine ages the final cooldown leg on its own clock after AlifeBalance stops pushing. Min Minutes=120 (default) = the engine's spawn fires about two game hours after the last advance. Min Minutes=60 = one hour. Min Minutes=360 stretches that to about six game hours. Higher values leave the engine more in charge of the final approach.

Presets:
  Aggressive (one threshold = full cooldown advancement):  advances=1, Min Minutes=60
  Default (one threshold = 25% advancement):               advances=4, Min Minutes=120
  Conservative (one threshold = 12.5% advancement):        advances=8, Min Minutes=360

Technical notes (for modders):

Runtime is xlibs, a reverse-engineered API over X-Ray internals.
No engine patches, no script replacement, no parallel schedulers, no entity creation.

What gets counted:
Deaths for all standard stalker factions and all mutant faction keys.
Protected squads (story, companions, task targets) and vermin (rats, tushkanos) are filtered before counting. Everything else contributes; the threshold mechanism (max squad size per faction per level) throttles big-squad species proportionally.

Threshold:
Kills needed before one advance is applied for a (level, faction) pair.
Set automatically to the largest squad size that can produce that faction at any eligible smart on the level. Cordon stalkers max at 3 NPCs per squad, so the Cordon stalker threshold is 3. Swamp boars max at 15, so the Swamp boar threshold is 15. Read from squad_descr LTX on first death and cached.
Mutant-heavy levels land around 15 to 20, stalker-only levels around 3 to 5.

Eligibility:
A smart is eligible when one of its respawn recipes has a squad section whose squad_descr faction matches.
Routing props are not checked (they govern where squads can move, not what recipes spawn).
The picker further filters to smarts with open per-recipe budget AND room for another advance.

Refill <= death:
Between consecutive engine spawns at any eligible smart for a (level, faction) pair, the engine refills no more NPCs of that faction on that level than have died.
The threshold caps each refill against the deaths that earned it.

Performance:
O(1) per death, O(active state) per scan, cached.
DEBUG-level tracing reports per-event and per-tick timings to alifebalance.log.
At WARN the mod is silent.

Requirements:

- Anomaly 1.5.3
- xlibs (https://www.moddb.com/mods/stalker-anomaly/addons/xlibs-1001) 1.5.1+
- MCM

Install (MO2):

1. Install xlibs.
2. Install AlifeBalance.
3. Load order does not matter.
4. Configure via MCM.

Uninstall: disable or remove in MO2.

Development:
Written against X-Ray Monolith engine source, Demonized exes source code, and Anomaly 1.5.3 unpacked gamedata.
Code patterns and engine usage validated against established work by reputable GAMMA modders (Demonized, Vintar0, RavenAscendant, xcvb).
The code is validated in real time by a multi-stage pipeline: luacheck, selene, tree-sitter AST analysis, contract rules, cross-file dependency resolution, cyclomatic complexity analysis, crash and vulnerability pattern detection, lua54 integration testing with X-Ray engine stubs, gitleaks secret scanning.
Full report in doc/test-report.log.

Credits:
Altogolik - support, ideas, source materials

Usage and License:
  Modpacks: allowed and encouraged. Keep the readme and license files.
  Addons, patches, integrations: allowed. Credit "AlifeBalance by Damian Sirbu" visibly on your mod page.
  Reproducing the implementation in other software: not allowed, even with credit.
  Full license in LICENSE file and on GitHub.

Keep the Zone alive while letting vanilla A-Life remain vanilla.
