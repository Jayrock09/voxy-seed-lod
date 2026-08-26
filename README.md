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
- Uses a priority worker queue with at most 512 pending jobs. Sparse full-distance tiles always run before detailed refinement.
- Worker-thread changes take effect while the world is open.
- Slightly overlaps Minecraft's normal render boundary to hide the empty transition ring.
- Marks every prediction so later predictions may replace it while real ingested chunk data remains authoritative.
- Supports prediction distances from 32 to 512 chunks and sample strides of 2, 4, or 8 blocks.

### Instant sparse loading

- Divides the full prediction radius into Voxy's native 512 by 512-block top-level tiles.
- Covers the starting area as well as the distant horizon, so the sparse pass has no intentional gap around spawn.
- Samples each tile on a 32-block grid and writes the result directly into Voxy's top LOD instead of creating 1,024 detailed predicted chunks.
- A complete tile needs 289 shared terrain samples, including its positive boundary, instead of as many as 16,384 stride-8 chunk samples for the same area.
- Reconstructs the 16-block Voxy cells between samples with the existing terrain smoother.
- Adds coarse terrain skirts and water so mountains remain closed and oceans remain visible.
- Detects completed tiles in Voxy's persistent cache, so reopening a world does not resample them.
- Stops scanning after the current radius and only plans unseen tiles after the player moves at least eight chunks.
- Keeps a configurable progressive reconstruction radius for the full terrain, lighting, water, smoothing and vegetation path.

### Progressive quality reconstruction

- The complete low-quality radius is scheduled before any refinement work.
- Refinement uses aligned 16 by 16-chunk regions matching Voxy's hierarchy instead of scattered individual chunks.
- Very close regions use stride 2, medium-distance regions use stride 4, and the outer reconstruction band uses up to stride 8.
- The barely visible horizon remains at the fast 32-block sparse representation.
- A coarse parent remains visible while a complete replacement region is built.
- Renderer notifications are deferred and coalesced until the replacement hierarchy is complete, preventing partially generated children from deleting rectangular pieces of their coarse parent.
- Moving closer can upgrade an existing region, while moving away does not waste time downgrading it.

### Terrain quality

- Optional adaptive sampling: stride 2 near the normal render boundary, stride 4 at medium range, and the configured maximum stride farther away.
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
| Instant sparse loading | On | Fills the complete radius first |
| Progressive reconstruction distance | `64` | HQ nearby, medium quality farther out, sparse horizon beyond it |
| Maximum sample stride | `8` | Fastest; smoothing makes it substantially less blocky |
| Seed LOD threads | `4` | Changes take effect immediately |
| Adaptive quality | Optional | Only controls legacy full-radius mode; sparse reconstruction is always distance-tiered |
| Smooth sampled terrain | On | Large quality gain for little additional sampling cost |
| Predicted vegetation | On | Cheap visual proxies |
| Fast datapack terrain sampling | On | Recommended for noise-settings datapacks |

The normal Voxy render distance must also be large enough to display the predicted range.

## Performance model

With instant sparse loading enabled, a 192-chunk radius generally needs roughly 120 to 170 top-level tile jobs, depending on alignment with the 32-chunk tile grid. Each new tile performs 289 shared generator samples. The old full-radius detailed path covers roughly 115,000 individual chunks and can require about 1.8 million stride-8 samples before accounting for repeated borders.

After the sparse radius appears, only regions inside the progressive reconstruction distance use the detailed generator. At stride 8, each detailed predicted chunk uses a 4 by 4 sample grid including its halo: 16 sample positions rather than generating all 256 full terrain columns plus surface rules, carvers, features, lighting, entities, and chunk persistence.

- Smoothing adds interpolation and voxel writes, not additional seed samples.
- Sparse tiles write directly into Voxy's top LOD, avoiding per-chunk voxelization and four levels of repeated mipping for the far field.
- Coarse work has priority over refinement, so moving or teleporting exposes new distant terrain before optional detail jobs.
- Reconstruction is committed in complete aligned regions, so the visible parent should not disappear while its replacement is still being generated.
- Cached tiles are reused on later sessions.
- Redesigned trees perform more tiny hash checks but produce roughly the same amount of leaf geometry as the earlier proxy implementation.
- A 1,024-chunk placement simulation averaged 3.95 vegetation candidates per chunk with no preferred local X/Z lane.
- Fast datapack sampling is expected to improve expensive noise-datapack sampling by roughly 1.5–4× compared with this patch's former full-column approach. This is an engineering estimate, not a universal benchmark.
- Tectonic and similar packs may remain slower than vanilla because their density functions are inherently more expensive.

Exact trees and structures would require most of Minecraft's generation pipeline through the `FEATURES` stage and a working neighborhood of temporary chunks. This patch deliberately stays approximate to retain its main performance advantage.

## Compatibility and limitations

- Minecraft Java `26.2`
- Fabric Loader
- Fabric API, Sodium, and the dependencies required by the matching Voxy development build
- Single-player Overworld only
- Multiplayer clients do not receive the server's complete seed/generator state
- No exact structures, caves, decorations, player edits, or exact decorated trees
- Terrain outside the progressive reconstruction distance is intentionally coarse and does not contain vegetation proxies
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

Predictions from seedlod.2 and later carry a marker and can be replaced automatically. The first seedlod.1 experiment did not. If seedlod.1 ever generated a world's cache, close Minecraft, back up the world, and remove only that world's `<world save>/voxy` derived-cache folder before testing a newer version.

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
