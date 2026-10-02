# S60 Restoration Plan

## Goal

Produce high-quality restored 1080p episode files that remove the network bug while preserving as much of the existing 1080p source as technically possible.

The 1080p source is the master. The DVD source is a clean donor/reference used only where the 1080p image is obscured by the network bug.

A later 4K upscale may be performed separately. This project should therefore avoid unnecessary generation loss and preserve fine detail, grain/noise structure, cadence, audio, and metadata wherever practical.

## Core preservation rules

1. **Do not reprocess pixels that do not need restoration.**
   - The ideal output differs from the 1080p source only inside the network-bug footprint and only on frames where the bug is present.
   - Avoid whole-frame filtering, denoising, sharpening, resizing, color conversion, or other global image processing unless later evidence proves it necessary.

2. **Do not replace the 1080p image with an upscaled DVD image by default.**
   - The DVD is lower resolution and is used primarily to reconstruct content hidden by the bug.
   - Native 1080p pixels remain authoritative everywhere they are visible unless a controlled pre-production comparison demonstrates that a full-frame DVD upscale is genuinely equal or better for the intended viewing/restoration goal.
   - AI or learned upscaling may be used selectively on the DVD donor region, including only the lower-left corner around the bug, if it materially improves mask estimation, registration, edge matching, or donor reconstruction without modifying unrelated 1080p pixels.

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

### Phase 6: Prototype quality evaluation and pre-production upscale gate

**No full episode should be processed until this gate has been completed.**

Use the same representative short time range for every candidate. The sample should include as many of the following as practical:

- static detailed backgrounds;
- faces, hair, skin, clothing, and other natural fine detail;
- faces or textured objects crossing the network-bug region;
- fast motion;
- camera movement;
- dark material;
- bright/high-contrast material;
- film/video noise or grain;
- text, credits, or graphics;
- scenes where the bug region contains detail that makes donor softness easy to judge.

Generate at least these controlled 1920×1080 candidates from the original sources:

**A — native-1080p restoration baseline**
- Start from the original 1080p source.
- Repair only the validated bug footprint using the clean DVD donor and the current restoration workflow.
- Preserve native 1080p picture content everywhere else.
- This represents the preservation-first workflow.

**B — full-frame pristine-DVD upscale**
- Start from the original clean 480p/DVD source for the identical time range.
- Normalize cadence and framing correctly.
- Upscale the complete clean picture to 1920×1080 with the strongest practical free/open modern upscaling model under test.
- Do not add unrelated grading, sharpening, denoising, or enhancement unless it is an inherent and documented part of the model/configuration.
- This tests whether modern reconstruction from the pristine lower-resolution source is perceptually equal to or better than repairing the bugged 1080p source.

**C — native 1080p plus upscaled DVD repair donor**
- Start from the original 1080p source.
- Upscale only the DVD pixels needed for the bug-region donor, preferably with a small registration/safety margin rather than the whole frame.
- Composite that enhanced donor into the validated mask.
- Preserve native 1080p pixels outside the repair region.
- This is a likely high-value hybrid because learned reconstruction is confined to pixels for which the clean high-resolution source does not exist.

**D — optional upscaled-corner reference for mask/registration**
- If direct DVD↔1080p comparison is too soft or unstable to define the mask or registration confidently, upscale only the lower-left DVD corner containing and surrounding the clean reference area.
- Use this enhanced corner as an analysis/reference image for bug-footprint estimation, registration, edge localization, and/or seam design.
- It need not become part of the final composite. Its first role is to provide a sharper clean reference against the 1080p bugged corner.
- Keep enough surrounding clean image outside the expected bug footprint to measure alignment and detect model-created edge artifacts.
- Validate any mask inferred from the AI-upscaled reference against the original DVD and multiple scenes so hallucinated detail cannot silently redefine the bug boundary.

For A, B, and C, use identical output geometry, cadence, clip boundaries, and final comparison encoding. Prefer lossless or near-lossless comparison intermediates so codec differences do not dominate the judgement.

The comparison package should include:

- complete synchronized candidate clips;
- lossless PNG stills from identical frames;
- enlarged crops of faces, hair, text, textures, edges, and the repaired/former-bug region;
- side-by-side and split-screen video;
- an alternating A/B/C presentation, preferably with a blind or semi-blind viewing pass before labels are revealed;
- difference images where they are diagnostically useful;
- objective metrics where meaningful, while treating visual temporal quality as authoritative when metrics disagree.

Evaluate two spatial regimes separately:

1. **Outside the bug footprint**
   - A and C retain genuine 1080p source detail.
   - B contains reconstructed detail from the DVD.
   - Judge whether the full-frame upscale actually reproduces, loses, or invents detail relative to the native 1080p source.

2. **Inside and immediately around the bug footprint**
   - Compare native-filtered DVD repair against learned-upscale donor reconstruction.
   - Inspect seam quality, texture continuity, edge fidelity, temporal stability, and whether the learned model creates plausible but false detail.

Look specifically for:
- softness inside the patch;
- shimmer/flicker or other temporal instability;
- hallucinated detail;
- facial or text deformation;
- mismatched grain/noise;
- ringing;
- edge halos;
- color mismatch;
- visible mask boundaries;
- inconsistent sharpness across the mask;
- temporal judder from incorrect DVD cadence mapping.

**Decision rule before batch production:**
- If A is clearly superior overall, continue with the preservation-first native-1080p workflow.
- If B is genuinely equal or better over representative material, reconsider whether full-frame pristine-DVD upscaling is a simpler or better production path.
- If C gives the best repair while retaining the native 1080p advantage elsewhere, adopt selective learned upscaling for the donor region only.
- D may be adopted independently as an analysis aid even if its pixels are never used in final output.
- Record the tested model, exact version, weights, parameters, preprocessing, hardware/backend, and hashes/configuration needed to reproduce the result.

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
- resulting files are suitable as inputs to later high-quality 4K upscaling;
- the Phase 6 A/B/C upscale gate has been completed before any full-episode production run;
- any AI-upscaled donor or mask-reference region has been checked for hallucinated edges/detail and temporal instability rather than trusted solely because it appears sharper.

## Current episode 1 findings

See `docs/episode-01-calibration.md`.

Key current conclusions:

- DVD and 1080p require piecewise temporal mapping.
- DVD cadence contains a clean repeated-frame pattern consistent with 23.976-origin material represented at 29.97.
- Spatial registration appears tractable with a lightweight affine transform after active-picture normalization.
- Initial persistent-difference analysis points to a compact lower-left overlay region.
- Network-bug mask refinement is currently in progress.

## Near-term queue

1. Finish the episode 1 network-bug mask experiment.
2. Produce several actual restored-frame prototypes.
3. Compare hard-mask, feathered-mask, and local color/luma-matched compositing.
4. Test temporal stability over the existing short sample.
5. Build and run the Phase 6 controlled A/B/C upscale comparison on the same sample before processing any full episode.
6. If mask estimation or donor matching benefits, add candidate D: AI-upscale only the lower-left clean DVD reference corner and test whether it improves registration/mask precision without introducing false boundaries.
7. Use the 720p source only as a secondary validation/reference source, since it also contains the network bug.
8. Lock the restoration path and minimum-quality final encode strategy only after the visual comparison is complete.
9. Generalize characterization and timeline mapping across all 22 episodes.
