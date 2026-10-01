# Episode 1 source calibration

## Confirmed source characteristics

Episode 1 was downloaded directly from the Seedhost FTP source and probed with FFmpeg.

### DVD reference
- Path: `/downloads/S60/DVD/Studio 60 - 01 - Pilot.avi`
- File size: 736,067,452 bytes
- Video: MPEG-4 Part 2 / Xvid
- Frame size: 848×476
- Pixel aspect ratio: 1:1
- Display aspect ratio: 212:119 (~1.7815:1)
- Frame rate: 30000/1001 (~29.970 fps)
- Reported video duration: 2746.810733 s
- Reported frame count: 82,322
- Audio: AC-3, 48 kHz, 5.1(side), 448 kb/s
- Repeated cropdetect samples: `848:464:0:6`

### 1080p reference
- Path: `/downloads/S60/1080p/Studio 60 S01E01.mp4`
- File size: 1,425,858,280 bytes
- Video: H.264 High
- Frame size: 1920×1080
- Pixel aspect ratio: 1:1
- Display aspect ratio: 16:9
- Field order: progressive
- Frame rate: 24000/1001 (~23.976 fps)
- Reported video duration: 2737.317917 s
- Reported frame count: 65,630
- Audio: AC-3, 48 kHz, 5.1(side)
- Sampled cropdetect: usually `1904:1072:10:4`, once `1920:1072:0:4`

## Immediate implications

The sources are not directly frame-index compatible. The DVD runs at 29.97 fps while the 1080p source runs at 23.976 fps, and the DVD episode is about 9.493 seconds longer overall. A production restoration pipeline therefore needs an explicit source-to-source time mapping before pixels from the DVD can be used to reconstruct the network-bug region.

The duration difference must not yet be interpreted as a uniform speed difference. It could arise from head/tail material, inserted/removed material, commercial-edit differences, or a combination. The next calibration step is local audio alignment at several anchors through the episode, plus a short cadence probe of the DVD to determine whether its 29.97 stream is a straightforward telecine/duplicate-frame representation of 23.976 material.

Cropdetect also indicates small edge differences between the encodes. Registration should therefore be solved from picture content rather than assuming a simple whole-frame scale from 848×476 to 1920×1080.

## Restoration principle

Treat 1080p as the master image everywhere it is unobscured. The DVD is a clean reference for reconstruction of the network-bug region, after temporal and spatial registration. Avoid replacing larger 1080p areas with DVD-derived pixels than necessary.


## Local audio alignment findings

A 1 kHz mono audio-envelope cross-correlation was run at five anchors, using the 1080p source as the time reference.

| 1080p anchor | Best DVD - 1080p offset | Correlation |
| ---: | ---: | ---: |
| 300 s | +1.930 s | 0.674 |
| 900 s | -0.440 s | 0.915 |
| 1500 s | -0.600 s | 0.964 |
| 2100 s | +0.590 s | 0.860 |
| 2550 s | +0.590 s | 1.006 |

The offset changes sign and later jumps again. This is incompatible with one fixed global offset and does not look like smooth clock drift. The working model is a small number of discrete edit/timeline discontinuities. Production registration should therefore use a piecewise time map.

## DVD cadence finding

A 12-second DVD sample at 300 s was decoded at native cadence and adjacent-frame mean absolute differences were grouped by transition index modulo 5. One modulo class had a mean MAD of only 0.130, while the other four classes were roughly 8.2–8.9. The lowest-difference transitions overwhelmingly occurred every fifth frame.

This is strong evidence that the DVD's 29.97 fps stream represents 23.976-origin material by repeating one frame in each five-frame cycle. A deterministic duplicate-frame decimation can therefore recover a 23.976-like clean reference before spatial registration. The exact phase should be verified per episode and around edit discontinuities rather than assumed globally.

## Next calibration step

Run a denser audio-offset map across episode 1 to locate the discontinuity boundaries. Once the piecewise time map is known, compare registered picture samples to solve scale/crop/translation and then characterize the network-bug footprint itself.


## Dense timing map

A denser 60-second audio-envelope correlation pass confirms stable timing plateaus rather than continuous drift:

| 1080p anchor range | DVD - 1080p offset |
| --- | ---: |
| 120–540 s | +1.930 s |
| 600–1140 s | -0.440 s |
| 1200–1620 s | -0.600 s |
| ~1680 s | +0.540 s transitional point |
| 1740–1980 s | +1.170 s |
| 2040–2640 s | +0.590 s |

