# Doctyn Local Course Video Agent

Turns a PowerPoint deck and a slide-by-slide voiceover script into a narrated
course video — locally, on Windows, from Git Bash.

```
PPTX + DOCX voiceover
→ validate slide mapping
→ render slides (LibreOffice → PDF → 1920x1080 PNG)
→ TTS, one MP3 per slide (Windows local voices or OpenAI)
→ exact durations (ffprobe)
→ one H.264/AAC segment per slide (FFmpeg)
→ final-video.mp4 + captions.vtt/.srt + transcript.txt + thumbnail.jpg + course.json
→ final validation
```

Everything is written to `./output/<course-slug>/`. Nothing is uploaded anywhere.
With `TTS_PROVIDER=windows` the whole build is offline; with `TTS_PROVIDER=openai`
the only network call is OpenAI TTS during `npm run build-course`.

---

## 1. Requirements

| Tool | Notes |
|---|---|
| Node.js 20+ | `node -v` |
| FFmpeg + ffprobe | Must include `libx264`, `aac`, `libmp3lame` (the Gyan "full" build does) |
| LibreOffice | Detected automatically at `C:\Program Files\LibreOffice\program\soffice.exe` |
| TTS | Either Windows built-in voices (free, nothing to install) **or** an OpenAI API key |

PDF → PNG uses `pdftoppm` if it is installed; otherwise the bundled MuPDF npm
package is used. No extra install is required.

Install FFmpeg / LibreOffice if missing (run in Git Bash, then **restart Git Bash**):

```bash
winget install Gyan.FFmpeg
winget install TheDocumentFoundation.LibreOffice
```

## 2. Setup (once)

```bash
cd ~/OneDrive/Desktop/doctyn-course-agent
npm install
cp -n .env.example .env     # skip if .env already exists
```

Then choose a TTS provider in `.env` (section 2a or 2b). All other settings have
defaults — see `.env.example` (lead-in/tail silence per slide, concurrency, tool paths).

### 2a. Windows local voices — free, offline

```
TTS_PROVIDER=windows
WINDOWS_TTS_VOICE=
WINDOWS_TTS_RATE=0
WINDOWS_TTS_VOLUME=100
```

- Uses the built-in Windows speech engine (System.Speech via Windows PowerShell). No API key, no internet, no cost.
- `WINDOWS_TTS_VOICE` empty → an installed English voice is chosen automatically (British English first, then US English), and the build prints which one it is using.
- Set it to an exact or partial name to pick a voice: `WINDOWS_TTS_VOICE=Zira`, `WINDOWS_TTS_VOICE=David`, `WINDOWS_TTS_VOICE=Microsoft Hazel Desktop`.
- `WINDOWS_TTS_RATE`: -10 (slowest) to 10 (fastest), 0 = normal. `WINDOWS_TTS_VOLUME`: 0 to 100.
- Windows writes WAV; FFmpeg converts each one to `audio/slide-NNN.mp3`.
- `npm run test-services` lists the installed voices. More voices: Settings → Time & language → Speech → Manage voices (only voices that appear in `test-services` can be used).

### 2b. OpenAI voices — paid, higher quality

```
TTS_PROVIDER=openai
OPENAI_API_KEY=sk-...
TTS_MODEL=gpt-4o-mini-tts
TTS_VOICE=alloy
# optional
TTS_INSTRUCTIONS=Speak in a calm, clear, professional tone suitable for a medical education course.
TTS_SPEED=1.0
```

### Switching providers

The provider and voice are part of each slide's audio checkpoint key. Switching
provider or voice and re-running `npm run build-course` regenerates the audio
(and therefore the segments and final video) so voices are never mixed. Slide
images are **not** re-rendered — they only depend on the PPTX.

Tool paths in `.env` may be written Windows style or Git Bash style:

```
LIBREOFFICE_PATH=C:\Program Files\LibreOffice\program\soffice.exe
LIBREOFFICE_PATH=/c/Program Files/LibreOffice/program/soffice.exe
```

## 3. Check the machine

```bash
npm run test-services
```

