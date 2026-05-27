---
name: video-podcast-clipper
description: >
  AI-powered short-form clip extractor for videos, podcasts, webinars, interviews,
  and all long-form content. Transcribes with faster-whisper, scores segments with
  a neuroscience-informed 8-dimension rubric, tracks faces with MediaPipe for smart
  crop positioning, renders vertical 1080x1920 clips with ffmpeg, generates animated
  captions with Remotion, and exports platform-optimized files.
  Works with any AI coding agent (Claude Code, Hermes, OpenClaw, Codex, etc.).
  Use when the user says "clip this", "extract shorts", "make clips", "short-form",
  "vertical clips", "reels", "shorts", "tiktok", or "repurpose".
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - AskUserQuestion
---

# Video Podcast Clipper — AI Short-Form Clip Extractor for Any Long-Form Content

Turn long-form video or audio into viral-ready vertical short clips.
Designed to run on any AI coding agent with no proprietary dependencies.

## Quick Start (Minimal Path)

If you just want to get clips out fast:

```bash
# 1. Install dependencies
pip install faster-whisper mediapipe opencv-python-headless 2>/dev/null
# ffmpeg must be available: ffmpeg -version
# Node.js must be available for Remotion captions: node --version

# 2. Put your source video in the working directory as source.mp4

# 3. Tell the agent:
#    "Clip this video: source.mp4"
```

The agent will transcribe, find the best moments, track faces for smart crop positioning, render vertical clips with animated captions, and place them in `./exports/`.

\---

## Full Pipeline (12 Steps)

### Step 1: Preflight

Validate the input and environment before doing any work.

```bash
# Check ffmpeg
ffmpeg -version 2>\&1 | head -1

# Check disk space (need at least 2x the source file size free)
df -h .

# Validate the source file
ffprobe -v error -show\_entries stream=codec\_name,width,height,duration \\
  -of csv=p=0 source.mp4
```

Report to the user: source resolution, duration, codec, disk space, estimated
processing time. If the source is under 720p, warn that vertical upscaling
will be soft.

**Supported input formats:** MP4, MKV, MOV, AVI, WebM, MP3, WAV, M4A.
**Supported sources:** Local file, YouTube URL (via yt-dlp), direct video URL.

\---

### Step 2: Source Acquisition

If the user provides a local file, verify it with ffprobe and move on.

If the user provides a URL:

```bash
# YouTube
yt-dlp -f "bestvideo\[height<=1080]\[ext=mp4]+bestaudio\[ext=m4a]/best\[height<=1080]" \\
  -o "source.mp4" "<URL>"

# Direct video URL
curl -L -o source.mp4 "<URL>"
```

After download, verify the file is actually video (not an HTML error page):

```bash
file source.mp4
# Should show: ISO Media, MP4 v2, etc.
# If it shows "HTML document", the download failed
```

\---

### Step 3: Transcribe

Extract audio and transcribe with faster-whisper. Always use the Python
library (not the CLI — the CLI times out on CPU).

```python
# Extract audio
ffmpeg -y -i source.mp4 -vn -acodec pcm\_s16le -ar 16000 -ac 1 /tmp/audio.wav

# Transcribe (run inside the agent's Python execution environment)
from faster\_whisper import WhisperModel
model = WhisperModel("base", device="cpu", compute\_type="int8")
segments, info = model.transcribe("/tmp/audio.wav", language="en",
                                   word\_timestamps=True, vad\_filter=True)

# Collect results
words = \[]
for seg in segments:
    for w in seg.words:
        words.append({"word": w.word, "start": w.start, "end": w.end})

# Write transcript
import json
with open("transcript.json", "w") as f:
    json.dump({"words": words, "duration": info.duration}, f)
```

**Model selection:**

* `base` — good accuracy, \~145MB, fast on CPU (recommended default)
* `small` — better accuracy, \~244MB, moderate speed
* `large-v3` — best accuracy, \~1.6GB, slow on CPU (use only with GPU)

**For videos >30 minutes:** Split audio into 10-minute chunks and transcribe
in batches to avoid timeout. Combine the word lists with adjusted timestamps.

**VAD filter note:** `vad\_filter=True` removes silence and secondary speakers.
This is ideal for guest-focused clips. Set `vad\_filter=False` if you need
host questions captured in the transcript.

\---

### Step 4: Analyze — Score Segment Candidates

Read the full transcript and identify 6-12 candidate segments using the
**8-dimension Neuro-Engagement Scoring Rubric** (`references/scoring-rubric.md`).

