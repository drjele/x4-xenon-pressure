# Development

## Checks and formatting

Use Python 3.10 or newer and Bash. Install the pinned tools in a virtual environment:

```bash
python3 -m venv .venv
.venv/bin/python -m pip install -r requirements-dev.txt
export PATH="$PWD/.venv/bin:$PATH"
python3 scripts/check.py
```

Run `python3 scripts/check.py --fix` to format Python and shell and normalize text whitespace. The same checks run on pushes and pull requests. XML is checked for well-formedness; game schemas, XPath matches and gameplay require separate X4 validation. Blender scripts are parsed and linted without importing Blender.

Use UTF-8, LF, a final newline, spaces and no trailing whitespace. Indent code with four spaces and workflow YAML with two. Use descriptive names, uppercase shell variables, constant-first equality comparisons and explicit boolean checks. Ruff's E712 rule is disabled to retain explicit boolean comparisons. Keep shell free of prose comments. Keep only short, non-obvious constraints in code; put explanations here. XML continuation attributes may align with their opening attribute. Preserve XPath selectors, savegame identifiers and embedded game expressions when applying formatting.

## Installation and publishing helpers

`install.sh` and `publish.sh` both source `lib/find_x4.sh`. The library searches usual Steam roots and additional library folders. `X4_PATH`, `X_TOOLS_PATH`
and `PROTON_PATH` override discovery. Proton Experimental is preferred when found; otherwise the helper uses the last matching Proton directory it encounters.

Installation replaces the extension directory with a copy of `extension/`. Refresh it after edits; X4 enumerates real extension directories, so a symlink does not substitute for installation. Restart the game after installing or removing.

Publishing stages a separate copy inside the game's extensions directory. The repository keeps its readable extension id; `steam/workshop-id` holds the numeric Workshop id. The helper changes only the staged manifest, runs WorkshopTool in batch mode and uses an exit trap to restore the manual installation after both success and failure. On Linux it runs WorkshopTool through Proton and maps paths through drive Z. If restoring the manual installation fails, rerun `./install.sh`.

## Release metadata

`content.xml` uses an integer version multiplied by 100 and an ISO release date. The date matches the corresponding released entry in `CHANGELOG.md`. Development changes belong under `Unreleased`; they do not advance the manifest's release version or date. An unreleased scaffold may retain its initial creation date until its first release. Keep existing extension ids stable.

## Implementation constraints

### extension/md/drjele_xenon_pressure.xml

Configuration only. Re-read on every savegame load, so editing a value and reloading is enough - no new game required. Each one can also be overridden at runtime without touching this file, by setting the matching global variable, which is also what the in-game options menu writes - see drjele_xenon_pressure_options.xml.

Almost every consumer is a vanilla script that runs on its own schedule and reads the globals at the moment it evaluates: the station library every one to two minutes, the goal scripts on every evaluation, the attack handler on every hit. Those need no apply step.

The one exception is long range raids. $ExpeditionEnemies is a variable of the Xenon faction manager cue - a param of the Manager library, reset to [] once by a savegame patch - and the invasion goal reads it from there, so it is written rather than patched. Apply runs on md.Setup.Start through a child that waits for the manager variable to exist (both scripts hang off the same signal and the order between them is undefined), and Reapply is signalled by the options script when the toggle moves. The existence check doubles as the "faction logic is running" test: managers only exist in the main galaxy, so in a Timelines scenario, a tutorial or the workshop map the mod's apply cue never fires. The list is built from global.$FactionManagers.keys.list, which vanilla itself iterates, minus xenon and khaak - so it only ever holds factions that exist in this game, without naming any DLC faction id in the mod.

Every setting is looked up the same way at the point of use: the global the options menu writes wins, otherwise the table Configuration publishes, otherwise the vanilla value. All of them are global variables, so a configuration that fails to load degrades to stock behaviour rather than breaking the Xenon. The table is tested for existence rather than the field inside it, so a value edited down to the vanilla number is still read as configured.

### extension/md/drjele_xenon_pressure_options.xml

