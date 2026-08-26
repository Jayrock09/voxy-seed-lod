> [!WARNING]
> **AI USE DISCLOSURE:** Generative AI was used heavily to design, implement, debug, document, and iterate on this experimental mod. Some of this is therefore, affectionately and occasionally literally, **AI slop**. I would be more than happy for an organic life form to audit it, clean it up, or remake the idea without the slop. Human-written improvements and clean-room reimplementations are extremely welcome.

# Voxy Seed LOD

An unofficial experimental patch for [Voxy](https://github.com/MCRcortex/voxy) that predicts distant Overworld terrain directly from the integrated server's seed-backed `ChunkGenerator` without requesting or loading every faraway Minecraft chunk.

The goal is simple: fill Voxy's distant horizon quickly enough to be useful at 128 to 1024-chunk distances while keeping ordinary generated chunks authoritative when the player reaches them.

## Important upstream notice

Voxy is created by MCRcortex and is marked **All Rights Reserved / Do not redistribute**. This repository therefore does **not** contain Voxy's source or compiled JAR. It contains only the original patch, documentation, and instructions needed to apply the experiment to an authorized local checkout.

This project is unofficial, unsupported by MCRcortex, and not affiliated with Mojang or Microsoft. Do not report problems caused by this patch to upstream Voxy.

## What this adds

### Seed-sampled distant terrain

- Samples the active single-player Overworld `ChunkGenerator` without creating faraway chunks.
- Writes predictions into Voxy's existing voxel, mipping, storage, meshing, occlusion, and rendering pipeline.
- Uses a distance-priority worker queue with at most 256 pending wave tiles.
- Worker-thread changes take effect while the world is open.
- Begins at the tile containing the player. Existing real chunks fill the center while predictions continue directly outward.
- Marks every prediction so later predictions may replace it while real ingested chunk data remains authoritative.
- Supports prediction distances from 32 to 1024 chunks and sample strides of 4, 8, 16, 32, or 64 blocks.
- Places all experimental controls on a dedicated Seed LOD settings page instead of crowding Voxy's General page.

### Player-centered connected loading wave

- Divides prediction work into 4 by 4-chunk tiles and starts with the tile containing the player.
- Schedules complete square rings from nearest to farthest. Ring N+1 cannot start until every in-range tile in ring N finishes, so unfinished work stays on the outer edge instead of leaving holes behind it.
- Sorts the 16 chunks inside each tile by player distance before inserting them.
- Shares one generator sample grid across the whole tile, removing repeated chunk-border and halo queries.
- Stride 16 through 64 also share generator samples between adjacent tiles through a bounded world-aligned cache.
- Uses ordinary bottom-up chunk insertion at stride 4 and 8, then direct N-sized Voxy cells at stride 16 through 64.
- Keeps real ingested chunks authoritative and safely upgrades predicted tiles when the player moves closer.
- Recenters after four chunks of movement. A teleport of at least 32 chunks cancels stale queued work and begins a new wave at the destination.

### Distance-based quality

- With adaptive quality enabled, nearby tiles use stride 4 and outward bands progressively use stride 8, 16, 32, and up to 64.
- A configurable band width from 4 to 64 chunks controls when each quality drop happens. A width of 4 doubles the stride every 4 chunks, while 64 preserves each tier for 64 chunks.
- Sampling is aligned to a global world-space lattice, so strides larger than the 64-block loading tile remain continuous across tile borders.
- The selected quality and output size are decided before a tile is generated.
- Moving closer can regenerate an existing predicted tile at a finer stride. Moving away does not waste time downgrading completed data.
- Voxy still chooses its rendered screen-space LOD normally after the predicted chunks enter the cache.

### Direct N-sized generation

- Optional and enabled by default. Disable it to retain seedlod.11-style level-0 reconstruction.
- Stride 4 and 8 keep one-block output for the two highest-quality bands.
- Stride 16 generates N=4 cells directly at Voxy level 2.
- Stride 32 generates N=8 cells directly at Voxy level 3.
- Stride 64 generates N=16 cells directly at Voxy level 4.
- Coarse leaves have their own persistent presence state instead of falsely claiming finer children exist.
- Before refinement becomes visible, a complete finer child is initialized by expanding its coarse parent. Fine predictions or authoritative real chunks then overlay that background.
- Direct writes skip regions already owned by finer children, so coarse work cannot replace real or higher-quality data.

### Terrain quality

- Optional adaptive sampling starts at stride 4 and progressively increases through power-of-two bands toward the configured horizon maximum.
- Optional terrain smoothing reconstructs continuous surfaces between sparse samples. The two nearest quality bands use one-block columns, while direct N-sized bands emit larger cells.
- Large height changes use continuous slope-aware interpolation, keeping cliffs steep without creating stride-sized towers.
- Most predicted terrain is only a four-block surface shell. A one-block reconstruction halo identifies exposed drops and extends only those visible cliff edges to the lower adjacent surface.
- Predicted outdoor air carries full skylight, preventing black terrain in daylight.
- Ocean floors remain represented, but water is a single flat visible layer at the active generator's datapack sea level. It is no longer smoothed as terrain or filled down to the seafloor, preventing water from climbing mountains.

### Lightweight vegetation proxies

- Deterministic biome-aware oak, birch, spruce, jungle, acacia, dark-oak, cherry, and mangrove-style silhouettes.
- Irregular layered crowns instead of simple leaf cubes.
- Tiered conifers and bent/branched flat-canopy trees.
- World-space local-minimum placement replaces fixed chunk quadrants, removing obvious rows and stripes.
- Placement is stable for a given seed but intentionally does not claim to reproduce Minecraft's exact decoration stage.
- Tree density fades continuously with player distance and never reaches a hard zero ring. Very distant bands retain sparse silhouettes without an obvious forest border.

### Datapack awareness

- Terrain shape and biome selection come from the active generator, so vanilla-style noise-settings datapacks participate automatically.
- A fast datapack mode uses early-exit height queries instead of constructing a complete vertical noise column for every sample.
- Fast mode requires one solid-surface height query per lattice point on both land and ocean.
- Custom-biome profiles are cached.
- Biome tags, identifiers, climate, and placed-feature identifiers help infer proxy tree density and species.
- Common vanilla-block surface themes such as grass, snow, stone, sand, gravel, mud, calcite, and basalt are approximated from biome metadata.

This is useful for datapacks such as Tectonic, Terralith, and other vanilla-style terrain/biome packs, but it does not execute their complete surface-rule or configured-feature stages. Unusual surfaces and custom blocks may be approximated incorrectly.

## Recommended starting settings

| Setting | Recommendation | Notes |
|---|---:|---|
| Seed LOD distance | `192` | Radius in chunks |
| Maximum sample stride | `64` | Fastest useful horizon tier for the 4 by 4-chunk wave tile |
| Seed LOD threads | `4` | Changes take effect immediately |
| Adaptive quality | On | Stride 4 nearby, then 8, 16, 32, and 64 toward the horizon |
| Adaptive quality band width | `32` | Each stride tier lasts 32 chunks |
| Direct N-sized generation | On | Skips unnecessary fine reconstruction in stride 16 through 64 bands |
| Smooth sampled terrain | On | Large quality gain for little additional sampling cost |
| Predicted vegetation | On | Cheap visual proxies |
| Fast datapack terrain sampling | On | Recommended for noise-settings datapacks |

The normal Voxy render distance must also be large enough to display the predicted range.

## Performance model

Without cross-tile reuse, each 64 by 64-block tile and its interpolation halo needs:

- 324 generator samples at stride 4.
- 100 samples at stride 8.
- 36 samples at stride 16.
- 16 samples at stride 32.
- 9 samples at stride 64.

At stride 64, the far tile performs about 11 times fewer generator queries than a stride-8 tile. A simple area-weighted model for the default five adaptive bands estimates roughly 6 times fewer generator queries than the seedlod.8 adaptive profile. This is a sampling estimate, not an end-to-end benchmark.

Seedlod.12 caches the globally aligned stride-16-and-higher samples across adjacent tiles. Over a large connected region, the amortized new generator samples approach 16 per tile at stride 16, 4 at stride 32, and 1 at stride 64. Compared with independently sampling every tile halo, that is approximately 2.25, 4, and 9 times fewer generator queries respectively. The cache is bounded at 500,000 samples.

Stride 64 is the useful cutoff for the current tile architecture. The 64 by 64-block tile plus its interpolation halo requires at least a 3 by 3 sample grid. Stride 128 still requires that same grid and the same 9 generator queries, while reconstruction, mipping, storage, and meshing also remain unchanged. It therefore provides worse terrain for no meaningful speed gain.

Seedlod.10 also removes a second bottleneck that sample stride alone did not fix. On flat land, older versions filled roughly one stride of solid blocks below every surface column. The four-block shell reduces flat-column voxel writes by about 2 times at stride 8, 4 times at stride 16, 8 times at stride 32, and 16 times at stride 64. Exposed cliff columns still extend far enough to close the visible face, so actual savings depend on terrain shape.

N-sized generation removes most of the remaining far-field reconstruction. A 64 by 64-block tile contains 4,096 one-block horizontal columns. Direct output reduces that to 256 N=4 columns at stride 16, 64 N=8 columns at stride 32, and 16 N=16 columns at stride 64. These are theoretical horizontal output reductions of 16, 64, and 256 times before vertical cliff, water, vegetation, hierarchy, storage, and meshing costs.

The largest savings happen at the horizon, which also contains most of the tiles. Terrain becomes visible near the player immediately and expands as workers finish each gated ring.

- Smoothing adds interpolation work but no additional generator queries.
- Direct coarse leaves are persisted separately from their finer-child mask. Refinement expands a complete parent fallback into the child before publishing it, addressing the ownership error that caused slabs and rectangular holes in seedlod.6 and seedlod.7.
- Ordinary movement keeps nearby queued work alive. Long teleports discard stale work and reprioritize the destination.
- Changing the maximum stride or adaptive-quality toggle while a world is open immediately cancels stale plans and rescans the wave at the new quality.
- Vegetation is hash-thinned before the more expensive local-minimum test. Density falls continuously with distance but never ends at a quality-band border.
- Predicted mapping IDs are cached per block state and biome, avoiding repeated mapper lock traffic across thousands of matching columns.
- Oceans use one visible sea-level water block per column instead of complete water volumes. This greatly reduces voxel writes and allocated vertical sections over deep water.
- Redesigned trees perform more tiny hash checks but produce roughly the same amount of leaf geometry as the earlier proxy implementation.
- A 1,024-chunk placement simulation averaged 3.95 vegetation candidates per chunk with no preferred local X/Z lane.
- Fast datapack sampling is expected to improve expensive noise-datapack sampling by roughly 1.5 to 4 times compared with this patch's former full-column approach. This is an engineering estimate, not a universal benchmark.
- Tectonic and similar packs may remain slower than vanilla because their density functions are inherently more expensive.

This is still not a custom planetary renderer. A 192-chunk radius contains roughly 115,000 chunk positions, and Voxy must still sample terrain, store sections, and build geometry. N-sized output removes most fine reconstruction at long distances, but exact speedups depend heavily on the generator, storage, CPU, and enabled vegetation.

### GPU offload finding

Voxy already uses OpenGL compute for visibility, node management, and rendering. The expensive seed prediction work is different: Minecraft's `ChunkGenerator`, density functions, biome sources, surface rules, and datapack codecs execute as Java objects on the CPU. OpenGL or OpenCL cannot directly execute those objects.

Offloading only interpolation would still require uploading samples and reading reconstructed block results back to Java before Voxy can assign mapping IDs, build its CPU voxel hierarchy, and store the cache. That readback introduces synchronization and can stall the render thread, so this patch does not provide a fake GPU toggle that is likely to run slower. A real generic GPU generator would require a separate shader compiler for Minecraft density functions and surface rules, plus a GPU-native Voxy ingestion path. That is a major renderer project rather than a safe setting.

The patch instead applies hardware-acceleration thinking where this architecture benefits: shared generator samples, native N-sized output, fewer mapper locks, fewer voxel writes, fewer vertical sections, bounded background threads, and Voxy's existing GPU renderer after ingestion.

Exact trees and structures would require most of Minecraft's generation pipeline through the `FEATURES` stage and a working neighborhood of temporary chunks. This patch deliberately stays approximate to retain its main performance advantage.

## Compatibility and limitations

- Minecraft Java `26.2`
- Fabric Loader
- Fabric API, Sodium, and the dependencies required by the matching Voxy development build
- Single-player Overworld only
- Multiplayer clients do not receive the server's complete seed/generator state
- No exact structures, caves, decorations, player edits, or exact decorated trees
- Direct N-sized generation is new experimental hierarchy code. Disable it if a renderer or storage compatibility problem appears
- A 1024-chunk radius contains millions of chunk positions. The slider allows it, but generation time and cache usage remain substantial even at stride 64
- Custom `ChunkGenerator` implementations fall back to the compatibility column path
- Experimental code: back up important worlds and Voxy caches

## Applying the patch

The patch targets Voxy commit `337b919d6638cce3d65264efb10b0d20cd060010` on the development line used for Minecraft 26.2.

```bash
git clone https://github.com/MCRcortex/voxy.git
cd voxy
git checkout 337b919d6638cce3d65264efb10b0d20cd060010
git apply /path/to/voxy-seed-lod-mc26.2.patch
```

Build using the JDK required by that Voxy/Minecraft version:

```bash
./gradlew build
```

On Windows:

```powershell
.\gradlew.bat build
```

The patch is available at [`patches/voxy-seed-lod-mc26.2.patch`](patches/voxy-seed-lod-mc26.2.patch).

## Cache migration

Seedlod.12 introduces a new persistent coarse-leaf layout. A clean cache is required when upgrading from any earlier seedlod version so old bottom-up predictions do not hide the new N-sized path. Close Minecraft, back up the world, and remove only that world's `<world save>/voxy` derived-cache folder before launching seedlod.12.

This cleanup is also required for caches touched by seedlod.6 and seedlod.7 direct-parent experiments or the original unmarked seedlod.1 experiment.

Do not delete the complete world folder.

## Verification

The published patch version compiled successfully and passed Voxy's Gradle build, test task, resource processing, and access-widener validation before publication. Seedlod.12's N-sized hierarchy is newly implemented and still needs broad in-game runtime testing across movement, restarts, seeds, and datapacks.

## Contributing

Please do improve it. In particular, an organic life form replacing the AI-slop portions with clearer architecture, tests, benchmarks, or a clean-room implementation would be genuinely appreciated.

Useful contribution areas include:

- Real profiling across vanilla, Tectonic, and Terralith seeds
- Better generic surface-rule approximation
- More natural proxy vegetation without running full decoration
- Safe structure-location silhouettes
- Automated visual and cache-migration tests
- Code cleanup, because the AI has undoubtedly hidden at least one tiny goblin in here

When reporting performance, include CPU, Java version, seed, datapacks, distance, stride, thread count, and whether adaptive quality/smoothing/vegetation were enabled.