**Scoring Formula:**

```
final_score = (visual_engagement   × 0.12)
            + (auditory_dynamics   × 0.12)
            + (linguistic_density  × 0.12)
            + (cross_modal_sync    × 0.10)
            + (hook_strength       × 0.15)
            + (emotional_arc       × 0.12)
            + (standalone_coherence × 0.15)
            + (payoff_resolution   × 0.12)
```

**Minimum threshold:** 65/100. **Target duration:** 15-55 seconds per clip.

The 8 dimensions:

| # | Dimension | Weight | What it measures |
|---|-----------|--------|------------------|
| 1 | Visual Engagement | 0.12 | Motion dynamics, facial expressiveness, scene changes |
| 2 | Auditory Dynamics | 0.12 | Vocal energy, prosodic variation, emotional voice qualities |
| 3 | Linguistic Density | 0.12 | Information richness, semantic novelty, conceptual depth |
| 4 | Cross-Modal Sync | 0.10 | Voice + face + words all amplifying the same moment ("goosebumps") |
| 5 | Hook Strength | 0.15 | First 3 seconds — bold claims, curiosity gaps, pattern interrupts |
| 6 | Emotional Arc | 0.12 | Does energy build, peak, and release? Brain tracks CHANGE, not absolute level |
| 7 | Standalone Coherence | 0.15 | Complete self-contained narrative arc. No external context needed |
| 8 | Payoff Resolution | 0.12 | Satisfying ending — punchline, twist, actionable takeaway |

Full scoring tables with levels and indicators are in `references/scoring-rubric.md`.
Use the **Quick-Score Reference Card** at the bottom for rapid scanning.

**Candidate selection rules:**
- Target 6-12 candidates from a typical 30-60 minute video
- Clips at least 90s apart in the source timeline (brain needs ~100s to reset between engagement peaks)
- Diversity: at least 2 clips each with strong Visual/Auditory/Linguistic scores (≥70)
- At least 1 "Build → Peak → Release" emotional arc
- Max 2 clips on same subtopic; max 2 clips with same hook archetype
- Align start/end with sentence boundaries, not mid-word
- For each candidate, record: start time, end time, duration, score on all 8 dimensions, hook text, and rationale

**Red flags (automatic low coherence score):**
- "As I mentioned earlier..." / "Going back to what we discussed..."
- Pronouns without clear referents ("he said that...")
- Cuts off mid-sentence at the end

**Clip selection policy (configurable):**
- Default: Guest-focused. Host only included for lead-in questions.
- Positive framing only: If a clip portrays the subject negatively
  (admitting failure, incompetence, products failing), drop it entirely.
- No dangling pronouns: If the first 3-5 words contain a pronoun or proper
  name without context, move the start forward to the next self-contained
  statement.

\---

### Step 5: Present — Show Candidates

Present candidates in a formatted table:

```
| # | Start | End | Dur | Hook | Engagement Score | Rationale |
|---|-------|-----|-----|------|-----------------|-----------|
| 1 | 04:22 → 05:01 | 39s | "Nobody talks about..." | 87 | Contrarian take with data |
```

Ask the user:

1. Which segments? (all, specific numbers, or "re-analyze")
2. Caption style? (none / clean / bold / bounce — see Step 9)
3. Platform? (youtube / tiktok / instagram / all)

Write approved selections to `clips.json`:

```json
{
  "source": "source.mp4",
  "clips": \[
    {
      "id": "01",
      "title": "Nobody talks about this",
      "start": 262.0,
      "end": 301.0,
      "duration": 39.0,
      "score": 87,
      "hook": "Nobody talks about this hidden cost"
    }
  ],
  "caption\_style": "none",
  "platform": "all"
}
```

\---

### Step 6: Snap Boundaries

Adjust cut points to natural audio boundaries so clips never cut mid-word.

**Rules:**

1. Snap start time to the beginning of the nearest word
2. Prefer sentence starts (after `.` `?` `!`) if within 1.5s
3. Extend end time to the next sentence boundary if within 3 seconds
4. Add 300ms padding after the last word
5. Enforce minimum 15s / maximum 60s duration

```bash
# Optional: detect silence points for cleaner cuts
ffmpeg -i source.mp4 -af "silencedetect=noise=-35dB:d=0.3" -f null - 2>\&1 | \\
  grep "silence\_start\\|silence\_end"
```

\---

### Step 7: Face Track — MediaPipe FaceLandmarker

Use MediaPipe's FaceLandmarker to automatically detect face position in each clip segment.

