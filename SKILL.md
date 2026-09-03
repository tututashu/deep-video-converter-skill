---
name: deep-video-converter
description: Deploy/operate the Deep Video Converter — a self-contained offline web tool that turns a short video into depth map, pose skeleton, face 478-landmark point cloud, and combined effects while keeping the original audio. Use when the user wants to install, set up, start, use, verify, or troubleshoot this tool, or asks to "convert video to depth/pose/face overlays", "run the local video CV tool", "start the video effects web app", "run setup.sh for deep-video-converter", or reports environment issues (ffmpeg, pyexpat/libexpat, model downloads, torch on Intel Mac, MediaPipe face detection missing on normal videos). Covers locating or cloning the repo, idempotent setup of dual Python venvs + offline models, launching the Flask web app, the five modes, e2e verification and frame reuse for fast re-compositing.
agent_created: true
---

# Deep Video Converter — Deploy & Operate

Deploy, run, verify, and troubleshoot the **Deep Video Converter**: an offline, local, browser-based tool that processes short videos with CV models (depth / pose / face) and exports the result with the **original audio preserved**.

## Workflow Decision Tree

1. **User wants it running / installed** → follow [Locate or clone the repo](#1-locate-or-clone-the-repo) → [One-shot setup](#2-one-shot-setup) → [Start the web app](#3-start-the-web-app).
2. **Server is running but something is wrong** (no models, blank page, 500 on process) → [Troubleshooting](#6-troubleshooting).
3. **User wants a CLI verification run** (no browser) → [Verify & debug without the browser](#5-verify--debug-without-the-browser).
4. **Models or venvs already exist** → skip straight to [Start the web app](#3-start-the-web-app) (setup is idempotent).

## 1. Locate or clone the repo

- Prefer an existing local checkout if one is already on the machine; otherwise clone the public repo:
  ```bash
  git clone https://github.com/tututashu/deep-video-converter.git
  cd deep-video-converter
  ```
- Repo layout: `backend/` (Flask app + pipeline), `frontend/index.html` (single page), `scripts/` (setup/run/e2e), `models/` (weights, NOT in git — downloaded by setup), `venvs/` (created by setup, ~2GB), `jobs/` (per-job intermediates).

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
3. Downloads 4 model artifacts into `models/` (~450MB total; torch weights 237MB via `download.pytorch.org`, depth model via `hf-mirror.com` default `HF_ENDPOINT`). **Network-restricted users must pre-seed `models/` or the download fails — see Troubleshooting.**

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

In the browser: drag-drop a short video (recommend ≤15s; processing caps the long side at 480px, see `MAX_DIM`) → pick one of five modes → 开始转换 → watch progress → download result.

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
| Model download fails / offline machine | Pre-seed `models/` from another machine (same layout), or set `HF_ENDPOINT=https://hf-mirror.com` and retry; `run.sh` sets `HF_HUB_OFFLINE=1` so runtime never phones home |
| Face point cloud empty on normal (non-selfie) video | Expected with a bare FaceLandmarker — this repo uses two-stage detection (BlazeFace full-frame → crop → FaceLandmarker). Check `models/face_detector/blaze_face_short_range.tflite` exists |
| torch install fails on Intel Mac | Repo pins `torch==2.2.2` + `torchvision==0.17.2` (last macOS x86_64 wheels); never bump to ≥2.4 (arm64-only) |
| Slow CPU processing | Normal on Intel Mac; Apple Silicon auto-uses MPS (`PYTORCH_ENABLE_MPS_FALLBACK=1`); lower `MAX_DIM` or fps for long videos |

## Architecture in one paragraph

MediaPipe and torch must never share one process (Apple Silicon MPS/Metal conflict — and safer on Intel too). The orchestrator (env-main) spawns an **env-face subprocess** for face work **first**, then does depth+pose with torch, composites per-frame overlays, and ffmpeg-muxes the original audio. Per-stage frames are cached under `jobs/<id>/…` so re-compositing doesn't rerun inference. Full details in `references/project-notes.md`.

## Resources

- `references/project-notes.md` — architecture constraints, model paths, tunable parameters, and gotchas worth reading before modifying code or debugging inference.
