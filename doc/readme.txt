AlifeBalance: A-Life balance layer for STALKER Anomaly, by Damian
Version: 1.0.0 (xlibs 1.5.1)
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

The engine's respawn_idle is one number applied to every smart on every level. Shorten it and every smart cycles faster, including the ones a hundred metres behind you. The engine's actor-distance gate blocks the spawn only while you stand inside its radius; the moment you walk out, the smart's overdue cooldown triggers an engine spawn and the squad lands at your back. AlifeBalance applies one targeted advance per threshold of deaths, and the picker chooses the eligible smart farthest from you. Refills happen on the level where the kills happened, away from your current combat, one squad at a time.

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

Smart Pacing accelerates vanilla recovery when combat pressure exceeds the engine's pace.

Deaths accumulate into per-level, per-faction counters.
A periodic scanner reads them.
For each pair where the counter has reached the threshold and at least one eligible smart has open spawn budget, Smart Pacing applies advances while pressure remains. Each advance picks the smart farthest from the actor that still has room, pushes its cooldown timestamp closer to expiry by an equal share, subtracts threshold from the counter, and drops the smart from this tick's pool when it hits the minimum-remaining floor. The next advance picks the next-farthest smart, and so on. Refills are biased toward smarts farthest from the actor.

Burst combat that produces many kills in one minute can apply many advances on one tick, distributed across multiple smarts. Leftover kills under threshold carry to the next tick.

MCM exposes two knobs: advance count (default 4) and minimum cooldown remaining in game minutes (default 120). Each advance subtracts the same constant from the picked smart. Smart Pacing never pushes below the floor.

Example:

A heavy firefight wipes 30 bandits on Cordon in one minute.
The death counter reaches 30. Threshold is 3.
On the next tick Smart Pacing applies up to 10 advances: pick a smart, advance, subtract 3, repeat. A smart that hits its floor drops out and other eligibles take the remaining advances. Several smarts can hit floor in the same tick, queueing several refills for shortly after.

If every eligible smart is at budget cap or at the floor, the advance is deferred and the counter holds.

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

Two knobs: advance count (1-8, default 4) and minimum cooldown remaining (10-360 game minutes, default 120).

Advance count is the number of advances needed to drive one smart from full cooldown to the min_minutes floor. One advance subtracts 1/advances of the cooldown cycle from the picked smart. AlifeBalance does not spawn anything: once a smart's cooldown is at the floor, the engine's own clock ages out the rest and the engine's try_respawn fires its normal spawn. At advances=4 (default), four thresholds of deaths drive one smart to the floor, 25% per advance. Lower advance count = larger advance per threshold = faster acceleration of one smart. Higher advance count = smaller advances = more sustained combat before any smart reaches the floor.

Min minutes controls how long the engine ages the final cooldown leg on its own clock after AlifeBalance stops pushing. min_minutes=120 (default) = the engine's spawn fires about two game hours after the last advance. min_minutes=60 = one hour. min_minutes=360 stretches that to about six game hours. Higher values leave the engine more in charge of the final approach.

Presets:
  Aggressive (one threshold = full cooldown advancement):  advances=1, min_minutes=60
  Default (one threshold = 25% advancement):               advances=4, min_minutes=120
  Conservative (one threshold = 12.5% advancement):        advances=8, min_minutes=360

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
