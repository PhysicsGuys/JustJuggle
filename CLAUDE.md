# CLAUDE.md — JustJuggle Project Context

> Drop this file in the root of the `PhysicsGuys/JustJuggle` GitHub repo.
> When starting a new Claude session, say: "Read CLAUDE.md and continue the project."

---

## What is JustJuggle?

JustJuggle is an AI-powered juggle counter. The user records a juggling session (soccer ball keepy-ups) either in-browser or via a mobile app, the video is analyzed by a computer vision pipeline, and the result — total juggles, best streak, confidence score, and contact timestamps — is displayed instantly.

**Live site:** https://physicsguys.github.io/JustJuggle  
**GitHub repo:** https://github.com/PhysicsGuys/JustJuggle  
**Owner:** PhysicsGuys

---

## Goals

### MVP (complete)
- User records video **inside** the app (no uploading from camera roll)
- Video auto-uploads to backend immediately on stop
- CV pipeline analyzes video and returns juggle count
- App displays: total juggles, best streak, confidence score, contact timestamps

### Out of scope for MVP
- Real-time detection
- Teams / leaderboards
- Coaching feedback
- Face masking
- Upload from gallery

### Future
- Deploy FastAPI backend to Railway or Render
- Publish mobile app to App Store / Google Play via EAS
- Improve CV accuracy on real footage beyond ~80%
- Add async processing queue for longer videos

---

## Project Structure

```
JustJuggle/
├── index.html              ← Full website (single file, GitHub Pages)
├── CLAUDE.md               ← This file
│
├── cv_service/
│   ├── juggle_detector.py  ← Phase 1: Python CV pipeline
│   └── test_cv_service.py  ← Phase 1 tests (8/8 juggle accuracy)
│
├── backend/
│   ├── main.py             ← Phase 2: FastAPI backend
│   ├── juggle_detector.py  ← CV service (copy used by backend)
│   ├── requirements.txt    ← Python deps
│   └── test_backend.py     ← Phase 2 tests (21/21 passing)
│
├── mobile/
│   ├── App.tsx             ← Phase 3: React Native / Expo app
│   ├── app.json            ← Expo config
│   └── package.json
│
└── project/
    ├── test_e2e.py         ← Phase 4: End-to-end integration tests
    └── README.md           ← Technical README
```

---

## Development Phases — Status

| Phase | Component | Status | Test Result |
|-------|-----------|--------|-------------|
| 1 | CV service (`juggle_detector.py`) | ✅ Complete | 8/8 juggles, 100% detection |
| 2 | FastAPI backend (`main.py`) | ✅ Complete | 21/21 tests passing |
| 3 | React Native app (`App.tsx`) | ✅ Complete | PRD compliant |
| 4 | End-to-end integration | ✅ Complete | 21/21 tests passing |
| Web | Browser version (`index.html`) | ✅ Live | physicsguys.github.io/JustJuggle |

---

## The CV Pipeline (juggle_detector.py + index.html)

This is the core of the product. It runs in two places:

### Python backend (juggle_detector.py)
Used when the FastAPI server is running locally or deployed.

**5-step pipeline:**
1. **Ball detection** — Per frame: HSV masking for white/color range → morphological cleanup → HoughCircles → best candidate scored by pixel fill inside circle
2. **Trajectory tracking** — Rolling 5-frame moving-average smooths position history
3. **Contact detection** — Ball reverses from downward→upward motion for ≥3 frames = one juggle. Debounced at 8 frames minimum between events
4. **Counting** — Each debounced direction reversal = +1 juggle
5. **Streak detection** — Gap of 20+ frames without ball detection = "drop" → resets streak counter

**Output JSON (matches PRD spec exactly):**
```json
{
  "total_juggles": 12,
  "best_streak": 8,
  "contacts": [{"time": 1.17, "type": "unknown"}],
  "confidence_score": 0.91,
  "_meta": {"request_id": "a3f9b2c1", "processing_time": 1.842}
}
```

### Browser CV (inside index.html — simulateAnalysis function)
Used when no backend is running (GitHub Pages deployment). Runs entirely in JavaScript.

