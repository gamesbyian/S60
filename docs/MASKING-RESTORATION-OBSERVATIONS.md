# Masking / donor-restoration observations

This document freezes the current state of the Episode 1 masking and donor-compositing investigation so the project can pivot to full-frame DVD upscaling without losing the restoration thread.

## Source and timing facts already established

- 1080p source is the authoritative master and contains the network bug.
- DVD source is clean and is the reconstruction donor.
- 720p source also contains the bug and is reference-only.
- Episode 1 1080p is 1920×1080 at 24000/1001 progressive.
- Episode 1 DVD is 848×476 at 30000/1001 and represents 23.976-origin material with a repeated-frame cadence.
- DVD and 1080p do not share one global offset. Episode 1 requires piecewise temporal mapping.
- Content-adaptive `mpdecimate` is unsafe for this donor. In low-motion material it removed additional frames and silently drifted the donor timeline.
- Deterministic `decimate=cycle=5` fixed that failure. It is the current inverse-cadence baseline.
- Per-shot donor phase search within a small ±3 cadence-normalized-frame window proved useful and stable.

## Bug localization / mask

The lower-left NBC + `.com` overlay was localized empirically.

Conservative search ROI:

- active-picture coordinates: approximately x=170..384, y=890..982
- full-frame coordinates: approximately x=180..394, y=894..986
- conservative rectangle: 214×92 pixels, about 0.95% of a 1920×1080 frame

A dark calibration frame around 2200 s produced a logo-shaped alpha mask with:

- active-picture mask bbox approximately x=185..371, y=890..968
- alpha > 1%: ~8,012 pixels
- alpha > 50%: ~6,415 pixels
- affected full-frame fraction: ~0.386% at alpha > 1%
- strong-replacement fraction: ~0.309%

The shaped mask was visually much better than a rectangular replacement and remains the preferred mask form if donor compositing is resumed.

## Rejected approaches

### Hard rectangular replacement

Rejected visually. It produces obvious rectangular seams and modifies far too much native 1080p image.

### Feathered rectangular replacement

Improved the hard edge but still modifies an unnecessarily large area and remains visibly different.

### Broad mean/std photometric matching

Rejected. On the earlier 900 s prototype the fit hit gain/bias clamps and created a visible tonal slab. Any future tonal correction must be local, robust, tightly clamped, and applied only to donor pixels beneath the shaped mask.

## Successful short-form evidence

A six-second stable-motion validation around 2197–2203 s worked well once deterministic cadence correspondence was respected:

- 143/143 registrations accepted
- representative repaired-logo residual ~0.77 luma levels to the registered clean donor
- pre-repair residual roughly ~52 luma levels
- temporal change inside/around repair remained plausible

A separate 60-second mixed-shot dry run across three 20-second clips established:

- deterministic inverse cadence is essential;
- per-shot donor phase is useful;
- robust shot-level transform fallback can recover low-feature frames;
- all three clips reached 100% recoverable registration coverage after cadence correction.

These results showed that timing and broad registration are tractable, but they did not prove long-form visual invisibility of the repair.

## Three-minute test: critical finding

A 180.013 s test containing 4,316 frames and 59 detected shots passed all numerical gates:

- unsupported shots: 0
- raw anchor registrations: 384
- shot donor-match MAD median: ~3.51
- shot donor-match MAD p95: ~5.51
- temporal repaired-ROI delta mean: ~2.32–2.36
- temporal repaired-ROI delta p95: ~9.49–9.56

Despite those strong metrics, visual inspection of the actual rendered clip failed.

### Visual symptom

Across many shots, the lower-left repair remains perceptible as an NBC.com-shaped dark/embossed ghost. In some frames the interior contains mismatched image structure, not merely a brightness offset.

This is the most important current lesson: registration/temporal metrics alone are not sufficient acceptance criteria. The artifact must be visually inspected over motion and cuts.

## Tone-fit experiment

Hypothesis: the logo-shaped ghost might be caused primarily by local DVD↔HD luminance/chroma mismatch.

Experiment:

- estimate a robust local channel-wise donor→HD fit from a narrow clean ring around the logo;
- trim outlier differences;
- clamp gain to approximately ±6%;
- clamp bias to approximately ±8 code values;
- apply the correction only to donor pixels beneath the shaped alpha.

Result:

- numerical metrics remained green;
- visual ghost remained;
- therefore tone mismatch alone does not explain the failure.

## Current leading geometry hypothesis

The next active masking hypothesis is that the whole-shot affine is sufficiently accurate for global metrics but leaves a small local residual in the lower-left corner. A 1–few-pixel local mismatch is enough for the clean donor itself to become a logo-shaped visible patch after compositing.

A follow-up experiment was prepared that:

- keeps the existing shot-level affine;
- searches a local ±6 px residual translation using only a clean ring around the repair;
- excludes the logo interior from the score;
- applies local correction before the bounded tone fit;
- records local x/y offsets and residual ring MAD.

This experiment should be resumed if the project returns to masking.

## Open questions for later masking work

1. Does local residual alignment remove the ghost?
2. If not, is the remaining mismatch caused by:
   - different sharpening/ringing between DVD and HD encodes;
   - local geometric distortion not captured by affine+translation;
   - grain/noise mismatch;
   - chroma siting / colorspace differences;
   - learned detail reconstruction being needed inside the donor region?
3. Would a learned upscale of only the DVD repair donor improve local sharpness enough to make the patch invisible?
4. Would an AI-upscaled lower-left clean reference improve mask edge localization or local registration while remaining analysis-only?
5. Are some shots fundamentally unsuitable for direct DVD patching and better served by a different restoration method?

## Resume point

If masking is resumed, do not restart from logo detection or broad timing work.

Resume from the current shaped mask + deterministic cadence + per-shot donor phase architecture, and first evaluate the pending local-residual-registration hypothesis against the same three-minute visual sample.

Do not merge or treat the current three-minute masking prototype as production-ready until the repaired corner is visually invisible across representative motion and scene changes.
