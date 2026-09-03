---
name: deep-video-converter
description: Deploy/operate an existing "deep video converter" style local web tool — the dual-venv Flask app that turns short videos into grayscale depth map, pose skeleton, face 478-landmark point cloud and combined overlays while keeping the original audio. Use when the user has such a project checked out locally (from any source: their own clone, fork, or an existing directory) and wants to install/set it up, start the web app, run one of the five modes, verify it end-to-end, or debug environment failures (ffmpeg, pyexpat/libexpat on macOS Sequoia, model downloads, torch pinned for Intel Mac, missing face detections on ordinary video). Covers locating the project by its structure, idempotent dual-venv + offline-model setup, launching the Flask server, the five modes, CLI verification, and reusing cached per-layer frames for fast re-compositing. This skill is a generic pattern and is not bound to any specific source repository — the target project is provided by the user.
agent_created: true
---

# Deep Video Converter Tool — Deploy & Operate

Generic deployment/operation pattern for a **"deep video converter"** local web tool: an offline, browser-based app that processes short videos with CV models (depth / pose / face) and exports the result with the **original audio preserved**. This skill is **not bound to any specific repository** — operate on the user's own checkout (from any source).

## Workflow Decision Tree

1. **User wants it running / installed** → [Locate the project](#1-locate-the-project) → [One-shot setup](#2-one-shot-setup) → [Start the web app](#3-start-the-web-app).
2. **Server is running but something is wrong** (no models, blank page, 500 on process) → [Troubleshooting](#6-troubleshooting).
3. **User wants a CLI verification run** (no browser) → [Verify & debug without the browser](#5-verify--debug-without-the-browser).
4. **Models or venvs already exist** → skip straight to [Start the web app](#3-start-the-web-app) (setup is idempotent).

## 1. Locate the project

- Ask the user for the project path, or confirm a candidate directory. The target must contain this repo layout (the contract this skill operates against):
  - `backend/` — `app.py` (Flask server), `pipeline.py` (orchestrator: depth+pose+composite), `face_worker.py` (face subprocess), `config.py` (paths/tunables)
  - `frontend/index.html` — single page (drag-drop + mode cards + progress)
  - `scripts/` — `setup.sh`/`setup.bat` (idempotent env+model bootstrap), `run.sh`/`run.bat` (launch), `run_e2e_all.py`, `recomposite_test.py`
  - `models/` (weights — typically NOT in git, downloaded by setup), `venvs/` (created by setup, ~2GB), `jobs/` (per-job intermediates)
- **If the user asks to install a deep video converter but does not have the code**, this skill does not bundle a source repo — point that out and ask them to provide their checkout (clone/fork URL or existing directory). Do not invent a download source.
- Run all commands from the project root.

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
| Model download fails / offline machine | Pre-seed `models/` from another machine (same layout), or set the download endpoint mirror (`HF_ENDPOINT` for HuggingFace parts) and retry; the runtime typically sets `HF_HUB_OFFLINE=1` so it never phones home once set up |
| Face point cloud empty on normal (non-selfie) video | Expected with a bare FaceLandmarker — this pattern uses two-stage detection (BlazeFace full-frame → crop → FaceLandmarker). Check the BlazeFace detector model file exists |
| torch install fails on Intel Mac | Pin `torch==2.2.2` + `torchvision==0.17.2` (last macOS x86_64 wheels); never bump to ≥2.4 (arm64-only) |
| Slow CPU processing | Normal on Intel Mac; Apple Silicon auto-uses MPS (`PYTORCH_ENABLE_MPS_FALLBACK=1`); lower `MAX_DIM` or fps for long videos |

## Architecture in one paragraph

MediaPipe and torch must never share one process (Apple Silicon MPS/Metal conflict — and safer on Intel too). The orchestrator (env-main) spawns an **env-face subprocess** for face work **first**, then does depth+pose with torch, composites per-frame overlays, and ffmpeg-muxes the original audio. Per-stage frames are cached under `jobs/<id>/…` so re-compositing doesn't rerun inference. Full details in `references/project-notes.md`.

## Resources

- `references/project-notes.md` — architecture constraints, model paths, tunable parameters, and gotchas worth reading before modifying code or debugging inference.
