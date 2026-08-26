> [!WARNING]
> **AI USE DISCLOSURE:** Generative AI was used heavily to design, implement, debug, document, and iterate on this experimental mod. Some of this is therefore, affectionately and occasionally literally, **AI slop**. I would be more than happy for an organic life form to audit it, clean it up, or remake the idea without the slop. Human-written improvements and clean-room reimplementations are extremely welcome.

# Voxy Seed LOD

An unofficial experimental patch for [Voxy](https://github.com/MCRcortex/voxy) that predicts distant Overworld terrain directly from the integrated server's seed-backed `ChunkGenerator` without requesting or loading every faraway Minecraft chunk.

The goal is simple: fill Voxy's distant horizon quickly enough to be useful at 128–512 chunk distances while keeping ordinary generated chunks authoritative when the player reaches them.

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
- Supports prediction distances from 32 to 512 chunks and sample strides of 4, 8, 16, 32, 64, or 128 blocks.

### Player-centered connected loading wave

- Divides prediction work into 4 by 4-chunk tiles and starts with the tile containing the player.
- Schedules complete square rings from nearest to farthest. Ring N+1 cannot start until every in-range tile in ring N finishes, so unfinished work stays on the outer edge instead of leaving holes behind it.
- Sorts the 16 chunks inside each tile by player distance before inserting them.
- Shares one generator sample grid across the whole tile, removing repeated chunk-border and halo queries.
- Uses Voxy's ordinary bottom-up `WorldUpdater` path for every predicted chunk. It does not inject giant parent slabs into the top LOD.
- Keeps real ingested chunks authoritative and safely upgrades predicted tiles when the player moves closer.
- Recenters after four chunks of movement. A teleport of at least 32 chunks cancels stale queued work and begins a new wave at the destination.

### Distance-based quality

- With adaptive quality enabled, nearby tiles use stride 4 and outward bands progressively use stride 8, 16, 32, 64, and up to 128.
- Sampling is aligned to a global world-space lattice, so strides larger than the 64-block loading tile remain continuous across tile borders.
- The selected quality is decided before a tile is generated, so there is no destructive coarse-parent replacement phase.
- Moving closer can regenerate an existing predicted tile at a finer stride. Moving away does not waste time downgrading completed data.
- Voxy still chooses its rendered screen-space LOD normally after the predicted chunks enter the cache.

### Terrain quality

- Optional adaptive sampling starts at stride 4 and progressively increases through power-of-two bands toward the configured horizon maximum.
- Optional terrain smoothing reconstructs one-block columns between sparse samples.
- Large height changes use continuous slope-aware interpolation, keeping cliffs steep without creating stride-sized towers.
- Neighbor-aware terrain skirts close steep undersides and chunk-border holes.
- Predicted outdoor air carries full skylight, preventing black terrain in daylight.
- Ocean floors and complete water columns are represented.

### Lightweight vegetation proxies

- Deterministic biome-aware oak, birch, spruce, jungle, acacia, dark-oak, cherry, and mangrove-style silhouettes.
- Irregular layered crowns instead of simple leaf cubes.
- Tiered conifers and bent/branched flat-canopy trees.
- World-space local-minimum placement replaces fixed chunk quadrants, removing obvious rows and stripes.
- Placement is stable for a given seed but intentionally does not claim to reproduce Minecraft's exact decoration stage.
- Tree proxies stop after stride 16. Stride 32 through 128 omit vegetation to keep the barely visible horizon cheap.

### Datapack awareness

- Terrain shape and biome selection come from the active generator, so vanilla-style noise-settings datapacks participate automatically.
- A fast datapack mode uses early-exit height queries instead of constructing a complete vertical noise column for every sample.
- Most land samples require one height query; possible ocean samples add an ocean-floor query.
- Custom-biome profiles are cached.
- Biome tags, identifiers, climate, and placed-feature identifiers help infer proxy tree density and species.
- Common vanilla-block surface themes such as grass, snow, stone, sand, gravel, mud, calcite, and basalt are approximated from biome metadata.

This is useful for datapacks such as Tectonic, Terralith, and other vanilla-style terrain/biome packs, but it does not execute their complete surface-rule or configured-feature stages. Unusual surfaces and custom blocks may be approximated incorrectly.

## Recommended starting settings

| Setting | Recommendation | Notes |
|---|---:|---|
| Seed LOD distance | `192` | Radius in chunks |
| Maximum sample stride | `128` | Fastest horizon; global alignment and smoothing prevent tile seams |
| Seed LOD threads | `4` | Changes take effect immediately |
| Adaptive quality | On | Stride 4 nearby, then 8, 16, 32, 64, and 128 toward the horizon |
| Smooth sampled terrain | On | Large quality gain for little additional sampling cost |
| Predicted vegetation | On | Cheap visual proxies |
| Fast datapack terrain sampling | On | Recommended for noise-settings datapacks |

The normal Voxy render distance must also be large enough to display the predicted range.

## Performance model

The wave loader shares globally aligned generator samples across each 64 by 64-block tile. Including the interpolation and terrain-skirt halo, a tile needs:

- 324 generator samples at stride 4.
- 100 samples at stride 8.
- 36 samples at stride 16.
- 16 samples at stride 32.
- 9 samples at stride 64 or 128.

At stride 128, the far tile performs about 11 times fewer generator queries than a stride-8 tile. A simple area-weighted model for the default six adaptive bands estimates roughly 6 times fewer generator queries than the seedlod.8 adaptive profile. This is a sampling estimate, not an end-to-end benchmark.

The largest savings happen at the horizon, which also contains most of the tiles. Terrain becomes visible near the player immediately and expands as workers finish each gated ring.

- Smoothing adds interpolation and voxel writes, not additional seed samples.
- Every tile follows Voxy's normal hierarchy-building path, eliminating the unsupported top-level parent replacement that caused slabs and rectangular holes in seedlod.6 and seedlod.7.
- Ordinary movement keeps nearby queued work alive. Long teleports discard stale work and reprioritize the destination.
- Changing the maximum stride or adaptive-quality toggle while a world is open immediately cancels stale plans and rescans the wave at the new quality.
- Ultra-coarse stride 32 through 128 skips proxy vegetation, reducing hashing and geometry work at the horizon.
- Redesigned trees perform more tiny hash checks but produce roughly the same amount of leaf geometry as the earlier proxy implementation.
- A 1,024-chunk placement simulation averaged 3.95 vegetation candidates per chunk with no preferred local X/Z lane.
- Fast datapack sampling is expected to improve expensive noise-datapack sampling by roughly 1.5–4× compared with this patch's former full-column approach. This is an engineering estimate, not a universal benchmark.
- Tectonic and similar packs may remain slower than vanilla because their density functions are inherently more expensive.

This is a scheduling and sampling acceleration, not a custom planetary renderer. A 192-chunk radius still contains roughly 115,000 chunks, and Voxy still has to reconstruct columns, mip them, store them, and build geometry. Expect a much cleaner and faster outward fill, not the instant 65,536-block coverage shown by a purpose-built heightmap LOD renderer.

Exact trees and structures would require most of Minecraft's generation pipeline through the `FEATURES` stage and a working neighborhood of temporary chunks. This patch deliberately stays approximate to retain its main performance advantage.

## Compatibility and limitations

- Minecraft Java `26.2`
- Fabric Loader
- Fabric API, Sodium, and the dependencies required by the matching Voxy development build
- Single-player Overworld only
- Multiplayer clients do not receive the server's complete seed/generator state
- No exact structures, caves, decorations, player edits, or exact decorated trees
- The far horizon uses the configured coarse sampling stride, but still travels through ordinary Voxy chunk ingestion
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

Seedlod.6 and seedlod.7 wrote experimental data directly into Voxy's top LOD. Seedlod.8 and later deliberately remove that path, but an existing cache may still contain those old slabs. For a clean current-version test, close Minecraft, back up the world, and remove only that world's `<world save>/voxy` derived-cache folder before launching the new build.

This cleanup is also required if the original unmarked seedlod.1 experiment ever generated the cache. Predictions from the ordinary chunk path in seedlod.2 through seedlod.5 carry a marker and can be replaced automatically.

Do not delete the complete world folder.

## Verification

The published patch version compiled successfully and passed Voxy's Gradle build, test task, resource processing, and access-widener validation before publication. Runtime visuals have been iterated through user testing; hardware, datapacks, and seeds will still vary.

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