Checks Node, the input/output/temp folders, the TTS setup, FFmpeg (+ encoders),
ffprobe, LibreOffice, the PDF renderer and sharp. It then runs free local smoke
tests: builds a 2-slide PPTX, converts it with LibreOffice, renders PNGs, encodes
a test segment, and synthesizes a short sentence with Windows speech → MP3.

**No OpenAI call is ever made by `test-services`.**

- `TTS_PROVIDER=openai`: `OPENAI_API_KEY` must be set (its quota is not tested). Windows speech is reported as optional.
- `TTS_PROVIDER=windows`: the OpenAI key is not required and never checked against the API; Windows speech must work.

```bash
npm run test-services -- --quick    # skip the smoke tests
```

Exit code is non-zero if any check fails.

## 4. Add the course files

Put **exactly one** `.pptx` and **exactly one** `.docx` in `./input`:

```bash
cp "/c/path/to/Risk Factors for Diabetes.pptx" input/
cp "/c/path/to/Risk Factors for Diabetes - Voiceover.docx" input/
ls input
```

Office lock files such as `~$Risk Factors.docx` are ignored automatically.

### Voiceover DOCX format

```
Slide 01 — Risk Factors for Diabetes
VOICEOVER

Welcome to this module...

Slide 02 — Why Risk Factor Assessment Is Critical
VOICEOVER

Diabetes is not an on-and-off disease...
```

- Headings: `Slide NN — Title` (em dash, en dash, `-`, or `:` all work; `Slide 3` with no title is accepted with a warning).
- Everything after `VOICEOVER` up to the next slide heading is narration. Multiple paragraphs are kept.
- Other all-caps labels inside a slide block (e.g. `ON-SCREEN TEXT`, `SPEAKER NOTES`) end the narration, so their content is not spoken.
- If a block has no `VOICEOVER` marker, all text under the heading is used (with a warning).
- Text before the first `Slide` heading is ignored.

## 5. Validate (free, no API calls)

```bash
npm run validate
```

Reports PPTX slide count, narration count, missing / duplicate / out-of-range
slide numbers, empty narration, malformed headings, and a slide-by-slide
mapping table. Writes `output/<slug>/qa/slide-mapping.txt`, and on failure
`output/<slug>/qa/validation-error.txt`. Exits non-zero on failure.

## 6. Build the course

```bash
npm run build-course
```

If validation fails, the build stops **before** any TTS or video work, writes
`qa/validation-error.txt` and exits non-zero.

On success it prints:

```
COURSE BUILD COMPLETE

Course: Risk Factors for Diabetes
Slides: 52
Voiceovers: 52
Audio files: 52
Video segments: 52
Duration: 24m 31s

Output:
C:\...\output\risk-factors-for-diabetes\final-video.mp4

Validation: PASS
```

Open the result:

```bash
start output/risk-factors-for-diabetes/final-video.mp4
explorer "$(cygpath -w output/risk-factors-for-diabetes)"
```

### Resume / re-run

Every stage is recorded in `output/<slug>/checkpoint.json` with a content hash.
Re-running `npm run build-course` reuses anything whose inputs did not change
and whose files still exist:

- Edit one slide's narration → only that slide's MP3 is regenerated (one TTS job), then its segment, captions and the final video.
- Change `SLIDE_TAIL_SECONDS` → segments/captions/video are rebuilt; **no TTS calls**.
- A crash during FFmpeg → audio is never regenerated; each MP3 is checkpointed the moment it is written.

To force a full rebuild of one course:

```bash
rm -rf output/risk-factors-for-diabetes
```

To force regeneration of just one slide's audio, delete its file:

```bash
rm output/risk-factors-for-diabetes/audio/slide-007.mp3
npm run build-course
```

## 7. Output

