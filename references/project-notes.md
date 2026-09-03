# Project Notes — deep-video-converter internals

Read before modifying code or debugging inference failures. Complements the project README; this file captures the non-obvious constraints learned from building and validating the tool.

## Directory layout

```
deep-video-converter/
├── backend/
│   ├── app.py            # Flask: /api/process /api/status /api/download
│   ├── pipeline.py       # depth+pose+composite+mux (torch only, NEVER imports mediapipe)
│   ├── face_worker.py    # face subprocess entry (mediapipe only, NEVER imports torch)
│   ├── config.py         # paths, model dirs, tunables, env_python() (Win-aware)
│   ├── requirements_main.txt   # torch 2.2.2 / torchvision 0.17.2 / transformers / opencv / flask
│   └── requirements_face.txt   # mediapipe 0.10.14
├── frontend/index.html   # single page: drag-drop + mode cards + progress
├── models/               # self-contained weights (NOT in git; setup.sh downloads)
│   ├── face_landmarker/face_landmarker.task
│   ├── face_detector/blaze_face_short_range.tflite
│   ├── depth_anything_v2/          (Depth-Anything-V2-Small-hf snapshot)
│   └── keypointrcnn/keypointrcnn_resnet50_fpn.pth   (237MB, must be the -9f466800.pth build)
├── venvs/env-main  venvs/env-face  (~2GB total, created by setup, NOT in git)
├── jobs/<job_id>/      # per-job: orig/depth/pose/face/out_frames/ + result mp4
├── tmp/  verification_outputs/     (local demo artifacts, NOT in git)
└── scripts/  setup.sh / setup.bat / run.sh / run.bat / run_e2e_all.py / recomposite_test.py
```

## Hard architectural constraints (do not violate)

1. **Process isolation is mandatory.** `pipeline.py` must never `import mediapipe`; `face_worker.py` must never `import torch`. On Apple Silicon, torch(MPS) + MediaPipe(Metal) in one address space crashes; on Intel/Windows it's still the safe design.
2. **Face runs before torch.** Orchestrator order: ① face (env-face subprocess) → ② depth+pose (torch) → ③ composite per mode → ④ ffmpeg encode + mux original audio: `ffmpeg -y -i silent_out.mp4 -i input -map 0:v:0 -map 1:a:0 -c copy -shortest` (ffprobe audio first; silent fallback).
3. **Inter-process handoff via files**, not shared memory: per-frame BGRA overlays + `meta.json`.
4. **Offline-safe**: `run.sh` exports `HF_HUB_OFFLINE=1 TRANSFORMERS_OFFLINE=1 PYTORCH_ENABLE_MPS_FALLBACK=1`; models load with `local_files_only=True`. Depth via `AutoImageProcessor` + `AutoModelForDepthEstimation`; resize model-native resolution back to frame size before compositing (differs from input size).
5. **KeypointRCNN must not phone home**: construct `keypointrcnn_resnet50_fpn(num_classes=2, weights=None, weights_backbone=None)` then `load_state_dict` from the local `.pth` (else torchvision fetches the ResNet50 backbone online).

## Two-stage face detection (why face recall is high on normal videos)

`FaceLandmarker`'s built-in detector only works at selfie distance — 0 faces on walking/mid-shot/side-face footage. The repo therefore:
1. Runs BlazeFace `blaze_face_short_range.tflite` on the full frame at low confidence (`FACE_DET_CONF=0.3`).
2. Crops each bbox with padding (`FACE_CROP_PAD=0.25`), runs FaceLandmarker on the enlarged crop.
3. Maps crop-normalized landmarks back to full-frame coords and draws a z-depth blue→red gradient overlay.

Small frames are upscaled (≥640px long side) before detection. Both stages are mediapipe-only, so they stay inside the isolated env-face worker.

## Tunable parameters (`backend/config.py`)

| Param | Default | Meaning |
|-------|---------|---------|
| `MAX_DIM` | 480 | processing resolution cap (long side); result video is exported at this resolution |
| `CONF_POSE` | 0.7 | KeypointRCNN keypoint confidence threshold |
| `FACE_NUM` | 1 | max faces tracked |
| `FACE_DET_CONF` | 0.3 | BlazeFace full-frame confidence (lower = better recall at distance) |
| `FACE_PRESENCE_CONF` | 0.5 | FaceLandmarker presence confidence on the crop |
| `FACE_CROP_PAD` | 0.25 | bbox padding ratio when cropping for stage 2 |

## Platform notes

- **Intel Mac**: pinned `torch==2.2.2`/`torchvision==0.17.2` are the last releases shipping macOS x86_64 wheels (≥2.4 is arm64-only). CPU inference ~tens of seconds for a few-second clip.
- **Apple Silicon**: auto MPS via `PYTORCH_ENABLE_MPS_FALLBACK=1`; face still runs in the isolated py3.11 env-face subprocess.
- **Windows**: supported on x64 only (mediapipe 0.10.14 has no ARM64 wheel). `config.env_python()` auto-adapts `Scripts\python.exe`. Requires Python 3.11 + 3.12 via the `py` launcher and ffmpeg on PATH.
- **macOS Sequoia + Homebrew Python 3.11** `pyexpat` crash: fixed inside `setup.sh` (`brew install expat` + `install_name_tool -change` repoint of the `.so`).
- **GitHub repo hygiene**: `models/` (334MB), `venvs/` (2.2GB), `jobs/`, `tmp/`, `verification_outputs/` are git-ignored; the 237MB KeypointRCNN `.pth` exceeds GitHub's 100MB file limit anyway. Demo videos under `tmp/` contain real people (face-demographics dataset) — never commit them to a public repo.