```bash
python3 scripts/face\_track.py \\
  --source source.mp4 \\
  --clips clips.json \\
  --output face\_data.json \\
  --fps 10
```

**What it does:**

* Extracts frames from each clip at 10 fps (good balance of speed vs. accuracy)
* Runs MediaPipe FaceLandmarker (468-point face mesh) on each frame
* Outputs per-frame face position (nose tip X/Y, face width, confidence)
* Detects split-screen layouts automatically

**Output:** `face\_data.json` with per-clip frame arrays:

```json
{
  "clips": {
    "01": {
      "frames": \[
        {"time": 0.0, "face\_x": 0.52, "face\_y": 0.33, "confidence": 0.98, "detected": true},
        ...
      ],
      "split\_screen": false,
      "dominant\_side": "center"
    }
  }
}
```

**Model:** The FaceLandmarker model (\~3MB) auto-downloads on first run to
`\~/.mediapipe/`. Requires `pip install mediapipe opencv-python-headless`.

**Performance:** \~2-5 seconds per clip on CPU (depending on duration and resolution).

\---

### Step 8: Smart Crop Compute (NEW)

Convert face tracking data into smooth crop keyframes for ffmpeg.

```bash
python3 scripts/smart\_crop.py \\
  --face-data face\_data.json \\
  --clips clips.json \\
  --output crop\_data.json \\
  --source source.mp4 \\
  --padding 0.15 \\
  --smooth 5
```

**What it does:**

* Reads face position data from Step 7
* Applies moving-average smoothing (default: 5-frame window) to eliminate jitter
* Converts face X position to crop X offset using rule-of-thirds composition
(face positioned at \~38% from left of crop window)
* Handles gap-filling when face detection drops frames
* Detects split-screen and uses fixed crop targeting the guest side
* Deduplicates keyframes (only writes when crop\_x changes by 4+ pixels)

**Crop strategies (auto-selected):**

|Strategy|When Used|Behavior|
|-|-|-|
|`face\_track`|Face detected in ≥30% of frames|Animated crop follows speaker|
|`split\_screen`|Two faces detected on opposite sides|Fixed crop on guest side|
|`center`|No face detected|Static center crop (fallback)|

**Output:** `crop\_data.json` with per-clip keyframe arrays:

```json
{
  "clips": {
    "01": {
      "keyframes": \[
        {"time": 0.0, "crop\_x": 656},
        {"time": 1.5, "crop\_x": 620},
        {"time": 3.2, "crop\_x": 645}
      ],
      "crop\_w": 608,
      "crop\_h": 1080,
      "strategy": "face\_track"
    }
  }
}
```

**Parameters:**

* `--padding 0.15` — Extra space around face (15% of crop width). Increase for more headroom.
* `--smooth 5` — Smoothing window in frames. Higher = smoother but slower response to movement.

\---

### Step 9: Prepare — Extract Clips

Extract each segment via ffmpeg stream copy (near-instant, lossless):

```bash
ffmpeg -y -ss <start> -to <end> -i source.mp4 -c copy clips/clip\_01.mp4
```

### Step 10: Render — Vertical Clips with Smart Crop

Render each clip using the smart crop keyframes from Step 8:

```bash
python3 scripts/render.py \\
  --source source.mp4 \\
  --clips clips.json \\
  --output-dir exports/vertical/ \\
  --crop-data crop_data.json \\
  --crf 23
```

Face-tracked crop follows the speaker using time-varying crop X expressions interpolated between keyframes.

**Encoding settings:**

* `-crf 23` — good balance of quality and file size for review
* `-crf 18` — higher quality for final delivery
* `-preset veryfast` — fast encoding; use `-preset slow` for final masters
* `-movflags +faststart` — enables web playback before full download

**Also render horizontal (source resolution) clips first** as a reference check:

```bash
python3 scripts/render.py --source source.mp4 --clips clips.json \\
  --output-dir exports/vertical/ --horizontal --crf 23
```

**Batch rendering:** Render horizontals first (fast), then verticals. For
verticals at 1080p, render in batches of 3 to avoid timeout. At 720p, batches
of 5 are safe.

**Verify every output:**

```bash
ffprobe -v error -show\_entries stream=codec\_name,width,height,duration \\
  -of csv=p=0 exports/vertical/<id>\_v.mp4
```

\---

### Step 11: Captions — Remotion Animated Overlays (Enhanced)

**Do NOT generate captions by default.** Only when the user explicitly asks.