**Upgraded high-accuracy approach (latest version):**
- Seeks through video at 30fps equivalent using `video.currentTime` seeks
- **58-second analysis budget** — adaptively subsamples if video is too long
- Canvas size: 320×180px for better centroid accuracy
- Ball detection uses **color-specific pixel check** based on user's selected ball color
- **Outlier removal** — removes positions where ball "teleports" (speed > 400px/s = false positive)
- **5-tap Gaussian smooth** `[0.06, 0.24, 0.40, 0.24, 0.06]` on Y axis
- **Zero-crossing contact detection** — finds exact moment velocity crosses from positive (falling) to negative (rising)
- Hard minimum 0.2s gap between juggles (physical constraint)
- Streak gap threshold: 0.8s without contact = drop
- Shows live frame count during analysis: `scanning frames — 847 / 1800`
- Falls back to manual adjust UI if confidence < 35%

---

## Ball Color Detection

On first load, a fullscreen overlay asks the user to pick their ball color. This feeds directly into the CV pixel detection logic.

**Supported colors and their detection logic:**
```javascript
const BALL_COLOR_CONFIG = {
  white:  { check: (r,g,b) => (r+g+b)/3 > 185 && Math.max(r,g,b)-Math.min(r,g,b) < 60 },
  yellow: { check: (r,g,b) => r > 180 && g > 150 && b < 100 && r > b+80 },
  orange: { check: (r,g,b) => r > 180 && g > 80 && g < 160 && b < 80 && r > g+40 },
  red:    { check: (r,g,b) => r > 150 && g < 80 && b < 80 && r > g+70 },
  blue:   { check: (r,g,b) => b > 150 && r < 100 && b > r+50 },
  other:  // falls back to white detection
}
```

**To add a new color:** add an entry to `BALL_COLOR_CONFIG` in `index.html`, add a `.ball-option` button in the picker HTML with `onclick="selectBall('newcolor','#hexcode')"`.

---

## Website (index.html) — Full Feature List

Single HTML file. No build step. Deployed to GitHub Pages.

**Sections:**
- Fixed nav with smooth scroll
- Hero with animated grid background + green radial glow
- Stats strip (accuracy, tests, session length)
- "How it works" 5-step grid
- Live recording section (left: feature list, right: recorder card)
- CV pipeline tech breakdown cards
- Footer

**Recording flow (browser states):**
```
idle → recording → uploading → analyzing → results → (idle via "Record again")
                                                    ↓ (error state if CV fails)
```

**Recorder features:**
- Ball color picker overlay on first load
- MediaRecorder API (records as `video/webm`)
- Camera opened via `getUserMedia({ video: true })`
- Guide box overlay while recording ("keep ball visible")
- Blinking red REC dot + timer
- **Max recording time: 10 minutes (600 seconds)**
- Auto-stops at max duration
- After stop: immediately starts upload/analysis (no user action needed)

**Share button behavior:**
1. If `navigator.canShare({ files })` → opens native share sheet with video file attached (works on iPhone/Android)
2. If Web Share API but no file support → shares text result
3. Fallback → downloads the video file + shows toast "Video downloaded!"

**Manual correction widget:**
- Shown when `confidence < 0.35` or `total_juggles === 0`
- `−` / `+` buttons to adjust juggle count manually
- Updates `res-total` display live

---

## Backend API (main.py)

**Stack:** FastAPI + uvicorn + aiofiles

**Endpoints:**
| Route | Method | Description |
|-------|--------|-------------|
| `/` | GET | Service info |
| `/health` | GET | Health check → `{"status": "ok"}` |
| `/analyze` | POST | Upload video → juggle count JSON |

**POST /analyze — validation:**
- Accepted MIME types: `video/mp4`, `video/quicktime`, `video/x-msvideo`, `video/webm`
- Max file size: **150 MB** (enforced during streaming, not after full load)
- Chunk size: 256 KB
- Wrong type → 415, Too large → 413, Missing file → 422, CV error → 500
- Temp file always deleted in `finally` block
- Request ID + processing time returned in `_meta`

