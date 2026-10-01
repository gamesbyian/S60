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