Remotion renders animated caption overlays as transparent WebM files, which are
then composited onto the video. This produces much higher quality than ffmpeg's
built-in ASS subtitles — with per-word animations, custom colors, and smooth motion.

**Three caption styles:**

|Style|Font|Look|Best For|
|-|-|-|-|
|**Clean**|Arial 52|White text, black outline, bottom-third|Interviews, podcasts, professional|
|**Bold**|Arial 60 Bold|ALL CAPS, yellow active word, pop-in|Business, education, motivation|
|**Bounce**|Impact 72|Bouncy scale, rotating bright colors|Entertainment, reactions, energy|

**Step 11a: Prepare caption data**

```bash
python3 scripts/render_captions.py \\
  --clips clips.json \\
  --transcript transcript.json \\
  --output-dir captions_remotion/ \\
  --style bold
```

This reads the word-level transcript and generates per-clip caption data JSON files
with cleaned text (filler words removed), word timestamps, and style config.

**Step 11b: Render caption overlays with Remotion**

```bash
cd scripts/captions\_remotion
npm install  # first time only
node render\_all.js --data-dir ./ --output-dir ./
```

This renders each clip's captions as a transparent WebM (VP8 with alpha channel)
at 1080x1920, 30fps. Output: `<id>\_captions.webm`.

**Step 11b (alternative):** Use the `--render` flag to do both steps at once:

```bash
python3 scripts/render\_captions.py \\
  --clips clips.json --transcript transcript.json \\
  --output-dir captions\_remotion/ --style bold --render
```

**Step 11c: Composite captions onto video**

```bash
python3 scripts/render.py \\
  --source source.mp4 --clips clips.json \\
  --output-dir exports/vertical/ \\
  --crop-data crop\_data.json \\
  --captions-remotion-dir captions\_remotion/ \\
  --crf 23
```

This overlays the transparent caption WebM onto each clip using ffmpeg's
`overlay` filter. Output: `<id>\_v\_captions.mp4`.

**Caption rules:**

* Max 2 lines per caption event, \~6 words per line
* Position in bottom third of frame (MarginV=500 on 1920px canvas)
* 135px left/right margins (1/8 of 1080px)
* Short phrases (3-6 seconds each), not long transcript segments
* Filler words (um, uh, you know, like, I mean) are automatically removed
* Pop-in animation: spring-based scale + opacity (4 frames)
- Bold style: active word highlighted in accent color
* Bounce style: per-word color cycling with spring scale

**Fallback (no Node.js/Remotion):** If Remotion is not available, use the
original ASS subtitle method:

```bash
ffmpeg -y -i clip\_v.mp4 -vf "subtitles=captions.ass" \\
  -c:v libx264 -preset veryfast -crf 23 -c:a copy clip\_captioned.mp4
```

\---

### Step 12: Export — Platform-Optimized Encoding

Export with platform-specific settings:

|Platform|Codec|Video Bitrate|Audio|Max Duration|
|-|-|-|-|-|
|YouTube Shorts|H.264 High 4.2|12 Mbps|AAC 192k|60s|
|TikTok|H.264 CRF 18|10 Mbps cap|AAC 128k|60s|
|Instagram Reels|H.264 High 4.2|4.5 Mbps|AAC 128k|90s|

**YouTube Shorts (CPU):**

```bash
ffmpeg -y -i clip\_v.mp4 \\
  -c:v libx264 -preset slow -b:v 12M -maxrate 14M -bufsize 24M \\
  -profile:v high -level 4.2 \\
  -c:a aac -b:a 192k -ar 48000 \\
  -pix\_fmt yuv420p -movflags +faststart \\
  exports/shorts/short\_01\_yt.mp4
```

**TikTok (CPU):**

```bash
ffmpeg -y -i clip\_v.mp4 \\
  -c:v libx264 -preset slow -crf 18 \\
  -maxrate 10M -bufsize 20M \\
  -c:a aac -b:a 128k -ar 44100 \\
  -pix\_fmt yuv420p -movflags +faststart \\
  exports/shorts/short\_01\_tt.mp4
```

**Instagram Reels (CPU):**

```bash
ffmpeg -y -i clip\_v.mp4 \\
  -c:v libx264 -preset slow -b:v 4500k -maxrate 5000k -bufsize 10M \\
  -profile:v high -level 4.2 \\
  -c:a aac -b:a 128k -ar 44100 \\
  -pix\_fmt yuv420p -movflags +faststart \\
  exports/shorts/short\_01\_ig.mp4
```