**To run locally:**
```bash
cd backend
pip install -r requirements.txt
uvicorn main:app --host 0.0.0.0 --port 8000
# Docs at http://localhost:8000/docs
```

**To deploy (Railway — recommended for MVP):**
```dockerfile
FROM python:3.11-slim
RUN apt-get update && apt-get install -y \
    libgl1-mesa-glx libglib2.0-0 libsm6 libxrender1 libxext6
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```
```bash
npm install -g @railway/cli
railway login && railway init && railway up
```

---

## Mobile App (App.tsx)

**Stack:** React Native + Expo + expo-camera + expo-file-system

**6 screens:**
1. `IdleScreen` — "Start Session" button
2. `RecordingScreen` — live Expo Camera, red dot, timer, stop button
3. `UploadingScreen` — spinner
4. `AnalyzingScreen` — spinner
5. `ResultsScreen` — juggle count, streak, contacts, confidence bar
6. `ErrorScreen` — message + retry

**Key behavior:**
- Uses `CameraType.back`, records at `VideoQuality['720p']`
- Max duration: 60 seconds (mobile) — update `MAX_DURATION_SECONDS` to change
- On stop: calls `uploadAndAnalyze(videoUri)` immediately
- Deletes local temp file after successful upload
- Backend URL set via `EXPO_PUBLIC_API_URL` env var

**To run:**
```bash
cd mobile
npm install
EXPO_PUBLIC_API_URL=http://YOUR_IP:8000 npx expo start
```

**To publish (EAS):**
```bash
npm install -g eas-cli
eas login
eas build:configure
eas build --platform all --profile production
eas submit --platform ios    # → TestFlight
eas submit --platform android  # → Play Store
```

---

## Known Issues & Improvement Areas

### CV accuracy on real video
The browser CV works well for **white balls in good lighting**. For other colors, detection is less reliable because:
- Browser canvas pixel detection is less precise than OpenCV's HoughCircles
- Background objects of similar color get picked up as false positives
- The `radius < 60` blob size filter helps but isn't perfect

**Next improvement:** Add a background subtraction step — capture the first frame as "background", subtract it from subsequent frames, then only look for the ball in the difference image. This would dramatically reduce false positives.

### Analysis speed on long videos
10-minute videos at 30fps = 18,000 seeks. Even with the 58s budget and adaptive subsampling, this can be slow on low-end devices. Consider adding a "quick analysis" mode at 10fps for long sessions.

### Backend not deployed
The FastAPI backend only runs locally right now. The website falls back to browser CV. Once backend is deployed to Railway, update `API_URL` in `index.html` from `'http://localhost:8000'` to the live Railway URL.

---

## Design System

**Colors:**
```css
--green: #1D9E75
--green-light: #5DCAA5
--dark: #060d08
--dark2: #0a1a0f
--dark3: #0f2419
--red: #e24b4a  /* recording indicator */
```

**Fonts:**
- Display/headings: `Syne` (800 weight)
- Body: `DM Sans`
- Code/mono: `DM Mono`

**Border radius:** 16–20px on cards, 30–32px on pill buttons

---

## How to Continue in a New Session

Tell Claude:

> "Read CLAUDE.md. I want to [your next task]."

**Common next tasks:**
- "Deploy the FastAPI backend to Railway and update the API_URL in index.html"
- "Improve the CV background subtraction to reduce false positives"
- "Add a quick-analysis mode (10fps) for sessions longer than 2 minutes"
- "Add a results history so users can see their past sessions"
- "Fix the CV for [color] balls — it's not detecting them"
- "Add the GitHub MCP connector so you can push index.html directly"

---

## File That Gets Deployed

Only `index.html` is deployed to GitHub Pages. All other files are reference/backend code.

To update the live site:
1. Edit `index.html`
2. Commit and push to `main` branch
3. GitHub Pages auto-deploys in ~30 seconds
4. Live at: **https://physicsguys.github.io/JustJuggle**
