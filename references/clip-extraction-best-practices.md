---
name: clip-extraction-best-practices
description: "MANDATORY clip extraction workflow. Word-level boundary alignment, Descript transcript integration, FFmpeg safe practices. Must be loaded and followed on EVERY clip render. No exceptions."
version: 1.0.0
category: media
severity: CRITICAL
---
# Clip Extraction — Best Practices (MANDATORY)
**⚠️ CRITICAL: This skill MUST be loaded on every clip extraction. No ad-hoc FFmpeg commands. No shortcuts.**
## When to Use
EVERY time a clip is extracted from a source video. No exceptions.
## Prerequisites — Environment Setup
This workflow runs on a headless Linux host as an unprivileged (non-root) user. Before starting a clip extraction session, verify:
### Python Virtual Environment
```bash
source .venv/bin/activate
```
If venv doesn't exist:
```bash
uv venv && source .venv/bin/activate
uv pip install opencv-python mediapipe requests numpy
```
### MediaPipe Face Landmarker Model
```bash
ls models/mediapipe/face_landmarker_v2_with_blendshapes.task
```
If missing:
```bash
mkdir -p models/mediapipe
curl -sL -o models/mediapipe/face_landmarker_v2_with_blendshapes.task \
  "https://storage.googleapis.com/mediapipe-models/face_landmarker/face_landmarker/float16/1/face_landmarker.task"
```
### MediaPipe GL Library (Headless Host Workaround)
On a headless host without a GPU, MediaPipe's C bindings need `libGLESv2.so.2` and `libEGL.so.1` (not installed system-wide). The fix uses Chromium's bundled copies:
```bash
export LD_LIBRARY_PATH=".local/lib:$LD_LIBRARY_PATH"
```
Always set `LD_LIBRARY_PATH` before running any Python code that loads MediaPipe. Without it you get:
```javascript
OSError: libGLESv2.so.2: cannot open shared object file
```
### Descript API Import Path
```python
import sys
sys.path.insert(0, 'scripts/descript')
from descript_api import DescriptClient
```
### Working Directories
```bash
mkdir -p work/episode          # episode working dir
mkdir -p projects/Complete     # final delivery dir
```
## The 11 Pitfalls You Must Avoid
### 1. Arbitrary Timestamps
**NEVER** use rounded-second timestamps (e.g., `-ss 593`). Always find the EXACT word in the Descript transcript and use its millisecond-precise `startTime`/`endTime`.
❌ `ffmpeg -ss 593 -t 42 ...` — lands mid-word
✅ `ffmpeg -ss 599.82 -t 98.4 ...` — word-aligned
### 2. Descript Transcript Gaps
If the word-level transcript has gaps (60+ seconds of no words), use block-level speaker boundaries instead. Don't assume silence.
### 3. User Text vs Descript Text Mismatch
Match by KEY PHRASES, not word-by-word. The user's text defines the CONTENT; the Descript transcript defines the TIMING.
### 4. "Keep Length" Ambiguity
When the user provides exact text, the TEXT takes priority over duration comments.
### 5. Multiple Competing FFmpeg Processes
**ALWAYS** kill existing FFmpeg before starting new batch:
```bash
pkill -f "ffmpeg.*source.mp4" 2>/dev/null
sleep 1
```
### 6. Piping FFmpeg Through tail/grep
**NEVER** pipe FFmpeg output. It causes SIGPIPE and corrupts the moov atom.
❌ `ffmpeg ... 2>&1 | tail -3`
✅ `ffmpeg ... > /tmp/render.log 2>&1; tail -5 /tmp/render.log`
### 7. Discord File Size Limits
Create Discord versions at 540x960, crf 23, targeting 5-15MB.
### 8. Re-encoding from Already-Encoded Files
Always re-render from the ORIGINAL `source.mp4`, never from intermediate files.
### 9. Wrong Skill for the Job
When the user says "run the clipping process on this episode" with a Descript URL, use **THIS** skill (`clip-extraction-best-practices`). Do NOT load or use:
- the Premiere Pro UXP plugin skill — that's for editor-plugin development, not episode clipping
- the content-production loop skill — that's for the render → host → publish pipeline, not ad-hoc clipping
The clipping process = resolve Descript URL → publish → extract transcript → find word-aligned boundaries → render vertical clips with FFmpeg.
### 10. Large Source Video — Google Drive Streaming Alternative
When the source video is too large to fully download (e.g., a 3GB file on a box with \~2.6GB free), use HLS streaming from Google Drive to extract only the needed segments instead of downloading the full MP4 first.
**The problem:** `curl -L -o source.mp4 <drive_url>` downloads the ENTIRE file before FFmpeg can start extracting clips. This fails when the file exceeds available disk space.
**The fix — HLS streaming extraction:**
```bash
# 1. Get the share page URL from Descript publish output
share_url="https://share.descript.com/view/<slug>"

# 2. Scrape the share page HTML to find the HLS playlist URL
curl -sL "$share_url" > share_page.html
# Look for the .m3u8 URL in the HTML — typically a CloudFront or GCS HLS manifest
# Pattern: "https://*.cloudfront.net/*.m3u8" or similar

# 3. Get the HLS playlist (may be a nested master → media playlist)
hls_master=$(grep -oP 'https?://[^"'"'"' ]+\.m3u8[^"'"'"' ]*' share_page.html | head -1)
# Download the master playlist, find the highest quality variant
# Download the media playlist for segment URLs

# 4. Generate a concat demuxer file with just the segments for each clip
#    Use FFprobe on the HLS playlist to map clip times to segment indices
#    Then create a concat file referencing only those segments

# 5. Extract each clip
ffmpeg -y -f concat -safe 0 -i concat_<clip>.txt \
  -c:v libx264 -preset fast -crf 18 \
  -c:a aac -b:a 128k -movflags +faststart \
  output_clip.mp4 > /tmp/render.log 2>&1
```
**Key details:**
- The Descript share page HTML contains the HLS manifest URL as a JSON blob in the page
- HLS segments are .ts files (typically 2-10s each)
- Each vertical clip needs its own concat file listing just the segments in its time range
- After extracting horizontal clips via HLS, render vertical versions from the horizontal masters (second pass)
- This avoids downloading the full 3GB source — only the \~100MB of segments you actually need
**Pitfall: HLS URL expires.** The share page HLS URLs also expire (\~24h like the download URL). Extract the manifest and all segment URLs immediately after publishing.
**Pitfall: Segment alignment.** HLS segment boundaries may not align perfectly with your desired clip boundaries. Add 2-3 seconds of padding on each side and trim to exact boundaries in the final render pass:
```bash
# First pass: extract wider range via concat
# Second pass: trim to exact word-boundary timestamps
ffmpeg -y -ss <start_pad> -i rough_concat.mp4 -t <exact_duration> -c copy final_clip.mp4
```
### 11. Descript Upload Rate Limiting (429)
The Descript upload endpoint (`POST /v1/jobs/import/project_media`) has SEPARATE, stricter rate limits than the publish endpoint. After uploading 2-3 files in quick succession, you WILL get 429 "too many requests" errors. This is session-scoped.
**Fix:**
- Wait 120s between upload attempts (not 60s — the rate limit window is longer than you think)
- Create a fresh `DescriptClient()` instance before each retry (resets session pool)
- Use 30s gaps between successful uploads to avoid triggering the limit
- If you get 409 "already exists", the file was uploaded successfully — skip it
**Pitfall: Upload ≠ processing.** A 200 response on the PUT means the file landed in GCS. Descript still needs to process it before it appears in the project UI. The API may return empty `media_files` for several minutes after upload.
## Correct Workflow (Follow Exactly)
### Step 0: Load Show Profile (MANDATORY)
Before anything else, load the show's profile file `show_profiles/<CODE>.json` AND its `Branding Guidelines` (`show_profiles/<CODE>.branding.md`). Together they lock layout, palette, fonts, caption style, crop rule, lower-thirds/end-cards, and verify gates for that show.
- Branding rules in the guidelines are binding: exact palettes, font assignments, mandatory elements (wrappers, end cards) and compliance gates are specified per show and must be followed literally. Never infer brand specs.
- Compliance gates are per-show and non-negotiable — they typically require scrubbing all client data, never exposing system names or plant layouts, and treating exact title spelling as a hard QA gate. Read the show's gates before rendering.
- `palette_hint` hexes are reference only — confirm from show masters (templates, asset folders) before export. Never reconstruct logos/type from screenshots.
- If any field is `"tbd"` or in `needs` (caption font/colors/style, layout confirm), STOP and ask the user — never guess brand specs.
- Captions are ALWAYS burned in, per-show style. `captions.burned=false` is not a valid profile state.
- `crop.forbidden` (`scale_pad_letterbox`, `center_crop_blind`) must never appear in any FFmpeg filtergraph. The `scale+pad` path is deprecated.
### Step 1: Resolve the Episode → Source Video + Transcript
**For `web.descript.com/<project_id>/<hint>` URLs** (the episode clipping use case):
```python
from descript_api import DescriptClient
client = DescriptClient()

# 1. Resolve URL
info = client.resolve_web_url(url)
comp = info['matched_composition']
project_id = info['project']['id']
comp_id = comp['id']

# 2. Publish → get share URL + download URL
result = client.publish_project(project_id, comp_id)
job_id = result['job_id']
final = client.wait_for_job(job_id, timeout=900)
share_url = final['result']['share_url']       # share.descript.com/view/<slug>
download_url = final['result']['download_url']  # GCS MP4, expires 24h

# 3. Download source video immediately
os.makedirs('work/episode', exist_ok=True)
os.makedirs('work/episode/rendered/<slug>', exist_ok=True)
curl -L -o "work/episode/<slug>-source.mp4" "<download_url>"

# 4. Extract word-level transcript from share page
#    The transcript URL is embedded in the share page HTML as JSON metadata
import requests, re
r = requests.get(share_url, timeout=30)
m = re.search('"transcript"\s*:\s*\{\s*"url"\s*:\s*"([^"]+)"', r.text)
transcript_url = html.unescape(m.group(1))
tdata = requests.get(transcript_url, timeout=30).json()
with open('work/episode/<slug>-transcript.json', 'w') as f:
    json.dump(tdata, f)
```
**CRITICAL: Download the video and transcript IMMEDIATELY after publish.** The download URL expires in \~24h and the transcript URL is time-limited (\~12h).
### Step 2: Analyze Transcript → Identify Clip Candidates
Read the full transcript text and score segments on the 5-dimension rubric:
- **Hook Strength (30%)**: Contrarian claims, curiosity gaps, value promises
- **Standalone Coherence (25%)**: Self-contained narrative arc
- **Emotional Intensity (20%)**: Passion, surprise, vulnerability, story
- **Value Density (15%)**: Frameworks, named concepts, specific numbers
- **Payoff Quality (10%)**: Punchline, natural stopping point
Target: 6-10 clips at 25-55s each. Min score: 65/100.
For **solo monologues** (single speaker, no interviews): Descript may produce one giant turn covering the entire episode. In this case, work entirely with word-level segments — the turn-level data is not useful for boundary detection.
### Step 3: Find Word-Aligned Boundaries
```python
def find_phrase_in_transcript(phrase, segments, start_after=0, end_before=9999):
    """Find exact word sequence. Returns (start_time, end_time)."""
    words = phrase.lower().split()
    filtered = [(s['startTime'], s['endTime'], s['body']) 
                for s in segments 
                if start_after <= s['startTime'] <= end_before
                and s['speaker'] not in ('Intro', 'Outro',)]
    for j in range(len(filtered) - len(words) + 1):
        if all(filtered[j+k][2].lower().strip(".,!?;:'\"()") == words[k] 
               for k in range(len(words))):
            return filtered[j][0], filtered[j+len(words)-1][1]
    return None
```
Use the `execute_code` tool to run this against the loaded transcript JSON. Search for phrases you identified in Step 2.
### Step 4: Adjust Boundaries for Clean Starts/Ends
**Pitfall: Mid-word starts.** After finding boundaries, check the FIRST word of the clip. If it starts mid-sentence (e.g., "we", "deal", "such", "ask"), the boundary will feel abrupt. Search for a slightly earlier start phrase that lands on a natural sentence start:
```javascript
# Bad start: "we need to think about, okay, how do you give yourself permission"
# Good start: "And so what do we need to think about, okay, how do you give yourself permission"
```
**Pitfall: Clips too short (\<25s).** When `find_phrase_in_transcript` returns boundaries that produce a clip under 25 seconds, EXPAND by finding a different start phrase further back or end phrase further forward — NOT by arbitrarily extending. Use the transcript preview to identify adjacent phrases:
```javascript
# Original: "how do you give yourself permission" → "that is where it sits together" = too short
# Expanded: "And so what do we need" → "that is where it sits together" = better
```
**Handle duration**: Use 1.0s handles by default. For clips where extending would overlap into adjacent content that sounds different (different topic/tone), use 0.5s handles instead.
### Step 5: Render from Original Source — Face-Tracked or Blurred-Background Vertical Crop
**Captions: ALWAYS burned in, per-show style from the show_profile.** Source timing from the Descript word-level transcript. Never ship a short without burned captions. Descript layout templates (e.g. a stacked split-screen layout) may be used ONLY when rendering inside Descript — the FFmpeg pipeline must produce a correct, branded short without any Descript template.
**⚠️ CRITICAL: Do NOT use letterbox/scale+pad.** The old approach produces black bars top/bottom. Use face-tracked crop or blurred-background layout instead.
**Two approaches, chosen based on content:**
1. **Face-tracked crop** (default for most talking-head content) — follows the speaker's face, no letterboxing. Use when the speaker is clearly visible and hat/cap/glasses don't obscure the face.
2. **Blurred-background vertical** (fallback for problematic frames) — center-crops a vertical strip and fills the background with a blurred full-frame version. Use when:
	- The speaker wears a hat/cap that gets cut off badly in a pure face-tracked crop
	- The speaker gestures widely and exits the crop frame
	- The source has multiple speakers on screen and no single face to track
	- Face detection fails consistently (&lt;50% detection rate)