**With NVIDIA GPU (NVENC):** Replace `-c:v libx264 -preset slow` with
`-c:v h264\_nvenc -preset p5 -tune hq` for 5-10x faster encoding.

**Audio normalization (optional):** Add `-af loudnorm=I=-14:TP=-1:LRA=11`
to normalize to -14 LUFS (streaming standard).

\---

## Edit Adjustments

When the user provides revision instructions, timestamps are **clip-relative**
(not source time). Convert: `source\_time = clip\_start + relative\_time`.

**Common edit patterns:**

```
"01\_edits": {
  "start": "start the clip at 00:27 right before 'lifespans have'",
  "end": "extend to finish the thought"
}
```

**Mute a word or phrase:**

```bash
ffmpeg -y -ss <start> -i source.mp4 -t <duration> \\
  -vf "crop=...,scale=1080:1920" \\
  -af "volume=enable='between(t,<clip\_t1>,<clip\_t2>)':volume=0" \\
  -c:v libx264 -preset veryfast -crf 23 -c:a aac -b:a 128k output.mp4
```

**Remove a mid-clip segment (concat):**

```bash
# Render two parts
ffmpeg -y -ss <start> -i source.mp4 -t <part1\_dur> -c copy partA.mp4
ffmpeg -y -ss <part2\_start> -i source.mp4 -t <part2\_dur> -c copy partB.mp4

# Concat
echo "file 'partA.mp4'" > concat.txt
echo "file 'partB.mp4'" >> concat.txt
ffmpeg -y -f concat -safe 0 -i concat.txt -c copy output.mp4
```

\---

## File Organization

```
project/
├── source.mp4              # Original source
├── transcript.json         # Word-level transcript (Step 3)
├── clips.json              # Clip definitions (Step 5)
├── face\_data.json          # Per-frame face positions (Step 7)
├── crop\_data.json          # Smart crop keyframes (Step 8)
├── scripts/
│   ├── transcribe.py       # Step 3: faster-whisper transcription
│   ├── snap\_boundaries.py  # Step 6: snap to word boundaries
│   ├── face\_track.py       # Step 7: MediaPipe face tracking
│   ├── smart\_crop.py       # Step 8: compute crop keyframes
│   ├── render.py           # Step 10: render vertical/horizontal clips
│   ├── render\_captions.py  # Step 11a: prepare caption data
│   ├── export.sh           # Step 12: platform-specific encoding
│   └── captions\_remotion/  # Step 11b: Remotion caption project
│       ├── package.json
│       ├── src/
│       │   ├── index.ts
│       │   ├── Root.tsx
│       │   └── CaptionsComposition.tsx
│       ├── render\_all.js
│       └── \*.json           # Per-clip caption data + rendered .webm overlays
├── exports/
│   ├── horizontal/         # Source resolution, crf 18
│   │   ├── 01\_h.mp4
│   │   └── ...
│   └── vertical/           # 1080x1920, crf 23
│       ├── 01\_v.mp4
│       ├── 01\_v\_captions.mp4  # (if captions requested)
│       └── ...
└── shorts/                 # Platform-optimized exports
    ├── 01\_yt.mp4
    ├── 01\_tt.mp4
    ├── 01\_ig.mp4
    └── ...
```

\---

## Edge Cases \& Pitfalls

### Transcription

* **Whisper CLI times out on CPU.** Always use the `faster\_whisper` Python
library, not the `whisper` CLI command.
* **VAD filter gaps.** `vad\_filter=True` may filter out secondary speakers.
Use `vad\_filter=False` if you need host questions captured.
* **Long videos.** For 50+ min sources, split audio into 10-min chunks and
transcribe in batches of 2 to stay under execution timeouts.

### Face Tracking (MediaPipe)

* **FaceLandmarker model auto-download.** First run downloads \~3MB model to
`\~/.mediapipe/`. This requires internet access. If the download fails, manually
download from: `https://storage.googleapis.com/mediapipe-models/face\_landmarker/face\_landmarker/float16/1/face\_landmarker.task`
* **GPU not required.** FaceLandmarker runs on CPU. Expect \~2-5s per clip at 10 fps.
- **Split-screen detection.** MediaPipe may detect both faces in a split-screen
layout. The smart_crop.py script handles this by selecting the dominant side.
* **Low-light or out-of-frame:** If the speaker moves out of frame or lighting is
poor, face detection drops. smart\_crop.py fills gaps with interpolation and
falls back to the last known position. The `--smooth` parameter controls how
aggressively it fills gaps.
- **Multiple speakers talking over each other.** FaceLandmarker may jump between
faces. Increase `--smooth` to 7-9 for more stable tracking.

