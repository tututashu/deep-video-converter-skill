---
name: deep-video-converter
description: Deploy/operate a "deep video converter" style local web tool — the dual-venv Flask app that turns short videos into grayscale depth map, pose skeleton, face 478-landmark point cloud and combined overlays while keeping the original audio. Use when the user wants such a tool installed and running, or has such a project checked out locally (their own clone, fork, or existing directory) and wants to set it up, start the web app, run one of the five modes, verify it end-to-end, or debug environment failures (ffmpeg, pyexpat/libexpat on macOS Sequoia, model downloads, torch pinned for Intel Mac, missing face detections on ordinary video). Covers an environment preflight, obtaining the project (configurable default install source — only used when the user has no checkout; override with the user's own clone/fork), idempotent dual-venv + offline-model setup, launching the Flask server, the five modes, CLI verification, and reusing cached per-layer frames for fast re-compositing. This skill is a generic pattern, not bound to any repository as a primary project.
agent_created: true
---

# Deep Video Converter Tool — Deploy & Operate

Generic deployment/operation pattern for a **"deep video converter"** local web tool: an offline, browser-based app that processes short videos with CV models (depth / pose / face) and exports the result with the **original audio preserved**. This skill is **not bound to any repository as a primary project** — it operates on the user's own checkout (from any source) or, only if the user has no checkout, a **configurable default install source** defined below.

## Workflow Decision Tree

1. **User wants it running / installed** → [Preflight](#0-preflight-environment-check) → [Locate or obtain the project](#1-locate-or-obtain-the-project) → [One-shot setup](#2-one-shot-setup) → [Start the web app](#3-start-the-web-app).
2. **Server is running but something is wrong** (no models, blank page, 500 on process) → [Troubleshooting](#6-troubleshooting).
3. **User wants a CLI verification run** (no browser) → [Verify & debug without the browser](#5-verify--debug-without-the-browser).
4. **Models or venvs already exist** → skip straight to [Start the web app](#3-start-the-web-app) (setup is idempotent).

## 0. Preflight (environment check)

Run before setup. Check for the two system prerequisites the tool cannot install itself:

```bash
command -v python3.11 || echo "MISSING: python3.11 (face env)"
command -v python3.12 || command -v python3 || echo "MISSING: python3.12 or 3.10+ (main env)"
command -v ffmpeg || echo "MISSING: ffmpeg"
```

If anything is missing, **offer to auto-install**: present the user a clear choice and wait for their answer — do not silently install system packages, and do not skip ahead without consent.

- Option 1 — **auto-install (recommended if user is on their own machine)**: run the installer for the detected OS, then re-run the preflight checks. Auto-install may require elevation; if a command fails for lack of permissions, report it and ask the user to run it themselves or approve elevation.
  - Note: `setup.sh`/`setup.bat` also self-install prerequisites when run with `AUTO_INSTALL_PREREQS=1` (macOS/Homebrew and Windows/winget need no sudo; Linux needs passwordless sudo). Prefer the interactive choice above unless the user has explicitly authorized unattended silent install.
  - **macOS** (needs Homebrew): `brew install python@3.11 python@3.12 ffmpeg` (if `brew` is absent, report: install Homebrew first — `https://brew.sh`).
  - **Linux (Debian/Ubuntu)**: `sudo apt-get update && sudo apt-get install -y ffmpeg python3.12`; for the **face env Python 3.11** specifically use `sudo add-apt-repository ppa:deadsnakes/ppa && sudo apt-get install -y python3.11` (or pyenv if the user prefers). For other distros, fall back to the distro package manager or pyenv.
  - **Windows**: `winget install -e --id Python.Python.3.11` + `Python.Python.3.12` + `Gyan.FFmpeg` (if winget is unavailable, report and ask the user to install from python.org / ffmpeg.org, checking "Add to PATH").
- Option 2 — **user installs manually**: print the exact commands for their OS (above) and wait.
- Option 3 — **skip**: proceed anyway; model downloads or runtime will surface errors later.

After any install path, re-run the preflight checks and confirm all green before setup. If the machine cannot get system packages at all, stop and explain rather than failing mid-setup.

## 1. Locate or obtain the project

Priority order:

1. **User already has a checkout** (path they name, current directory, or an obvious `deep-video-converter/` folder nearby) → use it. Confirm it matches the structure contract below; run all commands from that root.
2. **User has no checkout but this skill is installed for them and they want a working tool** → use the **default install source** (only when the user has no code and does not name another source):
   ```bash
   git clone https://github.com/tututashu/deep-video-converter.git
   cd deep-video-converter
   ```
   > The default source is **configurable** — it is a fallback, not a binding. If the user names their own clone/fork URL or an existing directory, always prefer that. Maintainers who fork this skill can replace the URL above with their own repository to make it their "one-click install" source.
3. **User asks for a tool but refuses/excludes the default source** (e.g. wants only their private repo, or no source at all) → ask for their URL or path; do not install from an unapproved source.

Structure contract the target must satisfy:

- `backend/` — `app.py` (Flask server), `pipeline.py` (orchestrator: depth+pose+composite), `face_worker.py` (face subprocess), `config.py` (paths/tunables)
- `frontend/index.html` — single page (drag-drop + mode cards + progress)
- `scripts/` — `setup.sh`/`setup.bat` (idempotent env+model bootstrap), `run.sh`/`run.bat` (launch), `run_e2e_all.py`, `recomposite_test.py`
- `models/` (weights — typically NOT in git, downloaded by setup), `venvs/` (created by setup, ~2GB), `jobs/` (per-job intermediates)

## 2. One-shot setup

Requires: **Python 3.11** (face env) + Python 3.12 or 3.10+ (main env), and **ffmpeg** on PATH.

```bash
# Mac / Linux
bash scripts/setup.sh     # idempotent: skip existing venvs/weights

# Windows (x64 only; mediapipe has no ARM64 wheel)
scripts\setup.bat
```

What setup does (details in `references/project-notes.md`):
1. Creates two venvs — `env-main` (torch/transformers/opencv/flask) and `env-face` (mediapipe). On macOS it auto-repairs the `pyexpat`/`libexpat` link for Homebrew Python 3.11.
2. Installs deps from `backend/requirements_main.txt` + `requirements_face.txt`.
3. Downloads model artifacts into `models/` (hundreds of MB total). Sources may mix reachable mirrors and foreign hosts — see Troubleshooting for network-restricted machines.

Run setup to completion before starting. Do not interrupt mid-download (weights would be truncated; rerun setup to resume — it is idempotent and checks file existence).

## 3. Start the web app

```bash
# Mac / Linux
bash scripts/run.sh          # or: PORT=9000 bash scripts/run.sh
# Windows
scripts\run.bat
```

- Serves on `http://localhost:8000` (override with `PORT`), bound to `0.0.0.0` (LAN-reachable).
- `run.sh` hard-requires ffmpeg and an existing `venvs/env-main` — errors out with a clear message otherwise.
- Keep the launching shell alive — exiting it can kill the server. For unattended/long-running use, wrap it with `nohup`, `tmux`/`screen`, or a service manager.

## 4. Use the tool (five modes)

In the browser: drag-drop a short video (recommend ≤15s; processing caps the long side at 480px, see `MAX_DIM`) → pick one of five modes → start → watch progress → download result.

| ID | Effect | Tech |
|----|--------|------|
| `depth` | grayscale depth map (near bright / far dark) | Depth-Anything-V2 |
| `pose` | body pose skeleton | PyTorch KeypointRCNN COCO-17 |
| `combo` | depth as base + skeleton overlay | depth + pose |
| `face` | face 478-landmark point cloud | MediaPipe FaceLandmarker |
| `all` | everything combined | depth + pose + face |

Output is H.264 + AAC with the **original audio muxed back** (silent fallback if input has no audio track).

## 5. Verify & debug without the browser

- Full CLI end-to-end run (mode=all) on a bundled test clip:
  ```bash
  PYTHONPATH=backend venvs/env-main/bin/python scripts/run_e2e_all.py
  ```
- **Reuse already-computed per-layer frames** to re-render other modes without re-running inference (fast iteration after a face/pose/depth bugfix):
  ```bash
  PYTHONPATH=backend venvs/env-main/bin/python scripts/recomposite_test.py
  ```

## 6. Troubleshooting

| Symptom | Fix |
|---------|-----|
| `bash scripts/run.sh` → env-main not ready | Run `bash scripts/setup.sh` first |
| ffmpeg not found | Mac `brew install ffmpeg`; Windows: install from ffmpeg.org and add to PATH |
| `env-face` import fails on macOS Sequoia (`pyexpat`/`libexpat`) | Rerun `setup.sh` — it runs `brew install expat` + `install_name_tool` repoint automatically |
| Model download fails / offline machine | Re-run `setup.sh` — it auto-falls-back: official source → your own `MIRROR_MEDIAPIPE`/`MIRROR_PYTORCH` env hooks → **GitHub Releases 分卷** (`models_face.zip` / `models_depth.zip` / `models_keypointrcnn.zip`, base overridable via `RELEASE_BASE`). HuggingFace parts use `HF_ENDPOINT` (default `hf-mirror.com`). Pre-seeding `models/` from another machine always works too; the runtime sets `HF_HUB_OFFLINE=1` so it never phones home once set up |
| Face point cloud empty on normal (non-selfie) video | Expected with a bare FaceLandmarker — this pattern uses two-stage detection (BlazeFace full-frame → crop → FaceLandmarker). Check the BlazeFace detector model file exists |
| torch install fails on Intel Mac | Pin `torch==2.2.2` + `torchvision==0.17.2` (last macOS x86_64 wheels); never bump to ≥2.4 (arm64-only) |
| Slow CPU processing | Normal on Intel Mac; Apple Silicon auto-uses MPS (`PYTORCH_ENABLE_MPS_FALLBACK=1`); lower `MAX_DIM` or fps for long videos |

## Architecture in one paragraph

MediaPipe and torch must never share one process (Apple Silicon MPS/Metal conflict — and safer on Intel too). The orchestrator (env-main) spawns an **env-face subprocess** for face work **first**, then does depth+pose with torch, composites per-frame overlays, and ffmpeg-muxes the original audio. Per-stage frames are cached under `jobs/<id>/…` so re-compositing doesn't rerun inference. Full details in `references/project-notes.md`.

## Resources

- `references/project-notes.md` — architecture constraints, model paths, tunable parameters, and gotchas worth reading before modifying code or debugging inference.
