# S60 Restoration Plan

## Goal

Produce high-quality restored 1080p episode files that remove the network bug while preserving as much of the existing 1080p source as technically possible.

The 1080p source is the master. The DVD source is a clean donor/reference used only where the 1080p image is obscured by the network bug.

A later 4K upscale may be performed separately. This project should therefore avoid unnecessary generation loss and preserve fine detail, grain/noise structure, cadence, audio, and metadata wherever practical.

## Core preservation rules

1. **Do not reprocess pixels that do not need restoration.**
   - The ideal output differs from the 1080p source only inside the network-bug footprint and only on frames where the bug is present.
   - Avoid whole-frame filtering, denoising, sharpening, resizing, color conversion, or other global image processing unless later evidence proves it necessary.

2. **Do not replace the 1080p image with an upscaled DVD image.**
   - The DVD is lower resolution and is used only to reconstruct content hidden by the bug.
   - Native 1080p pixels remain authoritative everywhere they are visible.

3. **Minimize codec generation loss.**
   - Preserve original streams by bitstream copy whenever a stream does not need modification.
   - Audio should normally be copied unchanged.
   - Subtitle, chapter, and compatible metadata streams should be copied unchanged when present.
   - Video must be re-encoded if even a subset of decoded frames is modified, but the encode should be designed to preserve the untouched regions as faithfully as possible.

4. **Never stack lossy encodes.**
   - All restoration work should operate from the original 1080p source plus the original DVD donor.
   - Intermediate working images should be lossless or uncompressed where practical.
   - The final delivery encode should be the first and only lossy re-encode of the restored video.

5. **Preserve source cadence.**
   - The 1080p source cadence, currently observed as 24000/1001 progressive for episode 1, is the delivery timeline.
   - The DVD's 30000/1001 representation should be inverse-cadence/duplicate-frame reduced only for donor alignment. It must not dictate output cadence.

6. **Prefer measured registration over assumptions.**
   - Match DVD and 1080p timelines piecewise where edits differ.
   - Register donor frames spatially against the corresponding 1080p picture.
   - Validate alignment outside the bug region before using donor pixels.

7. **Use the smallest defensible repair mask.**
   - The mask should cover the actual bug plus a small safety/feather margin, not a large rectangular corner by default.
   - Where possible, derive the mask empirically from persistent DVD↔1080p differences across multiple scenes.
   - Preserve original 1080p pixels right up to the edge of the repair where seam quality permits.

## Important codec reality

There is no general way to edit arbitrary pixels inside an H.264/AVC video while bit-for-bit preserving every other compressed video packet. H.264 compression predicts data spatially and temporally across blocks and frames, so changing a visible region normally requires decoding and re-encoding the video stream.

Therefore the preservation target is:

- **bitstream-copy every non-video stream that can be copied safely;**
- **perform only one final video encode;**
- **make the decoded output outside the repair region as close as practical to the original 1080p source;**
- **choose final codec/encoder settings that are effectively transparent at normal viewing and robust for later 4K upscaling.**

We should investigate whether constrained or selective re-encoding techniques can reduce changed compressed data, but they must not compromise compatibility, reliability, or quality. A conventional high-quality single re-encode may ultimately be safer than exotic partial-bitstream surgery.

## Source model

### 1080p
Role: authoritative master.

Preserve:
- frame geometry and framing;
- progressive cadence;
- all unobscured image content;
- audio bitstream where feasible;
- subtitle/chapter/metadata streams where feasible;
- color metadata unless testing shows it is erroneous.

### DVD
Role: clean donor/reference.

Use for:
- recovering pixels hidden by the network bug;
- temporal alignment;
- spatial registration;
- validation of bug location.

Do not use it to replace unobscured 1080p content.

### 720p
Role: secondary reference only.

The 720p versions also contain the network bug, so they are not a clean donor for pixels obscured in the 1080p source.

Potential uses:
- resolve ambiguous DVD↔1080p alignment;
- determine whether an apparent difference is source-specific;
- validate bug shape, timing, framing, opacity, or color;
- compare how the same overlay was encoded at a different resolution;
- help distinguish overlay artifacts from compression or registration artifacts.

The DVD remains the canonical clean donor for reconstruction unless another genuinely clean source is discovered.

## Technical work plan

### Phase 1: Source inventory and pairing

Status: substantially complete.

- Pair all 22 DVD and 1080p episodes by episode number.
- Record exact source paths.
- Characterize codecs, dimensions, frame rates, durations, audio, and framing.
- Add 720p characterization before production restoration.

Deliverables:
- `data/source-pairs.json`
- per-source metadata inventory.

### Phase 2: Temporal mapping

Status: episode 1 calibration in progress.

For each episode:

1. Use audio correlation to establish coarse correspondence between 1080p and DVD.
2. Detect discrete timeline/edit discontinuities.
3. Refine discontinuity boundaries only to the precision needed for frame correspondence.
4. Establish piecewise time-map segments.
5. Validate each segment at several anchors.
6. Detect and remove the DVD duplicate-frame cadence to recover donor frames corresponding to the 23.976 master timeline.

Do not assume episode 1's edit map applies to other episodes.

Deliverable:
- machine-readable per-episode time map with confidence/validation data.

### Phase 3: Spatial registration

For each stable timeline segment or shot as necessary:

1. Normalize active-picture geometry.
2. Upscale the DVD donor to the 1080p working geometry using a high-quality reconstruction filter.
3. Estimate scale/translation/rotation from unobscured picture content.
4. Exclude the network-bug region from transform fitting.
5. Measure residual error outside the bug.
6. Escalate to shot-specific registration only when segment-level registration is insufficient.

