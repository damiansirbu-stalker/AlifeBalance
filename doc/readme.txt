AlifeBalance: A-Life balance layer for STALKER Anomaly, by Damian
- Version: 1.0.0 (xlibs 1.5.1)
- Architecture: https://github.com/damiansirbu-stalker/AlifeBalance/blob/main/doc/architecture.md
- Changelog: https://github.com/damiansirbu-stalker/AlifeBalance/blob/main/doc/changelog
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
When kills cross a threshold, Smart Pacing picks an eligible smart with open spawn budget and pushes its cooldown timestamp closer to expiry by an equal share.
After enough waves the smart sits at the minimum-remaining floor, and the engine's own clock ages out the last leg and fires the spawn naturally.

MCM exposes two knobs: wave count (default 4) and minimum cooldown remaining in game hours (default 1).
Each wave subtracts the same constant from the picked smart.
Smart Pacing never pushes below the floor.

Example:

A firefight wipes several bandits on Cordon.
The death counter climbs past the threshold (around 3, the max NPC count of a Cordon bandit squad).
Smart Pacing picks a bandit-spawning Cordon smart that still has room and moves its cooldown forward.
The counter resets.
Several waves later the smart hits the floor.
The engine's ageing finishes the cooldown and fires the spawn.
If every eligible smart is at budget cap or at the floor, the wave defers and the counter holds.

Technical notes (for modders):

Runtime is xlibs, a reverse-engineered API over X-Ray internals.
No engine patches, no script replacement, no parallel schedulers, no entity creation.

What gets counted:
Deaths for all standard stalker factions and all mutant faction keys.
Protected squads (story, companions, task targets) and cowardly-tier mutants (rats, tushkanos, flesh, zombies, karliks) are filtered before counting.

Threshold:
Per (level, faction), the max npc_in_squad upper bound across squad sections producing the faction at any eligible smart on the level.
Read from squad_descr LTX on first death and cached.
Mutant-heavy levels land around 15 to 20, stalker-only levels around 3 to 5.

Eligibility:
A smart is eligible when one of its respawn recipes has a squad section whose squad_descr faction matches.
Routing props are not checked (they govern where squads can move, not what recipes spawn).
The picker further filters to smarts with open per-recipe budget AND room for another wave.

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
- Dynamic Despawner and AlifeGuard despawn without firing the death event. Never trigger waves.

MCM:

- Smart Pacing: enable, wave count, minimum cooldown remaining.
- Development: log level, map markers, show status, reset counters.
- Map markers are green PDA spots on every smart receiving a wave. Right-click to teleport or show wave stats.

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
