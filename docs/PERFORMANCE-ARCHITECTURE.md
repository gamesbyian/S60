# Restoration Performance Architecture

## Purpose

This document defines the implementation architecture to replace the current prototype's PNG-heavy, single-runner restoration path after the three-minute validation has established correctness.

The goal is to reduce wall-clock time dramatically **without changing restoration pixels, temporal mapping, spatial registration, mask behavior, or final-quality policy**.

The optimization target is the execution model, not the image model.

## Current bottleneck

The current three-minute prototype is intentionally simple but inefficient:

1. download full DVD and 1080p episode sources;
2. extract lossless FFV1 clips;
3. decode both clips to thousands of full-resolution PNG files;
4. reread those PNGs repeatedly for:
   - low-resolution cut detection;
   - donor-phase search;
   - sparse affine registration;
   - full-resolution restoration;
5. write thousands of restored 1080p PNG files;
6. reread those PNGs into ffmpeg for review encoding.

This produces enormous avoidable disk I/O and serialization overhead.

For a 180-second 24000/1001 clip, the prototype handles roughly 4,300 master frames, donor frames, and restored PNGs. Full-episode scaling using this execution model is unacceptable.

## Architectural principle

Split restoration into two explicit phases:

### Phase A: analysis

A cheap, deterministic pass produces a compact machine-readable **shot plan**.

The analysis pass determines:

- episode time-map segment;
- deterministic inverse-cadence mapping;
- HD shot boundaries;
- per-shot DVD donor phase;
- sparse registration anchors;
- robust per-shot affine transform;
- transform confidence;
- mask/layout identifier;
- frames/shots that must be rejected or escalated;
- exact shard boundaries for rendering.

Analysis should use low-resolution frames wherever possible and request full-resolution frames only for sparse spatial-registration anchors.

### Phase B: rendering

Rendering consumes a frozen shot plan and original source files.

For every output frame it:

1. decodes the authoritative 1080p frame;
2. decodes/selects the mapped DVD donor frame;
3. applies the shot-plan affine transform to the donor;
4. composites only through the validated logo alpha mask;
5. emits the restored 1080p frame directly to an encoder.

Rendering must not redo timeline analysis, cut detection, phase search, or registration unless the shot plan explicitly marks a frame for escalation.

## No-PNG streaming path

The production renderer must not materialize ordinary frames as PNG files.

Preferred pipeline:

```
ffmpeg HD decoder ─┐
                   ├─> streaming restoration worker ─> rawvideo pipe ─> ffmpeg encoder
ffmpeg DVD decoder ┘
```

Implementation options, in priority order:

1. Python/OpenCV reads rawvideo frames from ffmpeg subprocess pipes and writes restored rawvideo to an encoder pipe.
2. PyAV may replace subprocess piping later if it improves robustness without changing decoded pixels.
3. Frame files remain permitted only for deliberately retained diagnostics or failure cases.

Requirements:

- preserve 1920x1080 master geometry;
- preserve 24000/1001 output cadence;
- use original HD frame outside the repair alpha;
- use the same interpolation and affine math validated by the prototype;
- no global colorspace operation beyond unavoidable decode/encode processing;
- no intermediate lossy encode.

## Shot-plan schema

Create a versioned JSON artifact, initially `shot-plan-v1`.

Minimum structure:

```json
{
  "schema_version": 1,
  "episode": 1,
  "master_fps": "24000/1001",
  "mask_id": "episode01-nbccom-v1",
  "time_segments": [
    {
      "hd_start_frame": 0,
      "hd_end_frame": 12345,
      "dvd_base_time_seconds": 1.93
    }
  ],
  "shots": [
    {
      "shot_id": 0,
      "hd_start_frame": 42000,
      "hd_end_frame": 42116,
      "dvd_frame_shift": 3,
      "probe_mad": 3.14,
      "affine": {
        "scale": 0.988,
        "rotation_deg": 0.12,
        "tx": 8.4,
        "ty": -3.7
      },
      "anchor_count": 5,
      "confidence": "accepted"
    }
  ]
}
```

Exact frame-domain representation should be preferred over floating-point seconds once the episode time map is finalized.

The schema must be deterministic and sufficient to reproduce a render without rerunning analysis.

## GitHub Actions execution model

### Job 1: analyze

One job per episode:

- download the source pair once;
- derive/validate deterministic inverse cadence;
- build the shot plan;
- emit a compact JSON artifact;
- fail if unsupported shots remain;
- emit analysis diagnostics.

This is mostly CPU-light after low-resolution streaming is implemented.

### Jobs 2..N: render shards

Use a matrix strategy over **shot-aligned shards**.

Initial target: 6 to 8 render shards per episode.

