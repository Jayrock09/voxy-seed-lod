# Changelog

## seedlod.8: current patch

- Removed the seedlod.6 and seedlod.7 direct top-level sparse tiles and deferred parent replacement path.
- Added a player-centered loading wave made from 4 by 4-chunk tiles.
- Made the first tile contain the player and gated each complete square ring before the next ring can start, preventing holes behind the loading frontier.
- Shared one generator sample grid across every tile, reducing repeated halo sampling by about 1.38 times at stride 2, 1.78 times at stride 4, and 2.56 times at stride 8.
- Kept automatic stride-2 nearby, stride-4 medium, and stride-8 horizon quality without a destructive refinement pass.
- Kept normal movement work alive while making long teleports cancel and reprioritize stale tasks.
- Returned all prediction insertion to Voxy's ordinary bottom-up `WorldUpdater` hierarchy path.

## seedlod.7

- Extended the initial sparse pass through the starting area, removing the intentional inner gap.
- Replaced scattered detailed chunk updates with aligned 16 by 16-chunk reconstruction regions.
- Added automatic stride-2 nearby, stride-4 medium and stride-8 outer reconstruction tiers.
- Kept coarse parents visible while refined replacement regions are prepared.
- Added deferred, coalesced Voxy hierarchy publication to prevent partial children from cutting rectangular holes into coarse tiles.
- Allowed closer movement to upgrade completed regions without downgrading regions left behind.

## seedlod.6

- Added instant sparse loading using persistent 512 by 512-block Voxy top-level tiles.
- Added shared 32-block terrain sampling for the far field.
- Added the configurable sparse detail distance, defaulting to 64 chunks.
- Prioritized full-distance sparse coverage ahead of detailed refinement.
- Stopped completed sweeps from continuously rescanning the same radius.
- Added movement-based outer-band planning and persistent completed-tile detection.
- Made seed LOD worker-thread changes apply while a world is open.

## seedlod.5

- Added fast early-exit height sampling for vanilla-style noise datapacks.
- Added cached datapack biome profiles and placed-feature hints.
- Added common vanilla-block surface inference for custom biomes.
- Replaced chunk-quadrant tree placement with world-space local-minimum distribution.
- Redesigned round, conifer, and flat-canopy tree silhouettes.
- Added the fast datapack terrain sampling setting.

## seedlod.4

- Replaced hard cliff snapping with continuous slope-aware interpolation.

## seedlod.3

- Added adaptive sample quality.
- Added optional terrain smoothing.
- Added deterministic biome-aware vegetation proxies.

## seedlod.2

- Fixed black daylight rendering by lighting synthetic air.
- Closed terrain undersides with neighbor-aware skirts.
- Removed the transition gap.
- Added prediction markers for safe regeneration.

## seedlod.1

- Initial experimental seed-backed distant terrain sampler.