In-game options, through SirNukes Mod Support APIs. Everything that touches that API lives in this file, so if the API is not installed nothing here ever runs and the mod keeps working off the constants in drjele_xenon_pressure.xml.

The callbacks only write the global override variables the patches read; the long range raids callback also signals Reapply, because that setting is applied by writing a manager variable rather than read live.

The API only offers "button" and "slidercell" widgets, so the three-way miner behaviour is a 0-2 slider whose mouseover spells the values out.

The API persists an option's value under its $id through md.Userdata into uidata.xml, which is shared between savegames, and hands it straight back to the widget as its start value. A stored value outside the slider's range fails widget validation and takes the whole Extension Options menu down with it, so the range and the unit of an option must never change under a $id that has already shipped - give it a new one instead. That is why these ids carry their unit. The callback also fires once on every menu reload with the stored value, which is what puts the globals in place on a load. Slider values arrive as floats (the save shows them as largefloat), so every consumer that needs a whole number casts with the expression language's )i suffix; a float in the ceiling division would otherwise round 32 / 5 up to 8 instead of 7.

### extension/md/factionlogic_stations.xml

The targets live in the vanilla library Manage_Stations, instantiated once per faction by md/factionlogic.xml and reset every one to two minutes, so a patch to its actions is live on an existing save without any apply step.

The override is inserted before the "$ShipyardCount lt $DesiredShipyards" do_if rather than after the "@$DesiredShipyardPatchMarker" do_elseif that the DLC patches hang off. Extensions patch in load order and the mod's position relative to the DLC is not something to rely on; before the count check is after every addition whatever the order, and the last fallback is then the DLC-adjusted vanilla value, not a literal. The do_elseif on "$Faction == faction.xenon" is not a usable anchor either - it matches twice, once for shipyards and once for wharfs.

$DrJeleBuildChance is set to 5 unconditionally and then overridden inside the xenon do_if, so the replaced chance attribute keeps vanilla behaviour for every other faction. The chance attribute has type expression in common.xsd; vanilla uses chance="$DebugChance" in the same file.

The target is a floor: the library builds when the count is below it and never removes a surplus. Past the seven god-seeded sites it builds from the xen_shipyard / xen_wharf construction plans in a sector the faction already has stations in, and it scores a sector that already holds one of the type at minus one hundred, so the effective ceiling is the number of Xenon sectors.

The sector ratio reads $ClaimedSectors, the list of sectors owned or contested by the faction that the same library's Sector_Iterate fills on every cycle, before Analyse_Stations is signalled. Ceiling division is written as (count + per - 1) / per because both operands are integers. At 0 the vanilla, DLC-adjusted $DesiredShipyards is left as it is; there is no fixed slider any more, the ratio is the whole rule.

The construction cap counts in Sector_Iterate, next to vanilla's own shipyard and wharf counting, and is reset in Process next to vanilla's own resets, so it is always defined by the time Analyse_Stations reads it. A construction in the save is $Station.isconstruction; isplannedshipyard and isplannedwharf are true for it already, which is why vanilla counts constructions towards the target. When the cap is reached the target is pinned to the current count rather than the build variable zeroed, because the vanilla build runs inside the same do_if that rolls the chance and there is no anchor between the two. The wharf check adds $ShipyardsToBuild, since a shipyard started in the same cycle is not in the count yet.

### extension/md/factiongoal_invade_space.xml

$MinimumStrength and $MaximumStrength are locals of the invasion goal instance, recomputed on every EvaluateState from the faction's mood and its recon of the target, so the only place to scale them is right after vanilla's own two set_values. $Faction is on the goal instance, so the block is gated on it and other factions run untouched. The scale is a percentage in integer arithmetic; the strength values are floats so the division keeps the fraction.

$RequestStrengthAllowance is set in two places - the staging phase and the beachhead phase - with identical name and comment, so the two selectors are told apart by the phase do_if / do_elseif above them. The attribute is replaced rather than a set_value added after it, because it sits inside a do_else that has no other convenient anchor. The expression keeps the vanilla quarter for every faction that is not xenon.