Prefer the simplest transform that meets quality thresholds. Do not use optical-flow warping merely because it is available.

### Phase 4: Network-bug characterization

1. Compare multiple independently registered clean/bugged frame pairs.
2. Find the spatially persistent difference attributable to the bug.
3. Determine:
   - exact footprint;
   - antialiased/translucent boundary;
   - whether opacity/color changes;
   - whether bug position changes;
   - when it appears/disappears;
   - whether different episodes use different bugs.
4. Build a conservative per-layout mask.
5. Validate the mask on varied backgrounds, including faces, motion, fine texture, dark scenes, bright scenes, and text.

The mask should be shaped to the overlay rather than being an unnecessarily large rectangle.

### Phase 5: Donor reconstruction

Inside the bug mask:

1. Select the temporally corresponding clean DVD frame.
2. Apply cadence normalization and spatial registration.
3. Transform only the donor region needed by the mask.
4. Match local luminance/chroma if required.
5. Composite with a small, controlled feather or multiband transition only where seam tests show a benefit.
6. Avoid modifying pixels outside the validated mask.

Potentially use multiple neighboring DVD frames to improve donor reconstruction if temporal information helps recover detail, but only after establishing a strong single-frame baseline.

### Phase 6: Prototype quality evaluation

Before restoring full episodes, generate representative short samples containing:

- static detailed backgrounds;
- faces crossing the bug;
- fast motion;
- camera movement;
- dark material;
- bright/high-contrast material;
- film/video noise;
- credits or graphics near the bug.

Compare:

- original 1080p;
- registered DVD donor;
- restored composite;
- difference image;
- seam metrics around the mask;
- temporal stability across consecutive frames.

Look specifically for:
- softness inside the patch;
- shimmer/flicker;
- mismatched grain/noise;
- ringing;
- edge halos;
- color mismatch;
- visible mask boundaries;
- temporal judder from incorrect DVD cadence mapping.

### Phase 7: Final encoding strategy

Do not choose final encode settings until restoration samples have been visually and quantitatively validated.

Baseline principles:

- output resolution: preserve 1920×1080;
- output cadence: preserve 24000/1001 where that is the native 1080p cadence;
- progressive output;
- one lossy video encode only;
- high-quality encoder settings aimed at visually transparent retention of the source;
- no unnecessary resize;
- no global denoise/sharpen;
- copy original audio bitstream when container/codec compatibility permits;
- copy subtitle/chapter/metadata streams where feasible.

Candidate final codecs should be tested against the user's later 4K-upscaling workflow. H.264 may offer maximum compatibility and source-like behavior; HEVC/AV1 may offer better compression efficiency but changing codec is not automatically desirable. The decision should be based on transparent-quality tests and practical file sizes, not novelty.

For archival/restoration intermediates, consider a lossless or near-lossless master in addition to the distribution copy if storage permits. This would prevent the final delivery codec from becoming the only preserved restored source for later 4K work.

### Phase 8: Whole-episode validation

For each restored episode:

- verify duration and frame count;
- verify A/V sync at beginning, middle, and end;
- verify all time-map transitions;
- verify bug removal throughout;
- scan for frames where the mask is applied unnecessarily;
- scan for missed bug pixels;
- inspect representative repair frames;
- verify audio stream identity when copied;
- verify no unintended resolution, aspect, cadence, or color-metadata change.

Automate as much of this as possible and emit a per-episode QA report.

### Phase 9: Batch production

Only after the pipeline passes episode 1 and at least a small cross-episode sample:

1. characterize all 22 episodes;
2. generate/validate time maps;
3. determine whether one bug mask/layout works across the season;
4. restore episodes independently;
5. encode once;
6. run automated QA;
7. retain logs/manifests sufficient to reproduce each output.

## Quality gates

A restoration method is not production-ready unless:

- unobscured 1080p content is not intentionally replaced;
- temporal registration is verified;
- spatial alignment is stable;
- repair seams are not visible in representative motion;
- donor softness is confined to the bug footprint;
- the output does not acquire unnecessary global filtering;
- audio is unchanged unless a documented reason requires otherwise;
- there is no additional intermediate lossy encode;
- resulting files are suitable as inputs to later high-quality 4K upscaling.

## Current episode 1 findings

See `docs/episode-01-calibration.md`.

Key current conclusions:

- DVD and 1080p require piecewise temporal mapping.
- DVD cadence contains a clean repeated-frame pattern consistent with 23.976-origin material represented at 29.97.
- Spatial registration appears tractable with a lightweight affine transform after active-picture normalization.
- Initial persistent-difference analysis points to a compact lower-left overlay region.
- Network-bug mask refinement is currently in progress.

## Near-term queue

1. Finish and visually review the current three-minute Episode 1 restoration validation.
2. Implement the streaming performance architecture in `docs/PERFORMANCE-ARCHITECTURE.md`.
3. Reproduce the same three-minute window through the single-shard streaming renderer and prove pre-encode pixel parity with the validated prototype.
4. Benchmark 1, 2, 4, and 8 shot-aligned render shards; select the default based on wall-clock time, total runner-minutes, transfer overhead, and artifact overhead.
5. Finalize Episode 1's complete piecewise time map, including remaining head/tail and timing-boundary work.
6. Run the first whole-Episode-1 diagnostic restoration only after streaming parity and sharding validation pass.
7. Complete whole-episode QA and choose the final encode strategy.
8. Generalize characterization, mask/layout validation, and timeline mapping across all 22 episodes.
9. Keep the 720p source as secondary validation/reference only, since it also contains the network bug.
