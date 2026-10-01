# S60 restoration workspace

This repository supports restoration of *Studio 60 on the Sunset Strip* from two complementary source sets stored on the Seedhost FTP server.

## Canonical source roles

- `/downloads/S60/DVD` — pristine presentation source. These files do **not** contain the network bug/logo and are the reference for clean picture content.
- `/downloads/S60/1080p` — highest-resolution source. These files contain the network bug/logo and are the reference for high-resolution detail.
- `/downloads/S60/720p` — secondary source, retained as an additional comparison/reference tier.

The DVD and 1080p filenames are not expected to match literally. Pair episodes by the episode number encoded in each filename. Pairing code must fail/report ambiguity rather than silently choosing a file when an episode number is missing, duplicated, or conflicting.

## Restoration objective

Use the clean DVD source to identify/reconstruct the region obscured by the network bug in the 1080p source while preserving as much native 1080p information as possible. Final masters should remain suitable for a separate later 4K upscale.
