# Changelog

## seedlod.10: current patch

- Moved every experimental seed-generation control into a dedicated Seed LOD page in Voxy's settings.
- Increased the maximum prediction distance from 512 to 1024 chunks.
- Added a 4 to 64-chunk adaptive quality-band control. Each band doubles the stride until the configured maximum is reached.
- Replaced the hard stride-16 vegetation cutoff with a continuous distance fade that always retains a sparse far-field tree population.
- Replaced deep terrain bodies with a four-block surface shell while extending only actual exposed cliff edges to their lower neighbor.
- Replaced sampled water volumes with one flat visible water layer at the active generator's datapack sea level.
- Reduced fast noise-based terrain sampling to one solid-surface height query per lattice point, including oceans.
- Cached predicted block and biome mapping IDs instead of repeatedly taking Voxy's mapping locks for every reconstructed column.
- Evaluated OpenGL and OpenCL offload and deliberately did not add a misleading GPU toggle. Minecraft generator and datapack execution is CPU-side, while GPU readback would feed the result back into Voxy's CPU cache path and likely add stalls.

## seedlod.9

- Removed stride 2 and made stride 4 the minimum terrain sampling quality.
- Expanded the maximum sample stride from 8 to 128 with adaptive 4, 8, 16, 32, 64, and 128-block distance bands.
- Aligned sampling to a global world-space lattice so stride 64 and 128 remain continuous across 64-block loading-tile borders.
- Reduced far-tile generator queries from 100 at stride 8 to 9 at stride 128.
- Stopped proxy vegetation after stride 16 so ultra-coarse horizon tiles avoid unnecessary tree geometry.
- Made maximum-stride and adaptive-quality changes restart pending wave plans while the world remains open.

## seedlod.8

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
