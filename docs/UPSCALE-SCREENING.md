# 90-second DVD upscale screening

## Purpose

Before spending more time on network-bug masking, test whether a clean DVD-derived 1080p reconstruction is already visually competitive with the bugged native 1080p source.

If a clean full-frame DVD upscale is visually equal or preferable over representative material, the project may be able to avoid masking entirely. If masking remains clearly better, this experiment still establishes whether AI upscaling is useful as a selective donor or mask-reference aid.

## Test window

Episode 1:

- HD: 1760.0–1850.0 s
- DVD: corresponding +1.17 s timing plateau
- target duration: 90 s
- expected content: many scene changes based on the existing three-minute analysis

This span was chosen because it stays inside one known timing plateau while still exercising multiple edits and shot types.

## Input normalization

DVD:

- original 848×476 source
- deterministic inverse cadence with `decimate=cycle=5`
- crop to measured active picture: 848×464 at y=6
- neural tools operate on this clean active image

Output normalization:

- neural result reduced/normalized to 1904×1072 active geometry
- padded to 1920×1080 at x=10, y=4
- all candidates encoded for review with the same x264 CRF 10 / slow settings

Native reference:

- same 90 s HD interval
- 1920×1080, 24000/1001
- encoded with the same review-quality policy

The comparison is perceptual, not a claim that the DVD and HD masters have perfectly identical shot framing. Small source-framing differences should be distinguished from detail/reconstruction quality.

## Candidates

### Native 1080p reference

Bugged network master. This is the detail/reference target outside the bug footprint.

### Lanczos control

Conventional full-frame resize of the clean DVD. This shows how much benefit comes from neural reconstruction rather than simply scaling the clean source well.

### Real-ESRGAN

- implementation: portable `realesrgan-ncnn-vulkan`
- model: `realesr-general-x4v3`
- inference scale: 2×, followed by shared Lanczos normalization to exact HD geometry
- rationale: conservative general-scene model with weaker deblur/denoise behavior than the more aggressive x4 path; avoids asking the model to invent four times the spatial resolution when the target is only roughly 2.25× horizontally / 2.31× vertically
- role: perceptual real-world restoration / super-resolution candidate

### RealSR

- implementation: portable `realsr-ncnn-vulkan`
- model family: DF2K
- TTA: enabled (`-x`) following the project's own quality-oriented reference invocation
- role: independent real-world super-resolution candidate

### SRMD

- implementation: portable `srmd-ncnn-vulkan`
- scale: 4×
- denoise level: -1 (disabled)
- role: conservative super-resolution candidate without intentionally adding denoise

## Live-action DVD configuration rationale

Before locking the screening configuration, current project guidance and community practice for older live-action DVD material were reviewed. The resulting principles are:

- correct cadence/interlace state before super-resolution rather than asking the model to repair temporal structure;
- avoid automatic/aggressive denoise where the source is already reasonably clean;
- avoid face-enhancement models because they can fabricate identity/detail and create temporally unstable faces;
- prefer conservative reconstruction and smaller effective scale where possible;
- inspect moving faces, hair, text, fine textures, and shot changes rather than judging only static sharpness;
- preserve grain/noise structure rather than treating all source texture as damage.

No synthetic grain is added in this screening because the immediate question is whether the clean DVD can reconstruct detail competitively with the native 1080p source. Grain can be reconsidered later as a presentation choice, not used to conceal model artifacts during evaluation.

## Parallel execution

The 90 s source is split into six 15 s shards for each neural method.

Matrix:

- 3 methods
- 6 shards each
- 18 independent upscale jobs
- `max-parallel: 18`

Each shard:

1. downloads the common prepared 90 s DVD artifact;
2. extracts only its 15 s frame range;
3. runs one upscaler;
4. normalizes to the target HD geometry;
5. encodes a high-quality review chunk;
6. records neural inference wall time and input frame count.

Method assembly concatenates the six chunks without another video re-encode.

The final comparison job produces:

- full 90 s native reference;
- full 90 s Lanczos control;
- full 90 s output from each neural method;
- pairwise side-by-side videos against native 1080p;
- full-resolution stills at 5, 20, 35, 50, 65, and 80 s;
- runtime summary for all neural candidates.

## Evaluation

Judge full-frame candidates mainly outside the network-bug footprint because the native 1080p reference is contaminated there.

Inspect:

- faces and skin texture;
- hair;
- fabric;
- fine set texture;
- text/signage;
- high-contrast edges;
- camera movement;
- motion;
- scene changes;
- dark scenes;
- compression texture/noise;
- temporal stability.

Look specifically for:

- genuine detail retention/reconstruction;
- invented or unstable detail;
- plastic/waxy texture;
- ringing or halos;
- oversharpening;
- temporal shimmer;
- face deformation;
- text deformation;
- compression enhancement;
- inconsistent sharpness between shots.

## Decision logic

### If one clean-DVD upscale is visually as good as native 1080p

Strongly consider abandoning masking for full-frame DVD reconstruction, subject to:

- no unacceptable temporal artifacts;
- acceptable processing time;
- cross-episode confirmation;
- later 4K-upscale compatibility.

### If native 1080p is clearly better outside the bug

Keep the preservation-first masking path as the main candidate.

Use this experiment to decide whether neural upscaling is still valuable for:

- only the masked donor region;
- only the lower-left analysis/reference region;
- particular difficult shots.

### Runtime matters

Visual parity alone is not sufficient if full-frame AI upscaling is dramatically slower than a corrected, optimized masking pipeline.

Record:

- max shard wall time;
- total method compute seconds;
- frames processed;
- effective frames/second;
- observed workflow wall time.

Compare these with the optimized masking architecture once its streaming/sharded implementation exists.

## Masking work preserved

Current masking findings are frozen in:

- `docs/MASKING-RESTORATION-OBSERVATIONS.md`

Do not resume masking from scratch. The next masking experiment, if needed, is local residual registration at the repair region on top of deterministic cadence, per-shot donor phase, shaped alpha, and bounded local tone matching.
