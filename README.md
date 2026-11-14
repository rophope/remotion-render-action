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
**4 chunks that render in parallel on 4 separate runners**, then a
final job stitches them together, muxes the audio track and burns in
the subtitles.

| Chunk | Frames | Artifact |
|---|---|---|
| 1 | 0–435 | `chunk-0` |
| 2 | 436–871 | `chunk-436` |
| 3 | 872–1307 | `chunk-872` |
| 4 | 1308–1742 | `chunk-1308` |

Each shard renders with `--x264-preset=ultrafast` and
`--concurrency=$(nproc)`, so it adapts to however many cores the
runner has. The stitch job concatenates the chunks with a stream copy
(no re-encode), then produces:

- `podcast_master_perf.mp4` — video only, all 1743 frames
- `podcast_master_perf_audio.mp4` — with the project's `audio.mp3`
  muxed in and its `subtitle.srt` burned in (Roboto Slab Bold,
  MarginV 44 — the locked subtitle style)

## One-time setup

1. **Create the repository** (public — see *Exposure notes* below for
   why, and what that means). Suggested name: `aloy-remotion-render`.
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
   appears in the Drive folder (default: `remotion_project.tgz`), and
   optionally change **retention_days** (how many days the artifacts
   are kept; default **7**).
3. Wait for the 4 shard jobs and the stitch job to finish. Each job
   resolves the filename inside the folder (reading the folder's
   public listing), downloads the tarball, and works on its own copy.
   If the filename is not in the folder, the run fails immediately
   with an error that **lists the filenames the folder actually
   contains** — check for typos against that list.
4. Download the result from the finished run's **Artifacts** section:
   **`final-video`** contains `podcast_master_perf.mp4` and
   `podcast_master_perf_audio.mp4`. (The 4 `chunk-*` artifacts are
   intermediates; you normally don't need them.)

Note: hosted runners have **no GPU** — frames are drawn with software
GL. The speed comes from 4-way parallelism across machines, not from a
GPU. Each run also repeats setup (tarball download + `npm ci` +
browser ensure, roughly 1–2 minutes), in parallel on every shard.

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
  changes (frame ranges, composition name, presets).
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

## Baseline to beat

Colab T4, single machine, GPU (Vulkan/full Chrome) path, full timeline
frames 0–1742: **7 minutes 34 seconds** (render only). The sharded
run's figure to compare is the slowest shard's render time plus the
stitch job, from the timestamps echoed in each job's log.
