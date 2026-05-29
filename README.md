# Surveillance Unified

Single-page Flask app that turns an RTSP camera (or any uploaded video) into a small security-camera platform: live person detection, autonomous threat-triggered recording, a recordings library with AI threat scoring, and natural-language search.

YOLO does the person/motion gating. A llama.cpp OpenAI-compatible endpoint (Qwen-style vision model) handles the higher-level threat assessment over temporal mosaics of person frames.

## What it does

- **Live**: connect to an RTSP camera, stream MJPEG into the browser, run YOLO person detection on sampled frames.
- **Arm**: when armed, person dwell on the live feed triggers autonomous recording; clips are saved and queued for AI analysis. UI shifts to a red lockdown theme.
- **Arm schedule**: time-of-day window for auto-arm (e.g. 19:00–06:00).
- **Recordings library**: auto-captured + manual clips with first-frame thumbnails, custom names, star/important flags, threat scores (0–100), threat assessments, and natural-language AI search ("people tampering with the water heater"). Click a card to expand the player and full analysis; rerun analysis on demand.
- **Alerts**: collapsible event log of autonomous triggers and threat hits.
- **Analyze**: upload a single video for the same person-detection + temporal-mosaic + threat-assessment pipeline.

## How analysis works

1. YOLO (`yolo26s.pt` / `yolo26s.onnx`) filters frames containing people, biased toward high-motion samples.
2. Frames are tiled into 4×4 temporal mosaics spanning the clip's timeline (up to ~24 mosaics).
3. Each mosaic is captioned by the llama.cpp endpoint; a final pass scores the clip (0–100) and produces a written threat assessment.
4. Results are written to a `.json` sidecar next to the recording.

## Prerequisites

- Python 3.12+
- [`uv`](https://docs.astral.sh/uv/)
- `ffmpeg` + `ffprobe` on `PATH`
- CUDA GPU (default; set `REQUIRE_GPU=0` to allow CPU fallback)
- A running llama.cpp-compatible server exposing `POST /v1/chat/completions`
- An RTSP camera for the live features (optional — the Analyze upload flow works without one)

## Install & run

```bash
uv sync
./run.sh
```

Default URL: **http://127.0.0.1:33263**

Equivalent:

```bash
uv run flask --app main:app run --host 0.0.0.0 --port 33263
```

## Environment variables

**LLM**
- `LLAMACPP_BASE_URL` (default `http://127.0.0.1:8078`)
- `LLAMACPP_MODEL` (default `local-model`)

**Live / RTSP**
- `LIVE_RTSP_DEFAULT` — default camera URL
- `LIVE_RTSP_DISCOVER_USER`, `LIVE_RTSP_DISCOVER_PASSWORD` — creds for network discovery
- `LIVE_DISCOVERY_PROBE_TIMEOUT_S` (default `5`)
- `LIVE_RTSP_READ_FAILS_BEFORE_RECONNECT` (default `20`)

**Detection**
- `REQUIRE_GPU` (default `1`; `0` allows CPU fallback)
- `LIVE_DETECT_CONF` (default `0.5`)
- `LIVE_YOLO_EVERY_N_FRAMES` (default `8`)
- `LIVE_YOLO_IMGSZ` (default `640`)
- `YOLO_FRAME_WORKERS` (default `4`)

**Autonomous recording**
- `AUTONOMOUS_TRIGGER_HITS` (default `3`) — person-frame hits to start a clip
- `AUTONOMOUS_TRIGGER_MISSES` (default `12`) — empty frames before considering stop
- `AUTONOMOUS_DWELL_OFF_S` (default `10.0`) — dwell-buffer before actually stopping
- `AUTONOMOUS_MAX_CLIP_S` (default `180.0`) — hard cap on clip length

**Mosaic / analysis**
- `RECORDING_ANALYSIS_GRID_N` (default `4`) — mosaic grid edge
- `RECORDING_ANALYSIS_MOTION_BIAS` (default `0.85`)

Example (CPU fallback + custom LLM endpoint):

```bash
REQUIRE_GPU=0 LLAMACPP_BASE_URL=http://10.0.0.5:8080 ./run.sh
```

## Project layout

```
surveillance-unified/
├── main.py                # Flask app: live, recordings, analysis, autonomous workers
├── templates/index.html   # Single-page UI (Home / Live / Recordings / Analyze)
├── run.sh
├── pyproject.toml
├── person-detect/         # YOLO weights (yolo26s.pt / .onnx) + standalone demo
├── video-understanding/   # Earlier standalone experiments (reference)
└── runtime/               # Created on first run
    ├── uploads/           # User uploads
    ├── outputs/           # Source videos served for playback
    ├── recordings/        # Auto + manual clips with .json sidecars (score, flags, name)
    ├── cache/             # YOLO-filtered person-only clips + manifests
    └── events.json        # Rolling alert log (200 events)
```

## Key endpoints

- `GET /` — UI
- `GET /live/feed` — MJPEG stream
- `POST /live/start`, `POST /live/stop`, `GET /live/status`
- `POST /live/discover` — RTSP network probe
- `GET /recordings/list`, `POST /recordings/meta`, `POST /recordings/delete`
- `POST /recordings/analyze`, `POST /recordings/search`
- `GET /recordings/thumb/<id>`
- `POST /autonomous/start`, `POST /autonomous/stop`, `GET /autonomous/status`
- `POST /analyze/start`, `GET /analyze/status/<job_id>`, `GET /analyze/result/<job_id>`
- `GET /events`
- `GET /files/<path>` — serves anything under `runtime/`

## Notes

- Supported video types: `.mp4`, `.mov`, `.avi`, `.mkv`, `.webm`.
- Person-class filtering is automatic from the YOLO label set.
- Manually-armed state is preserved across schedule transitions — the schedule won't auto-disarm what you turned on by hand.

## Related

- `person-detect/README.md` — standalone person-detection demo.
