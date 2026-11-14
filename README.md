# Remotion — Sharded Render on GitHub Actions (Action-only repo)

This repository contains **only a GitHub Action** — the workflow file
`.github/workflows/render.yml` and this README. **The Remotion project
itself NEVER lives in this repo**: no source code, no audio, no
subtitles are committed here.

Instead, the project lives in **one shared Google Drive folder** that
acts as the pipeline's file store. At trigger time you type the
tarball's **filename**; every job then looks that filename up inside
the folder, downloads the tarball, extracts it, renders, and uploads
the result as a run artifact — the same pattern as the old Google
Colab runs (upload tarball → extract → render → download result),
except the runner pulls the tarball itself, because GitHub runners
cannot receive a direct upload mid-run.

The workflow renders the Remotion composition **PodcastComp**
(procedural 3D voxel podcast animation, 1080×1920, 30 fps,
1743 frames = frames 0–1742, ±58.1 s). The timeline is split into
**N chunks that render in parallel on N separate runners** — N is
the **shard_count** you choose when triggering (default **8**,
allowed 1–24; a small "plan" job computes the frame ranges at
trigger time, splitting the 1743 frames as evenly as possible). A
final job then stitches the chunks together, muxes the audio track
and burns in the subtitles.

With the default shard_count of 8, the split is:

| Chunk | Frames | Artifact |
|---|---|---|
| 1 | 0–217 | `chunk-0` |
| 2 | 218–435 | `chunk-218` |
| 3 | 436–653 | `chunk-436` |
| 4 | 654–871 | `chunk-654` |
| 5 | 872–1089 | `chunk-872` |
| 6 | 1090–1307 | `chunk-1090` |
| 7 | 1308–1525 | `chunk-1308` |
| 8 | 1526–1742 | `chunk-1526` |

Each shard renders with `--x264-preset=ultrafast` and
`--concurrency=$(nproc)`, so it adapts to however many cores the
runner has. The stitch job concatenates the chunks with a stream copy
(no re-encode), then produces:

- `podcast_master_perf.mp4` — video only, all 1743 frames
- `podcast_master_perf_audio.mp4` — with the project's `audio.mp3`
  muxed in and its `subtitle.srt` burned in (Roboto Slab Bold,
  MarginV 44 — the locked subtitle style)

The `final-video` artifact (a ZIP of both files) is ≈108 MB,
because it holds **two complete video encodes**. The file you watch
or publish is `podcast_master_perf_audio.mp4`; the video-only file
is a master you can keep if you ever want to redo the audio or
subtitles without re-rendering.

The stitch job **installs ffmpeg itself** (`apt-get install ffmpeg`)
as its first step: this runner image does **not** ship ffmpeg. An
earlier version of this workflow assumed ffmpeg was preinstalled —
its stitch job died in 6 seconds with `ffmpeg: command not found`
after all the render shards had already succeeded (their chunk
artifacts survived on the run page). Do not remove that install
step.

## One-time setup

1. **Create the repository** (public — see *Exposure notes* below for
   why, and what that means). Suggested name: `remotion-render-action`.
   Add exactly two files to it, keeping the folder structure:
   - `.github/workflows/render.yml`
   - `README.md` (this file)

2. **Prepare ONE Google Drive folder as the pipeline store.**
   - Create a folder in Google Drive (any name, e.g. `remotion-renders`).
     This folder is the pipeline's filing cabinet: project tarballs go
     in, and over time it accumulates every version you render.
   - Share the **folder** as **“Anyone with the link”** (role: Viewer
     is enough — the runner only ever reads). Every file you drop into
     the folder inherits this access, so you never share files one by
     one.
   - Drop the project tarball into the folder (e.g.
     `remotion_project.tgz` — the same tarball used for the Colab runs:
     the Remotion project's files at its root — `package.json`,
     `package-lock.json`, `src/`, `public/` with `audio.mp3` and the
     corrected `subtitle.srt`, `fonts/`, `scripts/`).
   - The folder's URL looks like
     `https://drive.google.com/drive/folders/FOLDER_ID` — the
     `FOLDER_ID` part is what the next step needs.

3. **Store the folder ID as a repository secret.**
   In the repo: **Settings → Secrets and variables → Actions →
   New repository secret**
   - Name: `DRIVE_FOLDER_ID`
   - Value: the `FOLDER_ID` from the folder URL in step 2.

   The value is **masked in all workflow logs** and **can never be
   viewed again** after saving (it can only be replaced). Note the
   secret identifies the *folder*, not any single file — it never
   needs updating when you add or replace tarballs.

## Each render

1. Go to the repo's **Actions** tab → **Render video (sharded)** →
   **Run workflow**.
2. Type the **filename** of the tarball to render, exactly as it
   appears in the Drive folder (default: `remotion_project.tgz`),
   optionally change **shard_count** (how many parallel shard jobs
   split the timeline; default **8** — see *Pushing performance*
   below), and optionally change **retention_days** (how many days
   the artifacts are kept; default **7**).
