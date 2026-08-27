# Voxy Seed LOD Rewrite Specification

## Goal

Rewrite Seed LOD as a small, honest render-chunk generator.

Given the integrated server's seed and active datapacks, generate the real visible world content that matters to Voxy:

- Real terrain density and aquifers
- Real biome assignment and surface rules
- Real carvers where they affect visible terrain
- Real structure starts and structure blocks
- Real biome vegetation and other explicitly visible placed features
- Real skylight and block-light values for the generated render region

The rewrite must remain faster and lighter than loading ordinary Minecraft chunks by omitting systems that do not affect the distant image:

- No persistent Minecraft chunk registration
- No region-file chunk saves
- No entity creation or mob spawning
- No entity ticking
- No points of interest
- No block ticks or fluid ticks
- No networking
- No client chunk objects
- No advancement, loot, map, or gameplay bookkeeping
- No long-lived block entities

Temporary generation data must be discarded immediately after Voxy extraction.

## Non-negotiable architecture

Use Minecraft's active `ChunkGenerator` and datapack registries. Do not imitate terrain, trees, structures, or surfaces with biome-name rules or handmade proxy geometry.

Generate reusable disposable regions rather than isolated chunks. A region contains:

1. A target area that will be published to Voxy.
2. A terrain and feature halo so vegetation crossing a chunk boundary is complete.
3. A structure-start halo large enough to discover structures intersecting the target.

Run only the generation stages required for the final render:

```text
structure starts in the structure halo
biomes in the terrain region
noise and aquifers in the terrain region
surface rules in the terrain region
carvers in the terrain region
structure references for the target and feature halo
structure placement and biome decoration for the target and feature halo
render-only lighting for the completed disposable region
N-sized Voxy extraction for the target
discard temporary region
```

Do not run spawn or full-chunk conversion.

## N-sized generation

Keep direct N-sized output.

- Near the player, preserve one-block detail before Voxy mipping.
- At medium distance, extract directly into larger Voxy cells.
- At the horizon, reduce output and meshing work without changing the Minecraft generation input required for correctness.
- All coarse jobs must publish complete aligned Voxy sections.
- A finer child may replace a coarse prediction only after complete sibling coverage exists.
- Real loaded chunks always replace disposable predictions and can never be overwritten by them.

N-sized extraction is an output optimization. It must not change structure positions, tree placement, biome selection, surface rules, or terrain generation.

## Lighting

Lighting must be derived from the generated block states, not hardcoded full brightness.

- Seed skylight from open sky columns.
- Propagate skylight through the disposable region using Minecraft block opacity.
- Seed block light from actual light-emitting generated blocks.
- Propagate block light through the disposable region.
- Use a one-chunk lighting halo and discard the halo after target extraction.
- Mark boundary values as provisional when a missing neighboring region could change them, then repair them when adjacent output becomes available.

## Structures and vegetation

- Use real structure placement and template data from the active datapacks.
- Use real biome decoration and placed-feature ordering.
- Capture only blocks relevant to rendering.
- Discard entities, scheduled ticks, block-entity runtime objects, loot initialization, and POI registration.
- Preserve visible block states from chests, banners, signs, and other structure blocks, but do not create their gameplay data unless required to choose the visible state.

## Scheduling and memory

- Generate nearest regions first.
- Bound queued work and temporary memory.
- Reuse halos between neighboring requests where safe.
- Cancel stale work after teleporting.
- Never expose disposable chunks through the live server chunk source.
- Never save disposable chunks.
- Make thread count configurable, but respect generator thread-safety and route unsafe stages through a controlled lane.

## Validation

Add automated tests for:

- Disposable chunks never reaching the live chunk map or storage.
- Real chunks always replacing predictions.
- Complete hierarchy coverage during coarse-to-fine refinement.
- Structures crossing target-region borders.
- Trees crossing chunk borders.
- Skylight and block-light propagation.
- Negative coordinates.
- Teleport cancellation.
- Datapack fingerprint invalidation.
- Bounded temporary memory.

Add a benchmark that separately reports generation-stage time, lighting time, Voxy extraction time, temporary peak memory, regions per second, and chunks-equivalent per second.

## Success criteria

The rewrite succeeds when distant terrain, surfaces, vegetation, structures, and lighting come from Minecraft's actual seed-backed generation rules, while the discarded render-only path is measurably faster and lighter than loading and retaining the equivalent ordinary chunks.

If exact execution of a requested stage is unavailable for a custom generator, report the region as unsupported. Do not silently replace it with fake visual content.