### Rendering

* **`-to` vs `-t`.** When `-ss` precedes `-i`, use `-t` (duration), NOT `-to`
(end time). `-to` is ignored for mp4 containers and the clip runs to the end.
* **moov atom not found.** Background ffmpeg processes may be killed before
writing the moov atom. Re-render in foreground mode with `timeout=300`.
* **360p source.** Direct crop from 360p to 1080x1920 is a 5.3x upscale.
The result will be visibly soft. Warn the user before rendering.
* **Batch timeouts.** At 1080p, render verticals in batches of 3. At 720p,
batches of 5 are safe.
* **Animated crop with face tracking adds \~20-30% render time** compared to
static crop, due to ffmpeg's per-frame crop expression evaluation.

### Remotion Captions

* **First-time setup requires `npm install`** in `scripts/captions\_remotion/`.
This downloads Remotion + Puppeteer (\~200MB). Only needed once.
* **Puppeteer downloads Chromium on first render.** Expect a \~150MB download.
If download fails, set `PUPPETEER\_EXECUTABLE\_PATH` to an existing Chrome/Chromium binary.
* **Transparent output requires VP8 codec.** Use `--codec=vp8` (not h264).
VP8 supports alpha channel; h264 does not.
* **Caption timing is relative to clip start.** render\_captions.py converts
absolute timestamps to clip-relative automatically.
* **Filler words are stripped automatically.** "um", "uh", "you know", "like",
"i mean" are removed from captions. If the user wants them kept, edit
the `FILLER\_WORDS` set in render\_captions.py.
* **Remotion falls back to ASS subtitles** if Node.js is not available.
The ASS method works but lacks per-word animations and custom colors.

### Crop Positioning

- **Face tracking handles positioning automatically.** Steps 7-8 (face_track.py + smart_crop.py) detect face position and compute optimal crop coordinates. No manual positioning needed.
- **Always verify with a test frame.** Render one frame before batch rendering to confirm composition.

### Disk \& Performance

* **Disk fills silently.** Delete WAV files immediately after transcription.
Check `df -h` before batch rendering.
* **Face tracking frame extraction** creates raw video in memory (not on disk),
so no additional disk space is needed beyond the output JSON.
* **File sizes.** Expect \~10-20MB per 40-50s vertical clip at crf 23,
\~20-50MB per horizontal clip at crf 18.
* **Caption overlays (WebM)** are \~2-5MB per clip and are rendered separately.

### Content Quality

* **Dangling pronouns.** If a clip starts with "He said..." or "That is..."
without context, move the start forward to a self-contained statement.
* **Negative framing.** Delete clips that portray the subject negatively.
Don't try to salvage with edits.
* **Timestamp vs content cue conflict.** When the user gives both a relative
timestamp AND a content cue ("start at 00:43 right before 'you don't need'"),
trust the content cue. The timestamp is approximate.

\---

## Configuration

These defaults work for most content. Adjust per-project as needed.

|Parameter|Default|When to Change|
|-|-|-|
|Whisper model|`base`|Use `small` for low VRAM, `large-v3` with GPU|
|Clip duration|15-55s|TikTok prefers 21-34s|
|Score threshold|60|Lower for longer videos with fewer highlights|
|Vertical crf|23|Use 18 for final master exports|
|Horizontal crf|18|Use 17 for archive quality|
|Handle duration|0s (exact cut)|Add 1s handles if user wants padding|
|Caption style|none|Only generate when explicitly requested|
|Platform|all|Single-platform if user specifies|

\---

## Dependencies

### Required

* **ffmpeg** — video/audio processing (`ffmpeg -version`)
* **Python 3.8+** — script execution
* **faster-whisper** — transcription (`pip install faster-whisper`)

### Optional

* **yt-dlp** — YouTube download (`pip install yt-dlp`)
* **NVIDIA GPU** — faster transcription and NVENC encoding
* **pysrt** — SRT subtitle parsing (`pip install pysrt`)

### Verify installation

```bash
ffmpeg -version 2>\&1 | head -1
python3 -c "import faster\_whisper; print('faster-whisper OK')"
python3 -c "import yt\_dlp; print('yt-dlp OK')" 2>/dev/null || echo "yt-dlp not installed (optional)"
```

\---

## Version History

* **v1.0** — Initial open-source release. Core pipeline: transcribe, score,
snap, crop, render, export. Three caption styles. Platform-specific encoding.