```
output/<course-slug>/
├── final-video.mp4          learner-facing video (H.264 / AAC / yuv420p / +faststart)
├── thumbnail.jpg            1280x720 from slide 1
├── captions.vtt
├── captions.srt
├── transcript.txt
├── course.json
├── presentation.pptx        copy of the input deck
├── voiceover-script.docx    copy of the input script
├── checkpoint.json          resume state
├── slides/slide-001.png …   1920x1080, aspect ratio preserved (padded if needed)
├── audio/slide-001.mp3 …
├── video-segments/slide-001.mp4 …
├── source/                  original input files with their original names
└── qa/
    ├── slide-mapping.txt     validation report + mapping table
    ├── validation-error.txt  only when validation fails
    ├── tts-script.txt        exact text sent to TTS (after notation normalization)
    └── final-validation.txt  final PASS/FAIL checklist
```

### Timing

Each segment = `SLIDE_LEAD_IN_SECONDS` + measured MP3 duration + `SLIDE_TAIL_SECONDS`,
rounded up to a whole video frame. The slide image is on screen for the whole
segment. Captions for a slide are placed inside its narration window, computed
from the ffprobe-measured durations; within the window, cues are split by
sentence and timed in proportion to their length.

### Medical notation for speech

Only the TTS input is normalized; captions and transcript keep the original text.
Examples: `β-cell` → "beta cell", `HbA1c` → "H B A one C", `mg/dL` →
"milligrams per decilitre", `kg/m²` → "kilograms per square metre", `mmol/L`,
`mmHg`, `≥`, `≤`, `±`, `e.g.`. Review `qa/tts-script.txt` to see exactly what was
spoken. Disable with `TTS_NORMALIZE_MEDICAL=false`.

## 8. Troubleshooting

| Message | Fix |
|---|---|
| `LibreOffice was not found. Checked: …` | Install LibreOffice or set `LIBREOFFICE_PATH` in `.env` |
| `LibreOffice conversion attempt 1 failed; retrying...` | Normal on the very first run (LibreOffice initialises its private profile in `temp/`) |
| `LibreOffice did not produce a PDF …` | Close all LibreOffice windows and `soffice.bin` in Task Manager; check the PPTX opens in Impress |
| `LibreOffice exported N PDF pages, but the PPTX has M slides` | Re-save the deck from PowerPoint; check for corrupt slides |
| `OpenAI rejected the API key (401)` | Fix `OPENAI_API_KEY` in `.env`, or use `TTS_PROVIDER=windows` |
| `Your OpenAI account has run out of quota` | Add billing credit, or set `TTS_PROVIDER=windows` and re-run (slide images are reused) |
| `WINDOWS_TTS_VOICE="…" is not an installed Windows voice` | Use a name listed by `npm run test-services`, or leave it empty |
| `No English Windows speech voice is installed` | Add one in Settings → Time & language → Speech, or set `WINDOWS_TTS_VOICE` |
| `Rate limit reached` | Lower `TTS_CONCURRENCY`, re-run |
| `FFmpeg was not found` | `winget install Gyan.FFmpeg`, restart Git Bash, or set `FFMPEG_PATH` |
| `The … folder is not writable` | Pause OneDrive sync or close the video player holding the file |

## Project layout

```
src/
├── build-course.ts         full pipeline (npm run build-course)
├── validate.ts             mapping check   (npm run validate)
├── test-services.ts        environment check (npm run test-services)
├── services/
│   ├── inputs.ts           find the one PPTX + one DOCX, ignore ~$ files
│   ├── parse-pptx.ts       slide count/order/titles from the PPTX ZIP/XML
│   ├── parse-docx.ts       "Slide NN — Title / VOICEOVER" sections (mammoth)
│   ├── validation.ts       slide ↔ narration mapping rules and reports
│   ├── course.ts           output paths, shared load + validation files
│   ├── render-slides.ts    LibreOffice → PDF → PNG, thumbnail
│   ├── tts.ts              OpenAI TTS, chunking, medical speech normalization, audio keys
│   ├── tts-windows.ts      Windows System.Speech TTS (PowerShell) → WAV → MP3
│   ├── video.ts            ffprobe, per-slide segments, final concat
│   └── captions.ts         VTT / SRT / transcript
└── utils/                  config (.env), checkpoint, exec, tool discovery, files, slug, log, errors
```

Type-check: `npx tsc --noEmit` (or `npm run typecheck`).
