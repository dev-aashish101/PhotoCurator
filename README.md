<div align="center">
  <img src="https://img.shields.io/badge/PhotoCurator-Native%20macOS%20Photo%20Pipeline-0F172A?style=for-the-badge&logo=apple&logoColor=white" alt="PhotoCurator Logo" />

  <h1>PhotoCurator</h1>
  <h3>AI-assisted culling, RAW processing, scoring, and export for photographers</h3>

  <p>
    <img src="https://img.shields.io/badge/Swift-5.9%2B-F05138?style=flat&logo=swift&logoColor=white" alt="Swift">
    <img src="https://img.shields.io/badge/macOS-14%2B-000000?style=flat&logo=apple&logoColor=white" alt="macOS">
    <img src="https://img.shields.io/badge/UI-SwiftUI%20%2B%20AppKit-0A84FF?style=flat" alt="UI">
    <img src="https://img.shields.io/badge/Build-SPM-1F6FEB?style=flat" alt="SPM">
    <img src="https://img.shields.io/badge/RAW-darktable--cli-8A2BE2?style=flat" alt="darktable-cli">
    <img src="https://img.shields.io/badge/AI-CoreML%20%2B%20Vision-34C759?style=flat" alt="CoreML and Vision">
    <img src="https://img.shields.io/badge/DB-SQLite3-003B57?style=flat&logo=sqlite&logoColor=white" alt="SQLite">
  </p>
</div>

---

PhotoCurator is a production-focused desktop workflow tool for high-volume image sets. It scans source folders, culls duplicates/weak frames, applies style-driven RAW development through `darktable-cli`, computes image quality scores, and exports delivery-ready files with metadata sidecars.

## Tech Stack

| Layer | Technology |
|---|---|
| App/UI | Swift, SwiftUI, AppKit |
| Build/Packaging | Swift Package Manager, macOS app bundle, DMG |
| Image Metadata/Decode | ImageIO, CoreGraphics |
| Culling/Sharpness | Accelerate/vImage (Laplacian variance) |
| Scene Understanding | Apple Vision (`VNClassifyImageRequest`, face detection) |
| Aesthetic Scoring | CoreML (NIMA models with fallback heuristic) |
| RAW Processing | `darktable-cli` via Swift `Process` wrapper |
| Persistence | SQLite3 C API in an actor-isolated `Database` layer |
| Concurrency | Swift structured concurrency (`async/await`, `TaskGroup`) |

## What This Project Delivers

- End-to-end local processing (no cloud dependency)
- Source-safe workflow (source images are treated as read-only)
- Multi-stage pipeline with toggles: Culling, Processing, Scoring, Export
- Scene-aware style routing into darktable styles
- Star-based export buckets for social media delivery
- Per-photo JSON sidecars for traceable metadata and scoring
- Live activity logs and inspector-driven debugging inside the app

## Workflow At A Glance

1. Select source folder.
2. Select destination folder.
3. Select photos (or all).
4. Run pipeline stages.
5. Review grouped/ranked results.
6. Export curated output folders.

### UI Preview

![PhotoCurator UI](assets/Screenshot%202026-03-14%20at%204.49.54%E2%80%AFPM.png)

## Pipeline Stages

| Stage | Purpose | Primary outputs |
|---|---|---|
| Library Scan | Reads supported files, extracts EXIF, writes/updates DB records | `Photo` records, folder groups |
| Culling | Blur scoring, duplicate grouping, rank assignment, rejection flags | `blurScore`, `blurStatus`, `duplicateGroupID`, `duplicateRank` |
| Scene Analysis | Vision labels + face-aware tags + style mapping | `sceneTags`, `dtStyleApplied` |
| Processing | Calls `darktable-cli` for eligible photos with selected style config | `_processed/.../*.jpg`, processing logs |
| Scoring | Computes aesthetic + technical quality, final score, stars | `nimaAesthetic`, `nimaTechnical`, `finalScore`, `starRating` |
| Export | Copies star-rated files to destination categories and writes JSON sidecars | `★★★ Instagram Ready`, `★★ Good`, `★ Keep` folders |

## Style Routing (Current)

Default style applied when no tag override matches:

- `BaseAuto1.1`

Overrides by scene tags:

- Sunset/sunrise tags (`sunset_sunrise`, `sunset`, `sunrise`) -> `BaseAutoSunsetBliss1.2`
- Nature/green tags (`forrest`, `forest`, `tree`, `trees`, `leaf`, `leaves`, `green_mountain`, `green mountain`, `mountain`) -> `BaseAutoAutumnal1.0`
- Urban tags (`city`, `street`, `urban`, `street photography`) -> `BaseAutoFuturist1.0`

Darktable processing is executed in single-style overwrite mode for each photo.

## Supported Formats

RAW:

- `cr2`, `cr3`, `nef`, `nrw`, `arw`, `raw`, `rw2`, `orf`, `dng`, `raf`, `3fr`, `mef`, `pef`, `srw`

Standard:

- `jpg`, `jpeg`, `png`, `tif`, `tiff`, `heic`, `heif`, `webp`


## Requirements

- macOS 14+
- Xcode command line tools / Swift 5.9+
- darktable installed (`darktable-cli` resolvable from known paths or `DARKTABLE_CLI_PATH`)
- Optional: `DARKTABLE_CONFIGDIR` for custom darktable config directory

## Export Output Layout

Processing workspace:

- `<Destination>/_processed/<folderDate|undated>/<filename>.jpg`

Delivery folders:

- `<Destination>/★★★ Instagram Ready/`
- `<Destination>/★★ Good/`
- `<Destination>/★ Keep/`

Each exported image includes a JSON sidecar with style, culling, score, and metadata fields.

## Runtime Data and Safety

- Pipeline DB path: `~/Library/Application Support/PhotoCurator/pipeline.db`
- Stage logs are stored in `log_entries` and shown in the in-app Activity Log.
- Stop behavior is cooperative between photos.
- Source folder contents are never overwritten by pipeline outputs.

## Troubleshooting

`darktable-cli` not found:

- Install darktable or set `DARKTABLE_CLI_PATH`.

Processing looks unexpected:

- Verify darktable style names exist and match exactly.
- Check darktable local config and sidecar behavior for consistency across GUI vs CLI runs.

Culling decode errors on a specific file:

- The file may be unsupported/corrupt for the decode path used in blur detection.

---

For architecture and implementation contracts, read:

1. `AGENTS.md`
2. `Sources/PhotoCurator/App/AppState.swift`
3. `Sources/PhotoCurator/Pipeline/PipelineOrchestrator.swift`