```bash
# Blurred-background approach (two-pass):
# Pass 1: Extract horizontal clip
# Pass 2: Create vertical with blurred background
ffmpeg -i horizontal_clip.mp4 \
  -vf "split[orig][blur]; \
       [blur]scale=1080:1920,boxblur=20:5[bg]; \
       [orig]scale=1080:1920:force_original_aspect_ratio=increase,crop=1080:1920[fg]; \
       [bg][fg]overlay=(W-w)/2:(H-h)/2" \
  -c:v libx264 -preset fast -crf 23 \
  -c:a aac -b:a 128k -movflags +faststart \
  vertical_clip.mp4 > /tmp/render.log 2>&1
```
**Pitfall: Blurred-background is heavier.** This requires a two-pass approach (or a complex filtergraph). Render horizontal first, then convert each to vertical. On a small machine, this doubles render time — plan accordingly.
**Pitfall: User may prefer direct crop.** Check with the user before using blurred-background as the default. The project's domain conventions specify "direct crop, NO blurred backgrounds" as the standard — blurred-background is the exception for problematic source material.
For talking-head / solo monologue content (the vast majority of podcast clips), the speaker's face is the focal point. A proper vertical crop follows the face, eliminating letterboxing and producing a professional full-frame result.
#### Step 5a: Detect Face Positions
**⚠️ Prerequisite:** Before running any MediaPipe code, set `LD_LIBRARY_PATH` (see Prerequisites section above). Without this, FaceLandmarker crashes with `libGLESv2.so.2: cannot open shared object file`.
```python
# detect_faces.py — uses MediaPipe Tasks API (0.10.x+)
import cv2, mediapipe as mp, json, sys
from mediapipe.tasks import python
from mediapipe.tasks.python import vision

base_options = python.BaseOptions(
    model_asset_path='models/mediapipe/face_landmarker_v2_with_blendshapes.task'
)
options = vision.FaceLandmarkerOptions(
    base_options=base_options,
    running_mode=vision.RunningMode.VIDEO,
    num_faces=1,
    min_face_detection_confidence=0.3,
    min_face_presence_confidence=0.3,
    min_tracking_confidence=0.3,
)
detector = vision.FaceLandmarker.create_from_options(options)

cap = cv2.VideoCapture(video_path)
fps = cap.get(cv2.CAP_PROP_FPS)
src_w = int(cap.get(cv2.CAP_PROP_FRAME_WIDTH))
src_h = int(cap.get(cv2.CAP_PROP_FRAME_HEIGHT))

# For 9:16 vertical from 1920x1080: crop_w = 607, crop_h = 1080
crop_h = src_h
crop_w = int(src_h * 9 / 16)

sample_interval = int(fps)  # every 1 second
frame_idx = 0
face_positions = []

while cap.isOpened():
    ret, frame = cap.read()
    if not ret:
        break
    if frame_idx % sample_interval == 0:
        timestamp = frame_idx / fps
        rgb = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
        mp_image = mp.Image(image_format=mp.ImageFormat.SRGB, data=rgb)
        result = detector.detect_for_video(mp_image, int(timestamp * 1000))
        if result.face_landmarks:
            landmarks = result.face_landmarks[0]
            xs = [lm.x for lm in landmarks]
            face_cx = sum(xs) / len(xs)
            crop_x = int(face_cx * src_w - 0.38 * crop_w)
            crop_x = max(0, min(crop_x, src_w - crop_w))
            face_positions.append({"time": round(timestamp, 2), "crop_x": crop_x})
    frame_idx += 1
cap.release()
detector.close()
```
**Pitfall: Wrong MediaPipe API.** MediaPipe 0.10.x removed `mp.solutions.face_detection`. You MUST use the Tasks API (`mp.tasks.python.vision.FaceLandmarker`). The model file is at `models/mediapipe/face_landmarker_v2_with_blendshapes.task`. The old `blaze_face_short_range.tblite` model does NOT work with the Tasks API FaceDetector in 0.10.x — it returns null detections.
**Pitfall: Normalized coordinates.** FaceLandmarker returns coordinates in 0-1 range (not pixel values). `face_cx = 0.44` means 44% of frame width. Convert to pixels: `face_cx * src_w`.
#### Step 5b: Compute Per-Clip Crop Window
For each clip, average the face positions across its time range:
```python
# Get average crop_x for the clip's time range
crop_x_samples = [f['crop_x'] for f in face_positions 
                  if clip_start <= f['time'] <= clip_end]
avg_crop_x = int(sum(crop_x_samples) / len(crop_x_samples))
avg_crop_x = max(0, min(avg_crop_x, src_w - crop_w))
```
For talking-head content, the face barely moves (typically ±50px), so a static per-clip crop is sufficient. Do NOT use animated keyframes unless the speaker moves significantly (\>100px variance).
#### Step 5c: Render
```bash
# Kill competing processes first
pkill -f "ffmpeg.*source.mp4" 2>/dev/null
sleep 1

# Render each clip with face-tracked crop + scale to vertical
ffmpeg -y -ss <start> -to <end> -i work/episode/<slug>-source.mp4 \
  -vf "crop=<crop_w>:<crop_h>:<avg_crop_x>:0,scale=1080:1920" \
  -c:v libx264 -preset fast -crf 20 -c:a aac -b:a 128k \
  -movflags +faststart \
  work/episode/rendered/<slug>/<clip_name>.mp4 > work/episode/rendered/<slug>/<clip_name>.log 2>&1
```
**Key difference from old approach:** `crop=607:1080:<x>:0,scale=1080:1920` crops a vertical strip at the face position, then scales to full vertical resolution. No letterboxing, no black bars.
Render sequentially, not in parallel — avoids I/O contention on small servers.
### Step 6: Verify Output (ffprobe + VISUAL gate)
```bash
for f in work/episode/rendered/<slug>/*.mp4; do
    dur=$(ffprobe -v quiet -show_entries format=duration -of csv=p=0 "$f")
    res=$(ffprobe -v quiet -select_streams v:0 -show_entries stream=width,height -of csv=s=x:p=0 "$f")
    audio=$(ffprobe -v quiet -select_streams a:0 -show_entries stream=codec_name -of csv=p=0 "$f")
    size=$(stat -c%s "$f")
    echo "$(basename $f): ${dur%.*}s $res audio:${audio} ${size}bytes"
done
```
Check: duration 25-55s, resolution 1080x1920, audio codec aac, size \> 0.
**Visual gate (MANDATORY — catches black screens that pass ffprobe):**
```bash
for f in work/episode/rendered/<slug>/*.mp4; do
    echo "== $(basename $f)"
    # 1. Black-frame detection over full duration
    ffmpeg -i "$f" -vf "blackdetect=d=0.5:pix_th=0.10" -an -f null - > /tmp/black.log 2>&1; grep -i black /tmp/black.log || echo "no black segments"
    # 2. Frame grabs at head/mid/tail for visual proof
    dur=$(ffprobe -v quiet -show_entries format=duration -of csv=p=0 "$f" | cut -d. -f1)
    for t in 1 $((dur/2)) $((dur-1)); do
        ffmpeg -y -v error -ss $t -i "$f" -frames:v 1 "/tmp/$(basename $f)_t${t}.jpg"
    done
done
```
**Reject rules:** any `blackdetect` segment covering \>50% of duration → REJECT, re-render, never upload. Missing audio stream → REJECT. No frame grabs → handoff incomplete. Frame grabs are kept as proof with the delivery.
**Ordering invariant:** render → host → write the verification checklist → THEN move the row's status. Never set a row to ready-for-review before the checklist is written. When linking the rendered video, use the correct relation property and re-read the row to confirm it persisted.
### Step 7: Deliver to the Projects Folder
After rendering and verification, organize the output into the project folder at `projects/Complete/<Episode-Name>/`:
```bash
# Create the project directory
mkdir -p projects/Complete/<Guest-Show>/
mkdir -p projects/Complete/<Guest-Show>/exports/horizontal
mkdir -p projects/Complete/<Guest-Show>/exports/vertical
mkdir -p projects/Complete/<Guest-Show>/scripts

# Copy exports
cp work/episode/rendered/<slug>/horizontal_*.mp4 \
   projects/Complete/<Guest-Show>/exports/horizontal/
cp work/episode/rendered/<slug>/vertical_*.mp4 \
   projects/Complete/<Guest-Show>/exports/vertical/

# Copy metadata
cp work/episode/<slug>/clip_candidates.json \
   projects/Complete/<Guest-Show>/
cp work/episode/<slug>/clip_candidates.md \
   projects/Complete/<Guest-Show>/
cp work/episode/<slug>/transcript.txt \
   projects/Complete/<Guest-Show>/
```
Then write a **PROJECT.md** in the project root following this template:
```markdown
# <Guest> (<Show>) — Clip Extraction Project

**Date:** <date>
**Guest:** <guest name>
**Show:** <show>
**Topic:** <episode topic>

---

## Source Specs

| Property | Value |
|----------|-------|
| Resolution | 1920×1080 |
| Duration | ~<duration> |
| Size | <size> |

---

## Clip List (<N> extracted)

### ✅ Clip <N>: "<Title>" — <Ns> (<start> → <end>)
<description>

| File | Path | Size |
|------|------|------|
| Horizontal | `exports/horizontal/<file>.mp4` | <size> |
| Vertical | `exports/vertical/<file>.mp4` | <size> |

(repeat for each clip)

---

## Project Size

| Component | Size |
|-----------|------|
| Exports (horizontal) | <total> |
| Exports (vertical) | <total> |
| **Total** | **<total>** |
```
**Naming convention:** Use a `<Guest>-<Show>` slug (e.g., `Jane-Doe-HSB`). This makes it findable alongside other episodes.
**Pitfall: Don't skip `PROJECT.md`.** The user expects a project summary in every Complete folder. A project without `PROJECT.md` is indistinguishable from work-in-progress.
## Verification Checklist (confirm ALL before handoff)
Before handoff, verify ALL of these:
- [ ] This workflow was loaded and followed
- [ ] Venv sourced: `source .venv/bin/activate`
- [ ] `LD_LIBRARY_PATH` set for MediaPipe (if using face detection)
- [ ] Word-level timestamps used (not rounded seconds)
- [ ] Transcript was searched for exact phrase boundaries
- [ ] 1-second handles added
- [ ] Rendered from ORIGINAL source file (not intermediate)
- [ ] No FFmpeg SIGPIPE (output went to log file, not piped)
- [ ] Output file verified (size \> 0, playable)
- [ ] Pkill ran before FFmpeg (no competing processes)
## find_phrase_in_transcript() — Canonical Implementation
```python
def find_phrase_in_transcript(phrase, segments, start_after=0, end_before=9999):
    """Find exact word sequence in Descript segments. Returns (start_time, end_time)."""
    words = phrase.lower().split()
    filtered = [(s['startTime'], s['endTime'], s['body']) 
                for s in segments 
                if start_after <= s['startTime'] <= end_before
                and s['speaker'] not in ('Intro', 'Outro',)]
    for j in range(len(filtered) - len(words) + 1):
        if all(filtered[j+k][2].lower().strip(".,!?;:'\"") == words[k] 
               for k in range(len(words))):
            return filtered[j][0], filtered[j+len(words)-1][1]
    return None
```
## References
- `references/clip-boundary-alignment.md` — Full pitfall documentation with session evidence
- `references/episode-clipping-workflow-2026-06-26.md` — End-to-end episode clipping walkthrough (a real episode): resolve → publish → extract transcript → find boundaries → render → verify. Contains boundary expansion technique, mid-word start detection, and handle tuning for solo monologues.
- `references/face-tracked-vertical-crop-2026-06-26.md` — MediaPipe FaceLandmarker API for proper 9:16 vertical crop (wrong APIs, crop math, detection stats).
- `references/hls-streaming-extraction-2026-06-27.md` — HLS streaming extraction from Descript share pages (when full 3GB source can't be downloaded). Contains segment mapping, concat demuxer generation, and two-pass render technique.
- `references/blurred-background-vertical-2026-06-27.md` — Blurred-background vertical crop (alternative to face-tracking for hat/headwear problematic content). Filtergraph breakdown, trade-off comparison, and real-session results.
- `references/setup-verification-2026-07-15.md` — Verified environment state: installed packages, GL lib workaround, model paths, working directories.
