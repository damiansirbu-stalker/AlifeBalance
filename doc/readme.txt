AlifeBalance: A-Life balance layer for STALKER Anomaly, by Damian
- Version: 1.0.0 (xlibs 1.5.1)
- Architecture: https://github.com/damiansirbu-stalker/AlifeBalance/blob/main/doc/architecture.md
- Changelog: https://github.com/damiansirbu-stalker/AlifeBalance/blob/main/doc/changelog
- Russian / Na russkom: https://github.com/damiansirbu-stalker/AlifeBalance/blob/main/doc/readme_ru.txt
- Bugs, suggestions: https://github.com/damiansirbu-stalker/AlifeBalance/issues

! Please use the RESET button in MCM when updating to a new version !

The Anomaly A-Life simulation runs respawn cooldowns on a fixed schedule.
The schedule does not know about combat pressure or quiet periods, so heavily fought regions empty out while ignored ones drift toward overcrowding.
AlifeBalance is a balance layer that corrects this gap without rewriting the engine.

What you'll notice:

- Active regions recover faster after heavy firefights.
- Inactive regions stop drifting toward overcrowding.
- Mutant lairs and stalker bases repopulate at a pace closer to the kill rate.
- Vanilla A-Life still owns every spawn.

Important:

AlifeBalance never spawns, destroys, or modifies smart terrain or squad configuration.
It reads engine state and nudges the respawn cooldown timestamps the game already reads.
The engine still owns timing, recipes, squad sections, budget caps, and NPC counts.

Features:

Smart Pacing

Smart Pacing accelerates vanilla recovery when combat pressure exceeds the engine's pace.

Death events fill per-(level, faction) counters.
A periodic scanner reads them.
For each pair where the counter has reached the threshold and at least one eligible smart has open spawn budget, Smart Pacing fires advances while pressure remains. Each advance picks the smart farthest from the actor that still has room, pushes its cooldown timestamp closer to expiry by an equal share, subtracts threshold from the counter, and drops the smart from this tick's pool when it hits the minimum-remaining floor. The next advance picks the next-farthest smart, and so on. Refills land away from the actor's current combat zone before nearby smarts.

Burst combat that produces many kills in one minute can fire many advances on one tick, distributed across multiple smarts. Leftover kills under threshold carry to the next tick.

MCM exposes two knobs: advance count (default 4) and minimum cooldown remaining in game minutes (default 60).
Each advance subtracts the same constant from the picked smart.
Smart Pacing never pushes below the floor.

Example, sustained combat:

A small firefight kills 3 bandits on Cordon.
The death counter reaches the threshold (3, the max NPC count of a Cordon bandit squad).
Smart Pacing picks a bandit-spawning Cordon smart that still has room and moves its cooldown forward.
The counter resets to 0.
Several advances later (across several ticks) the smart hits the floor.
The engine's ageing finishes the cooldown and fires the spawn.

Example, burst combat:

A heavy firefight wipes 30 bandits on Cordon in one minute.
The death counter reaches 30. Threshold is 3.
On the next tick Smart Pacing fires up to 10 advances: pick a smart, advance, subtract 3, repeat. A smart that hits its floor drops out and other eligibles take the remaining advances. Several smarts can hit floor in the same tick, queueing several refills for shortly after.

If every eligible smart is at budget cap or at the floor, the advance defers and the counter holds.

Technical notes (for modders):

Runtime is xlibs, a reverse-engineered API over X-Ray internals.
No engine patches, no script replacement, no parallel schedulers, no entity creation.

What gets counted:
Deaths for all standard stalker factions and all mutant faction keys.
Protected squads (story, companions, task targets) and vermin (rats, tushkanos) are filtered before counting. Everything else contributes; the threshold mechanism (max squad size per faction per level) throttles big-squad species proportionally.

Threshold:
Kills needed before one advance fires for a (level, faction) pair.
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

Compatibility:

Compatible with vanilla Anomaly 1.5.3, GAMMA, ZCP, Redone, Warfare, AlifeGuard, AlifePlus, Night Mutants, GAMMA Dynamic Despawner, Guards Spawner, Nocturnal Mutants.

- Vanilla and ZCP read the same cooldown field AlifeBalance writes.
- Redone and GAMMA NPC Spawns ship pure config. AlifeBalance writes runtime state on a different layer.
- Night Mutants spawns through the engine's own path. Squads still register against their origin smart.
- Nocturnal Mutants spawns outside smart terrains. Ignored.
- Dynamic Despawner and AlifeGuard despawn without firing the death event. Never trigger advances.

MCM:

- Smart Pacing: enable, advance count, minimum cooldown remaining.
- Development: log level, map markers, show status, reset counters.
- Map markers are green PDA spots on every smart receiving an advance. Right-click to teleport or show advance stats.

Configuration tips:

Two knobs: advance count (1-8, default 4) and minimum cooldown remaining (10-360 game minutes, default 60).

Advance count is the kill-to-refill ratio. One advance advances one smart by 1/advances of its cooldown cycle; one full cycle (advances advances on the same smart, or the smart hitting its floor through any combination) is what produces one engine spawn of up to threshold NPCs. So:
  advances=1 means each threshold crossing produces a refill. 3 stalker kills on Cordon (threshold 3) earn one stalker squad respawn.
  advances=4 (default) means four threshold crossings produce a refill. 12 stalker kills earn one respawn.
  advances=8 means eight threshold crossings produce a refill. 24 stalker kills earn one respawn.
Lower advance count makes refills cheaper; higher advance count means more sustained combat is needed before the engine spawns anything.

Min minutes controls how long the engine ages the final cooldown leg on its own clock after AlifeBalance stops pushing. min_minutes=60 = refill fires about one game hour after the last advance. min_minutes=360 stretches that to about six game hours.

Presets:
  Aggressive refill (1:1 kill-to-refill ratio):   advances=1, min_minutes=60
  Default (4:1 kill-to-refill ratio):             advances=4, min_minutes=60
  Conservative, sustained combat only (8:1):      advances=8, min_minutes=360

If a refill does not appear right after a threshold crossing, the engine's actor-distance gate is the usual reason. The engine skips its respawn check at any smart with the actor within respawn_radius. Walk a bit further from the marker, or stay where the fight happened.

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

Credits: Altogolik for support, ideas, and source materials.

License:

- Modpacks: allowed and encouraged. Keep the readme and license files.
- Addons, patches, integrations: allowed. Credit "AlifeBalance by Damian Sirbu" visibly on your mod page.
- Reproducing the implementation in other software: not allowed, even with credit.
- Full license in LICENSE and on GitHub.

Keep the Zone alive while letting vanilla A-Life remain vanilla.