Correlation scores across these anchors were generally strong. The working model is therefore several discrete edit/timing changes. The next pass should localize each boundary more precisely, then solve spatial registration within each stable segment.


## Refined discontinuity regions

Five-second audio-correlation probes narrowed the timing changes:

- First transition: stable at +1.930 s through 555 s; by 565–570 s it has settled at about -0.440 s. The 560 s anchor is inside the discontinuity.
- Second transition: stable at -0.440 s through 1145 s; settled at -0.600 s by 1150 s.
- Third transition: stable at -0.600 s through 1660 s; 1665–1685 s is transitional; settled at +1.170 s by 1690 s.
- A later transition from +1.170 s to +0.590 s occurs around the 2,000 s region and should be narrowed further only if needed for production mapping.

These are discrete timeline edits rather than drift.

## Spatial registration result

A matched still in the stable ~900 s plateau was registered after applying the measured active-picture crops. The DVD crop was 848×464 and the 1080p crop was 1904×1072.

The initial resize needed different horizontal and vertical scale factors because the two encodes use slightly different active-picture geometry:

- Initial X scale: 2.245283
- Initial Y scale: 2.310345

After that coarse resize, robust feature matching and RANSAC estimated a small residual transform:

- Residual uniform scale: 1.0162645
- Rotation: -0.0320°
- Translation X: +2.666 px
- Translation Y: -4.390 px
- Feature matches considered: 425
- RANSAC inliers: 79
- Median absolute luma difference outside the masked bug zone: 4
- Mean absolute luma difference outside the masked bug zone: 12.616

This indicates that the DVD and 1080p pictures are geometrically very close after active-picture normalization. A lightweight affine registration is likely sufficient within stable timeline segments. The low median residual is promising for using DVD-derived content only inside the bug mask, while retaining native 1080p pixels everywhere else.


## Focused network-bug mask result

A six-sample persistent-difference pass, restricted to the lower-left candidate region and using a percentile threshold robust to zero-inflated residuals, isolated one dominant component.

- Active-picture component: x=182, y=902, w=190, h=68
- Conservative active-picture bounding box: x=170..384, y=890..982
- Conservative full-frame 1080p bounding box: x=180..394, y=894..986
- Full-frame box size: 214×92 pixels
- Fraction of a 1920×1080 frame: about 0.95%

This is small enough to support the preservation-first strategy: native 1080p pixels can remain untouched across more than 99% of each affected frame.

Registration quality varied by sample. The 1500 s sample produced an implausible affine transform with only 19 inliers and must be rejected by production quality gates. Other samples produced plausible transforms with substantially more inliers. Restoration code must therefore validate transform scale/rotation/translation/inlier support and fall back or re-estimate rather than applying every estimated transform blindly.

Patch-region residuals before restoration were much larger than many seam-ring residuals, supporting the interpretation that the isolated region contains a real persistent overlay rather than merely global encode mismatch.


## First restored-frame prototype

Four registered episode 1 samples were rendered as side-by-side comparisons:

1. original 1080p with network bug;
2. registered DVD donor;
3. hard rectangular replacement;
4. feathered rectangular replacement.

The visual review confirms that the registered DVD donor is potentially usable, but a rectangular patch is not an acceptable production strategy.

Findings:

- Hard rectangular replacement produces obvious rectangular seams and is rejected.
- Feathering the rectangle reduces seam severity but still modifies far more native 1080p picture than necessary.
- The 2200 s sample is already visually encouraging after feathering, showing that the donor can plausibly blend into the HD master when registration and local tone are favorable.
- The 900 s sample reveals a major failure mode: simple mean/std photometric matching over the surrounding ring can hit its clamp limits and create a visible tonal slab. Photometric matching must therefore be quality-gated and more robust.
- The restoration mask should follow the actual network-logo silhouette, including only a very small antialias/feather margin, rather than using the 214×92 conservative bounding box as the compositing mask.
- The conservative bounding box remains useful as a search/processing ROI, not as the area to replace.

Current direction: derive a persistent logo-shaped alpha mask inside the ROI, fit donor-to-HD tone using robust paired pixels outside the logo, reject pathological fits, and modify only masked pixels.
