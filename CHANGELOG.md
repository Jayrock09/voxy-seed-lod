# Changelog

## seedlod.16: current patch

- Added an optional live Seed LOD debug HUD with scheduling, throughput, cache, hierarchy, replacement, cancellation, latency, and estimated-memory counters.
- Added an optional conceptual section-state map. Completed tiles are colored by Voxy output level, queued or running tiles are white, and missing tiles are dark.
- Added `/voxy seedlod debug` to toggle the HUD and `/voxy seedlod stats` to print a diagnostics snapshot into chat.
- Added low-contention worker counters and rolling rates for generator samples, generated cells, generated sections, average sample time, and average queue latency.
- Added direct visibility into level 0, level 2, level 3, and level 4 write counts, sample-cache hit rate, teleport cancellations, hierarchy repairs, materialized fallback children, and real-chunk replacement counts.
- Added nine automated hierarchy and serialization tests covering octant masks, partial coverage, fallback expansion, finer-data preservation, real-data preservation, concurrent publication, negative coordinates, current save/reload, and old-cache repair state.
- Added JUnit 5 to the Gradle test workflow so hierarchy invariants are checked by `gradlew test` and the normal release build.

## seedlod.15

- Fixed the remaining large rectangular holes caused by partial hierarchy refinement.
- Materializes every non-air sibling covered by a coarse parent before publishing its finer-child mask. Voxy can therefore keep its guarantee that every missing child is actually empty.
- Added a persistent coarse-child completion flag so sibling expansion runs once per coarse section instead of being repeated for every generated chunk.
- Repairs incomplete sibling coverage saved by seedlod.12 through seedlod.14 as the normal generation wave touches those sections. Existing worlds should not require a Voxy-cache reset.
- Preserves existing finer child octants while expanding fallback data into uncovered octants.
- Backfills missing level-0 volume without replacing existing non-air real terrain, allowing ordinary loaded chunks to remain authoritative.

## seedlod.14

- Fixed the remaining persistent rectangular gaps at screen-space quality transitions.
- Stopped leaving parent fallback cells empty merely because a finer child exists. Voxy can now select either parent or child geometry without exposing an unpopulated rectangle.
- Kept authoritative unmarked parent cells protected while allowing marked seed predictions to refresh older predicted fallback cells.
- Corrected the predicted water surface by one block. The generator's sea-level value is the boundary above the water, so vanilla sea level 63 now places the highest predicted water block at Y=62.
- Added a minimum adaptive sample-stride slider with choices 4, 8, and 16.
- Restricted maximum sample stride to 32, 64, and 128. The non-overlapping ranges make an invalid minimum and maximum ordering impossible.
- Adaptive quality now begins at the configured minimum and doubles at each quality band until reaching the maximum.
- Kept maximum stride as the fixed stride when adaptive quality is disabled, preserving the previous non-adaptive behavior.

## seedlod.13

- Fixed the rectangular gaps between N-sized quality bands.
- Replaced partial 64-block coarse patches with complete horizontally aligned native Voxy sections.
- Batched stride-16 work into 128-block sections, stride-32 work into 256-block sections, and stride-64 or stride-128 work into 512-block sections.
- Deduplicated each aligned N-sized section across all worker threads. This reduces far-field task and hierarchy-publication counts by 4 times at level 2, 16 times at level 3, and 64 times at level 4 compared with seedlod.12's 64-block patch jobs.
- Kept the player-centered 64-block scan wave for ordering and for stride 4 or 8 detail, while coalescing the farther scan tiles into complete native work units.
- Restored stride 128 as the maximum performance tier. With the new 512-block level-4 work unit, its cold interpolation grid is 6 by 6 instead of stride 64's 10 by 10, reducing far-section generator queries from 100 to 36 while retaining N=16 output cells.
- Kept stride 64 as the default maximum for stronger horizon detail. Stride 128 is an opt-in performance setting.
- Kept stride 256 out of the slider. It saves only 20 more cold queries per level-4 section after stride 128 while discarding substantially more terrain detail, which is the start of the practical diminishing-return range.

## seedlod.12

- Added optional direct N-sized generation, enabled by default.
- Kept stride 4 and 8 on the highest-quality level-0 path.
- Mapped stride 16 directly to N=4 cells, stride 32 to N=8 cells, and stride 64 to N=16 cells.
- Added persistent coarse-leaf state to Voxy sections so a non-empty direct LOD can exist without pretending it already owns finer children.
- Added parent-to-child fallback expansion before refinement. A complete child background is published before finer predictions or real chunks overlay it, preventing partial children from cutting rectangular air holes out of coarse parents.
- Added direct coarse-cell mipping and child-existence propagation through Voxy's stored hierarchy.
- Added a world-aligned shared sample cache for stride 16 through 64. Adjacent tiles reuse generator results instead of repeating halo queries.
- Kept ordinary chunks authoritative and made direct coarse writes skip octants already owned by finer children.
- Added coarse biome-aware vegetation cells and retained the flat datapack sea-level water model.
- Added an in-game switch to disable N-sized output while retaining shared sampling and the seedlod.11 level-0 path.

## seedlod.11

- Set stride 64 as the maximum useful sampling interval.
- Removed stride 128 from the settings slider and adaptive quality tiers.
- Clamped older configurations containing stride 128 to 64.
- Documented the sampling floor: both stride 64 and stride 128 require the same 3 by 3 interpolation grid and 9 generator queries for a 4 by 4-chunk wave tile, so 128 reduced quality without improving sampling speed.

## seedlod.10

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