3. Wait for the shard jobs and the stitch job to finish. Each shard
   job and the stitch job resolves the filename inside the folder
   (reading the folder's public listing), downloads the tarball, and
   works on its own copy.
   If the filename is not in the folder, the run fails immediately
   with an error that **lists the filenames the folder actually
   contains** — check for typos against that list.
4. Download the result from the finished run's **Artifacts** section:
   **`final-video`** contains `podcast_master_perf.mp4` and
   `podcast_master_perf_audio.mp4`. (The `chunk-*` artifacts are
   intermediates; you normally don't need them.)

Note: hosted runners have **no GPU** — frames are drawn with software
GL. The speed comes from parallelism across machines (one runner per
shard), not from a GPU. Each shard also repeats a setup (tarball
download + `npm ci` + browser ensure) before its first frame; in the
measured runs those steps took **seconds each** (resolve + download
+ extract ≈3 s, `npm ci` ≈6 s, browser ensure ≈3 s), so with runner
startup the fixed cost is well under a minute per shard, paid in
parallel.

## Pushing performance

The only speed lever at trigger time is **shard_count**:

- **More shards = more machines working in parallel = shorter
  wall-clock time.** The render finishes when the *slowest* shard
  finishes, so splitting the timeline thinner directly shortens the
  longest piece.
- **Every shard pays the same fixed setup cost** — tarball download
  + `npm ci` + browser ensure (measured in seconds, plus runner
  startup) — before it draws its first frame. That cost is paid in
  parallel, so it does not add up across shards, but it sets a floor:
  once a shard's slice of frames takes about as long to draw as the
  setup itself, adding more shards stops helping.
- For this 1743-frame composition, **8 is the default** and
  **12–16 is the practical ceiling** before setup overhead dominates.
  The plan job accepts 1–24 and refuses anything else with a clear
  error; smaller values (e.g. 4, the split this workflow used before)
  work too.
- **The per-frame drawing itself cannot be accelerated further on
  this route**: standard GitHub-hosted runners have no GPU, so frames
  are drawn with software GL, and each frame costs what it costs.
  Larger or GPU-equipped runners exist, but they are a **paid**
  GitHub feature (larger runners on Team/Enterprise plans) and are
  not used here.

## Updating the project later

- Build a **new tarball** from the updated project and **drop it into
  the same Drive folder** — under a new filename, or replacing the old
  one. **Nothing in this repo changes**: no secret update, no workflow
  edit. On the next run you simply type the (new) filename.
- Use distinct filenames (e.g. `remotion_project_v2.tgz`) if you want
  to keep older versions renderable; keep exactly one file per name —
  if two files in the folder share a name, the workflow uses the
  first one it finds.
- The workflow file only needs editing if the render recipe itself
  changes (composition name, presets, the 1–24 shard limit). The
  frame ranges are computed at trigger time from **shard_count**, so
  changing the split never requires an edit.
- **Adapting this workflow to a different composition:** it is
  currently configured for the composition `PodcastComp` and a total
  of **1743 frames**. Two places hold those values: `TOTAL_FRAMES`
  in the plan job, and the render command in the shards job
  (`src/index.ts PodcastComp`). Edit both to reuse the pipeline for
  another project; everything else (Drive lookup, sharding, stitch,
  mux style) is project-agnostic.
- You can **revoke access at any time** in Drive: change the folder's
  sharing back to restricted, or delete the tarball — every future run
  will then fail its lookup/download until access is restored.

## Where outputs live (an honest boundary)

Results are **GitHub run artifacts**, kept for the retention period
you chose at trigger time (default 7 days) — download `final-video`
before it expires. The runner **cannot write results back into the
Drive folder**: uploading to Drive would require Google account
credentials, which this design deliberately does not give it. So the
folder archives the pipeline's **inputs** (tarballs). If you want
finished videos kept in Drive too, download them from the run and
drop them into the folder yourself.

## Exposure notes

- The repo is **public**, so the **workflow file is public** (it
  contains no secrets — the folder ID lives only in the masked
  secret) and the **run artifacts are downloadable by anyone signed
  in to GitHub** until they expire after the chosen retention period
  (default 7 days). Download the final video promptly; set a shorter
  retention if you prefer a smaller window.
- The **Drive folder is shared as “Anyone with the link”**: anyone
  who learns the folder link or a file link can view/download its
  contents. The folder ID is protected as a masked secret and never
  printed in logs, but treat the folder as *unlisted*, not *private* —
  don't store anything in it beyond this pipeline's files.
- The **project code and audio never sit in the repo** — they exist
  only inside the tarballs in the Drive folder and, during a run, on
  the throwaway runner, which is destroyed when the run ends.
- Why public at all: standard GitHub-hosted runners for **public**
  repos are **4 vCPU / 16 GB RAM** with free, unlimited minutes;
  **private** repos get **2 vCPU / 8 GB** and consume the account's
  metered minutes. This workflow is a speed measurement, so the
  public specs are the point.

## Measured results (2026-11-14, this composition)

| Route | Total | Notes |
|---|---|---|
| Colab, one T4 GPU machine (Vulkan/full Chrome) | **7m 34s** | Full-timeline render on a single machine |
| GitHub Actions, 4 shards (earlier workflow) | 15m 43s | All shards succeeded (slowest 15m 28s); the stitch job then **failed** — ffmpeg was not preinstalled on the runner — so no final video |
| GitHub Actions, 8 shards (this workflow) | **9m 22s** | Success end to end: plan 4 s, slowest shard (654–871) 8m 34s, stitch + mux 36 s (ffmpeg install included; the mux itself ≈5 s). Final file verified: h264 1080×1920 + AAC, 58.1 s |

Two honest conclusions from those numbers:

- Going from 4 to 8 shards cut the GitHub wall-clock from 15m 43s to
  9m 22s — but one Colab T4 (7m 34s) still beats the sharded route,
  because per-frame drawing here is software GL and costs ~10–20×
  more per frame than the GPU path.
- Per-frame cost varies a lot by scene region: in the 4-shard run
  the *last* shard was the slowest (≈2.1 s/frame), while in the
  8-shard run the tail shard (1526–1742) was the *fastest*
  (≈1.1 s/frame) and shard 654–871 set the total. Watch the slowest
  shard, whichever it turns out to be — not a fixed one.
