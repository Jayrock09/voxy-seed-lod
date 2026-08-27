# Rewrite changelog

## rewrite.4

- Fixed disposable chunks being downgraded from `CARVERS` to `STRUCTURE_REFERENCES` immediately before feature placement.
- Moved structure references into vanilla's correct stage order before biome and terrain generation.
- Made every disposable chunk status update monotonic so completed generation stages cannot be lost.
- Added regression tests for backward and forward chunk-status transitions.
- Bumped the prediction schema to `103` so failed rewrite.3 output is treated as stale.

## rewrite.3

- Fixed feature placement failing when a placement modifier checked the biome beyond the previous two-chunk decoration halo.
- Primed biome containers across the complete disposable structure region before any terrain or feature stages run.
- Kept noise terrain, surface generation, carvers, decoration, and lighting limited to their required generation halos.
- Bumped the prediction schema to `102` so incomplete rewrite.2 output is treated as stale.

## rewrite.2

- Fixed the first in-game failure, `Asking for biomes before we have biomes`.
- Changed disposable terrain generation to finish each status across the required region before advancing to the next status.
- Added an extra biome halo for surface rules, carvers, and decoration that query neighboring chunks.
- Added a five-second failed-region retry delay to prevent worker and chat log storms.
- Bumped the prediction schema so output from the broken pipeline is treated as stale.

## rewrite.1

- Replaced sampled terrain reconstruction with disposable real Minecraft world generation.
- Added real noise terrain, aquifers, surface rules, carvers, structure placement, and biome decoration.
- Added real vanilla sky-light and block-light propagation over a temporary halo.
- Removed proxy trees, biome-name surface guesses, smoothing, selective fake features, presets, sample-stride controls, and automatic quality targeting.
- Kept distance-based N-sized Voxy output with complete aligned hierarchy publication.
- Added 8 by 8 distant generation batches to amortize structure and feature halo work.
- Protected real Voxy sections, including empty sections, from later predicted writes.
- Added generator fingerprint invalidation and a real-generation diagnostics HUD.
- Reduced the settings page to enable, distance, high-detail radius, threads, and diagnostics.
- Added scheduler, fingerprint, negative-coordinate, concurrent hierarchy, and saved hierarchy repair tests.
- Verified the patch by applying it to a clean checkout of Voxy commit `337b919d6638cce3d65264efb10b0d20cd060010` and running the full Gradle build.

This is an experimental architecture build. In-game correctness and performance across vanilla and datapack generators still require field testing.
