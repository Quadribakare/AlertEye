# AlertEye

A real-time driver drowsiness detector that watches eye closure (EAR) and head pitch through a webcam, escalates through warning states, and speaks alerts out loud via OpenAI TTS. Built on MediaPipe's Face Landmarker and OpenCV.

> This is a local desktop script (webcam + on-screen HUD + spoken alerts) — there's no hosted web demo to link to; run it locally per the instructions below.

## How it works

- **Eye closure** — Eye Aspect Ratio (EAR) computed from 6 landmarks per eye (`EAR_THRESHOLD = 0.22`); a blink is a brief dip, drowsiness is a *sustained* dip.
- **Head pitch** — angle derived from nose/chin/brow landmark depth and vertical spacing (`HEAD_PITCH_THRESH = 15.0°`); a sustained tilt beyond this means the head is dropping.
- Both signals are tracked as **consecutive-frame counters** (`closed_frames`, `head_frames`), not single-frame flags, so one bad frame never triggers an alert — only sustained closure/drop does.
- Escalation is frame-count based: `WARNING_FRAMES = 20` frames of sustained closure/drop → a warning state; `CRITICAL_FRAMES = 40` → critical.
- Every state change speaks a pre-generated OpenAI TTS alert (`tts-1`, voice `nova`) through `pygame`, cached to disk on first run so playback has no API latency.
- A full session log is written per run: per-frame CSV, per-transition event CSV, a JSON summary, and a HUD screenshot on every state change.

## State machine

```mermaid
stateDiagram-v2
    [*] --> ALERT
    ALERT --> WARNING_EYE: closed_frames >= 20
    ALERT --> WARNING_HEAD: head_frames >= 20
    ALERT --> WARNING_COMBINED: closed_frames >= 20 AND head_frames >= 20
    WARNING_EYE --> CRITICAL: closed_frames >= 40
    WARNING_HEAD --> CRITICAL: head_frames >= 40
    WARNING_COMBINED --> CRITICAL: closed_frames >= 40 OR head_frames >= 40
    WARNING_EYE --> ALERT: eyes reopen
    WARNING_HEAD --> ALERT: head lifts
    WARNING_COMBINED --> ALERT: both recover
    CRITICAL --> ALERT: both recover
```

## Pipeline: webcam frame to alert

```mermaid
flowchart TD
    Cam["cv2 webcam frame"] --> RGB["BGR -> RGB"]
    RGB --> FL["MediaPipe FaceLandmarker\n(VIDEO mode, 478 landmarks)"]
    FL --> EAR["compute_ear()\nper eye, averaged"]
    FL --> Pitch["compute_head_pitch()\nnose/chin/brow"]
    EAR --> Counters["Consecutive-frame counters\nclosed_frames / head_frames"]
    Pitch --> Counters
    Counters --> State["State machine\nALERT -> WARNING_* -> CRITICAL"]
    State --> HUD["HUD overlay\nEAR, pitch, status, bar"]
    State --> TTS["OpenAIAlertEngine.speak()\ncached mp3 via pygame"]
    State --> Log["SessionLogger\nframe_log.csv, event_log.csv,\nsession_summary.json, screenshots/"]
```

## Installation

```bash
pip install opencv-python mediapipe numpy scipy openai pygame
```

Set your OpenAI key (used only for TTS alert generation) as an environment variable or paste it into `OPENAI_API_KEY` at the top of `drowsiness_detect_tts.py` — **do not commit a real key to this file**.

The MediaPipe face landmarker model (`face_landmarker.task`, ~30 MB) is downloaded automatically on first run.

## Usage

```bash
python drowsiness_detect_tts.py
```

On first run, all alert phrases are pre-generated as mp3s into `alert_audio_cache/` (one-time OpenAI TTS cost), then the webcam opens with the live HUD. Each run writes a new `session_<timestamp>/` folder with the full frame/event logs and screenshots described above.