A shard receives:

- episode identifier;
- shard identifier;
- first/last complete shot;
- shot-plan artifact;
- source paths/credentials.

Rules:

- shard boundaries must occur at shot boundaries;
- no shard may split a shot;
- each shard restores frames directly from original sources;
- no shard performs lossy pre-encoding intended for final concatenation;
- shard output should be a lossless intermediate suitable for bit-exact decode concatenation, or a carefully designed independently decodable final-codec segment if later tests prove concat-safe.

### Job N+1: assemble

After all shards succeed:

- download shard artifacts;
- verify exact frame counts and contiguous frame ranges;
- concatenate in canonical order;
- mux/copy original audio and compatible ancillary streams;
- run automated QA;
- produce final review/master artifact.

## Source-transfer strategy

Do **not** blindly make every render shard download both complete episode files if that dominates runtime or FTP traffic.

Benchmark these alternatives before locking the implementation:

1. each shard downloads only the time ranges it needs if the FTP/server and container formats allow practical range extraction;
2. each shard downloads complete source files independently;
3. a source-staging job uploads the episode pair as a temporary GHA artifact and shards download from Actions artifact storage;
4. use a self-hosted or persistent runner only if hosted Actions transfer becomes the dominant bottleneck.

The first implementation may use full independent downloads for simplicity, but timing must be measured explicitly.

## Sharding policy

Prefer approximately equal predicted render cost, not merely equal duration.

Cost estimate per shot may include:

- frame count;
- whether shot-level transform is sufficient;
- donor resizing/warping cost;
- diagnostic requirements.

Do not over-shard initially. Six to eight shards is enough to expose concurrency benefits without creating excessive startup/download/artifact overhead.

## Lossless shard format

Benchmark at least:

- FFV1 in Matroska;
- lossless x264;
- rawvideo only for local pipes, not artifact storage.

Selection criteria:

- decoded pixels identical to renderer output;
- fast encode/decode;
- seek/concat behavior;
- artifact size;
- GHA upload/download time.

The final delivery codec remains a separate decision.

## Quality invariants

Performance work is accepted only if it preserves these invariants:

- same deterministic DVD inverse cadence;
- same piecewise episode time map;
- same per-shot donor phase;
- same affine transform values;
- same alpha mask;
- same donor interpolation method;
- same 1080p pixels outside the mask before final video encoding;
- no extra lossy generation;
- original audio copied when compatible;
- no frame drops, duplication, or reordering;
- exact output frame count and cadence.

A performance optimization that changes restoration pixels must be treated as a restoration-method change and revalidated separately.

## Validation strategy

Before full-episode use:

1. take the already validated three-minute test window;
2. render it once with the current PNG prototype;
3. render it with the streaming architecture;
4. decode both outputs to lossless frames before the final lossy encode;
5. assert pixel equality or explain every difference;
6. compare frame counts, shot-plan values, temporal metrics, and mask application;
7. measure wall-clock time and peak storage.

Then enable sharding and compare:

- 1 shard;
- 2 shards;
- 4 shards;
- 8 shards.

Record total runner-minutes separately from wall-clock time so faster concurrency is not mistaken for lower compute cost.

## Implementation order

1. Extract prototype logic into reusable scripts/modules instead of inline workflow Python.
2. Define and validate `shot-plan-v1`.
3. Implement streaming low-resolution analysis.
4. Implement sparse full-resolution anchor registration.
5. Implement single-shard streaming renderer.
6. Prove pixel parity against the three-minute PNG prototype.
7. Add shot-aligned shard planner.
8. Add GHA render matrix.
9. Add lossless shard assembly.
10. Add whole-episode QA.
11. Benchmark 1/2/4/8 shards and choose the default concurrency.
12. Only then run the first full Episode 1 restoration.

## Files to introduce

Suggested structure:

```
src/s60/
  cadence.py
  timeline.py
  shots.py
  registration.py
  mask.py
  analyze.py
  render.py
  shot_plan.py

scripts/
  analyze_episode.py
  render_shard.py
  assemble_episode.py

schemas/
  shot-plan-v1.schema.json

.github/workflows/
  episode-analysis.yml
  episode-render.yml
```

The implementation should avoid encoding project logic directly into GitHub Actions YAML. Workflows orchestrate; versioned scripts own restoration behavior.

## Definition of ready

Architecture implementation can begin as soon as the current three-minute correctness run finishes and its restored artifact is judged acceptable.

The first implementation PR should **not** attempt a whole episode. It should reproduce the same three-minute window through the streaming single-shard path, prove parity against the validated prototype, and report timing/storage improvements.

Only after that parity gate passes should concurrency be enabled.
