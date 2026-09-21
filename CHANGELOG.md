# Changelog

All notable changes to this project will be documented in this file.

## [Unreleased]

### Added

- Square Xenon fleet preview matching the visual style of the other DrJele X4 mods.

### Fixed

- Workshop description now identifies the non-vanilla miner default consistently and explains how to clear long range raids before removal.
- Development and debugging documentation reflects batch publishing, restoration on exit and the limits of a quiet debug log.
- CI workflow indentation follows the documented two-space YAML convention.

## [v1.0.0] - 2026-09-21 - Initial release

### Added

- **Sectors per shipyard** and **Sectors per wharf** sliders, 0 to 5, setting `$DesiredShipyards` and `$DesiredWharfs` for `faction.xenon` to one station per that many sectors the Xenon own or contest, from the library's own `$ClaimedSectors`; inserted right before the vanilla count check so it lands after every DLC addition, and 0 leaves the vanilla target.
- **Constructions in progress** slider, 0 to 10, counting shipyard and wharf constructions in the vanilla sector pass and holding the target at the count while that many are still being built.
- **Build chance** slider, 1 to 100 percent, replacing the literal `chance="5"` on `$ShipyardsToBuild` and `$WharfsToBuild` with a variable that only the Xenon change.
- **Invasion fleet size** slider, 50 to 300 percent, scaling `$MinimumStrength` and `$MaximumStrength` of Xenon goals in `md/factiongoal_invade_space.xml` right after vanilla computes them.
- **Invasion build share** slider, 0 to 100 percent, replacing the two `$DesiredShipStrength / 4` build allowances in `md/factiongoal_invade_space.xml`, Xenon only.
- **Station attack groups** slider, 1 to 6, overriding `$MaxAttackEnemyStationSubGoals` for Xenon goals in `md/factiongoal_hold_space.xml`.
- **Long range raids** toggle, writing every faction with a faction manager into the Xenon manager's `$ExpeditionEnemies` on load and on change.
- **Miner behaviour** setting, 0 vanilla, 1 keep flying, 2 always flee without laser towers, gating the *flee* and *fight* branches of the vanilla `AttackHandler` in `aiscripts/interrupt.attacked.xml` for Xenon `shiptype.miner` hulls.
- Xenon M miner cargo 9 500 to 22 500 through `storage_xen_m_miner_01_a_macro`, shared by both miner hulls.
- Xenon job quotas raised in `libraries/jobs.xml`, with `maxgalaxy` kept at or above every raised `galaxy` quota.
- Xenon ship build resources halved in `libraries/wares.xml` for the N, M, P, K, I and both M miners, patched per attribute rather than by replacing the ware.
- Xenon station module resources and build times halved in `libraries/wares.xml`; the shared wharf build module and Argon defence modules only in their `xenon` production method.
- In-game options menu through SirNukes Mod Support APIs, with file constants and global overrides as the fallback when it is not installed.

### Notes

- Slider values reach the scripts as floats and are cast to integers with `)i` wherever a count is computed.

- Replaces Xenon Hell and AI Xenon Miners, which patch the same nodes of `interrupt.attacked.xml` and, in Xenon Hell's case, the same jobs and wares. The ship buffs Xenon Hell also carried (P laser damage, M travel engine thrust, miner hull) are not included.
- The destroyer's resources are half of the 9.00 recipe, not Xenon Hell's numbers, which were half of the 4.0 one.

[v1.0.0]: https://github.com/drjele/x4-xenon-pressure/releases/tag/v1.0.0