Raising the minimum without the build share makes invasions fail: prepare_for_invasion has a one hour ceiling ($PhaseMaxEndTime), after which a goal that has not reached its strength hands off. The README says so.

### extension/md/factiongoal_hold_space.xml

this.$MaxAttackEnemyStationSubGoals is set by the mood chain and read on the same evaluation to compute this.$AllowedAttackEnemyStationSubgoals, which is also capped by this.$MaxDefenceSubGoals; the override therefore lifts the defence cap to at least the attack cap so a value above 9 would not be silently clamped (the slider stops at 6, well below it). The anchor is the remove_value of $Mood that closes the chain - the only one in the file.

### extension/aiscripts/interrupt.attacked.xml

The mode is computed once as the first condition of the handler, with a set_value - a legal condition, always met, the same device vanilla uses for $fleerespond and $attackrespond further down - so the three check_values below read a local rather than re-evaluating the global lookup. It is 0 for anything that is not a Xenon shiptype.miner, which makes every added check a no-op for every other ship. The energy haulers are miner hulls too, because the xenon_free_trader_m_energy job selects the miner tag.

The flee block gets "mode != 1" prepended, so a mode-1 miner never flees; its inner check_any gets "mode == 2" prepended, so a mode-2 miner flees whatever its response setting, morale, skill or shields. The fight block gets "mode == 0", so only vanilla mode fights. Fight is gated on == 0 rather than != 1 on purpose: it must not depend on the check_any having already rejected the flee branch. The call-for-help block is untouched in every mode.

The deploydistraction parameter of the Flee order is the whole expression, replaced rather than appended to, because it is an attribute; it keeps the vanilla clause verbatim and adds "mode != 2" in front.

The variable is removed at the end of the actions next to vanilla's own cleanup. If the conditions fail after the set_value the variable lingers on the order script until the next hit, where it is overwritten before it is read.

The Pirate DLC patches the same handler: it appends a loanshark check_all after "call for help" and a do_if after the callforhelp do_elseif. Every selector here still matches exactly one node with that patch applied first. Xenon Hell and AI Xenon Miners replace the same nodes and must not be loaded alongside.

### extension/assets/units/size_m/macros/storage_xen_m_miner_01_a_macro.xml

Cargo is a macro property, read when the game starts and reaching every existing ship on the next load, so it cannot be an in-game option; the number is the file. Both Xenon M miner hulls connect this one storage macro, which is why a single patch covers them. The macro lives in 01.cat only and no DLC overrides it.

### extension/libraries/jobs.xml

Quotas are read when the game starts and cannot be changed by a script, so they cannot be sliders. Values are taken from Xenon Hell with two corrections. The libraries schema defines maxgalaxy as the count past which the job engine removes ships, defaulting to twice galaxy; a galaxy quota above an explicit maxgalaxy spawns and culls in a loop, which is what Xenon Hell shipped for the border carrier (5 against 1) and the cluster destroyer (21 against 18). Every explicit maxgalaxy here is at or above its galaxy, and the frigate patrol's is doubled to keep vanilla's headroom. The local miner job is new since Xenon Hell and is scaled by the same factor as the galaxy one. xenon_frigate_escort_m already has wing 5 in 9.00 and is not patched.

### extension/libraries/wares.xml

The Terran DLC adds location entries to three of these jobs and nothing to the wares; no DLC touches the patched attributes. Amounts are patched one attribute at a time instead of replacing the ware element, because a whole-element replace drops dismantlefactor and every attribute another mod may add. The destroyer's 9.00 recipe is not the 4.0 one Xenon Hell halved, so its numbers are half of the current recipe rather than Xenon Hell's.

The Xenon shipyard and wharf construction plans (xen_shipyard, xen_wharf in constructionplans.xml) use three wares the Commonwealth also builds: module_gen_build_dockarea_m_01 and the two Argon defence modules. Each of those carries a second production element, method="xenon", with the energy cells / ore / silicon recipe; the patches select that method by attribute so the default Commonwealth recipe is not touched. Xenon stations need no construction vessel, so wares and build time are the whole of what limits them. The zz1m_build_time mod patches module build times too, but none of the wares here.
