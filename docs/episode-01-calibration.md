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
