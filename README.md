# my-VIDEO-eNGINE

Engine manual for YouTube Shorts on the channel **Ai Simple with Usama**: beginner-friendly AI tools and tips, in English.

Every session starts with **"Follow README.md"**. This file is the only manual. Follow it from top to bottom.

**Repo rule:** this repo holds only `README.md` and `output/`. All tools, clones, working files and renders live **outside** the repo, in `$WORK` (defined below). Never commit anything else.

---

## 1. Purpose

I (Usama) supply a narration audio file. Claude Code transcribes it, directs and renders a finished vertical YouTube Short with the film styles and render toolkit of [lemo-opuscar](https://github.com/lemomo-ai/lemo-opuscar), timed exactly to my narration, checks it, and commits the video, a thumbnail and the metadata to `output/` in this repo.

---

## 2. Session setup (run every session; cloud sessions start empty)

Run each block in order. If any download fails (a network error, a proxy refusal, HTTP 403/407), **stop and report the exact URL that failed**. Do not work around it with a different source.

### 2.1 Work folder (outside the repo)

Use the session scratchpad directory if the system prompt lists one; otherwise `/tmp/video-engine`.

```bash
export WORK=/tmp/video-engine        # or <scratchpad>/video-engine
mkdir -p "$WORK/bin" "$WORK/projects"
export PATH="$WORK/bin:$PATH"
```

`$WORK/bin` must stay first on `PATH` in every later shell command (the shell does not keep `export` between tool calls, so prefix each command with `export WORK=... PATH="$WORK/bin:$PATH";`).

### 2.2 Clone Opuscar fresh

Always clone fresh so the newest styles are included.

```bash
rm -rf "$WORK/lemo-opuscar"
git clone --depth 1 https://github.com/lemomo-ai/lemo-opuscar "$WORK/lemo-opuscar"
export LIB="$WORK/lemo-opuscar"
```

### 2.3 System tools: Node 20+, ffmpeg 7+, Python 3.11+

```bash
node -v                      # need v20 or newer
ffmpeg -version | head -1    # need 7.x or newer
python3 --version            # need 3.11 or newer
```

Install only what is missing or too old:

- **ffmpeg** (Ubuntu 24.04 ships 6.1, which is too old: Opuscar's `mux.sh` rejects its audio length). Use the static build:
  ```bash
  mkdir -p "$WORK/ffdl" && cd "$WORK/ffdl" && curl -fsSL -o ff.tar.xz https://johnvansickle.com/ffmpeg/releases/ffmpeg-release-amd64-static.tar.xz && tar xf ff.tar.xz && cp ffmpeg-*-static/ffmpeg ffmpeg-*-static/ffprobe "$WORK/bin/"
  ```
  Then `ffmpeg -version` must print 7.x (with `$WORK/bin` first on `PATH`).
- **Node 20+**: `curl -fsSL https://deb.nodesource.com/setup_22.x | bash - && apt-get install -y nodejs`
- **Python 3.11+**: if `uv` exists, Opuscar's setup fetches Python itself; otherwise `apt-get install -y python3.12 python3.12-venv`.

### 2.4 Install the Opuscar dependencies

```bash
cd "$LIB" && LEMO_OPUSCAR_HOME="$LIB" sh plugin/skills/lemo-opuscar/scripts/setup.sh deps
```

This installs the Node packages, the headless Chromium renderer and the Python core tier into `$LIB/.venv`. It must end with `packages: core ✓`.

Add the transcriber (word-level timestamps). Do **not** install the `voice` tier: no text-to-speech is used.

```bash
cd "$LIB" && (uv pip install --python .venv/bin/python faster-whisper "av<19" || .venv/bin/python -m pip install faster-whisper "av<19")
.venv/bin/python -c "from faster_whisper import WhisperModel; WhisperModel('small.en', device='cpu', compute_type='int8'); print('whisper ok')"
```

(The model downloads from huggingface.co on first use.)

### 2.5 Smoke test

Render one 1080x1920 still from a trivial page to prove the browser works:

```bash
mkdir -p "$WORK/smoke" && cat > "$WORK/smoke/index.html" <<'EOF'
<!doctype html><body style="margin:0"><canvas id=c></canvas><script>
const c=document.getElementById('c');c.width=innerWidth;c.height=innerHeight;const x=c.getContext('2d');
window.DUR=1;window.render=t=>{x.fillStyle='#123';x.fillRect(0,0,c.width,c.height);x.fillStyle='#fff';x.font='120px sans-serif';x.fillText('ok '+t,100,960)};window.READY=true;</script>
EOF
cd "$LIB" && node core/render/still.mjs "$WORK/smoke" 0.5 --size 1080x1920
```

A note that WebGL uses software rendering is normal in the cloud: prefer 2D/canvas/CSS styles over heavy 3D ones, which render slowly.

### 2.6 Read the guidance

Read, in full: `$LIB/CLAUDE.md` (which points to `AGENTS.md`; read that too), `$LIB/DIRECTOR.md`, `$LIB/TECHNIQUE.md`, `$LIB/styles/README.md`, and `$LIB/core/README.md` (tool commands).

Use them as **filmmaking guidance only**. Where they conflict with this README, **this README wins**. In particular, ignore their instructions to:
- ask the user questions (language, style, treatment approval) — decide yourself (step 4h);
- generate a voice-over with TTS — the narration is my audio file;
- render 1920x1080 — always pass `--size 1080x1920`;
- build inside `$LIB/styles/` — build in `$WORK/projects/<slug>/`.

The treatment, cue map and checks in DIRECTOR.md are still useful: do them, but internally, without pausing.

---

## 3. Inputs per video

| Input | Required | How it arrives |
|---|---|---|
| Narration audio | yes | Attached in the chat, a URL, or a file in `input/` in this repo **only if I say so** (do not commit `input/`). Copy it to `$WORK/projects/<slug>/narration.<ext>`. |
| Topic name | yes | Used for the folder slug: lowercase, hyphens, ASCII only (e.g. "ChatGPT memory tips" → `chatgpt-memory-tips`). |
| Screenshots | optional | Attached images. The only allowed source of logos or brand visuals. |

If no topic is given, derive one from the transcript and record that in the notes.

---

## 4. Production steps

Project folder: `$WORK/projects/<slug>/`. Date: today's date (`YYYY-MM-DD`).

### a. Transcribe with word-level timestamps

```bash
cd "$LIB" && .venv/bin/python - "$WORK/projects/<slug>/narration.<ext>" "$WORK/projects/<slug>/words.json" <<'EOF'
import sys, json
from faster_whisper import WhisperModel
m = WhisperModel('small.en', device='cpu', compute_type='int8')
segs, info = m.transcribe(sys.argv[1], word_timestamps=True, language='en', vad_filter=False)
out = [{"t0": s.start, "t1": s.end, "text": s.text.strip(),
        "words": [{"w": w.word.strip(), "t0": w.start, "t1": w.end} for w in s.words]} for s in segs]
json.dump(out, open(sys.argv[2], "w"), indent=1)
print(" ".join(s["text"] for s in out))
EOF
```

Get the exact audio duration: `ffprobe -v error -show_entries format=duration -of csv=p=0 narration.<ext>`. Call it `DUR`.

Fix obvious transcription errors in the captions only where the audio clearly says something else (e.g. brand names like "ChatGPT"); never change what was said.

### b. The audio is the final narration

- No text-to-speech, no re-recording, no rewording, no trimming, no speed change.
- The film is exactly `DUR` seconds long (within one frame). The 2-second end card falls **inside** `DUR` (the last 2 s of the audio), not after it.
- If the audio is under ~15 s or over 3 minutes, still make it, and note it (Shorts allow up to 3 minutes).

### c. Choose the style

1. Read the **Style Log** (section 7). Every style listed there is excluded.
2. List the current styles: `$LIB/styles/README.md` (folder names) and each `$LIB/styles/<folder>/style.json`. The `num` field is the order styles were added: **higher `num` = newer**.
   ```bash
   cd "$LIB/styles" && for f in */style.json; do python3 -c "import json,sys;d=json.load(open(sys.argv[1]));print(d['num'],d['slug'],'|',d['en'],'|',d['category_en'])" "$f"; done | sort -rn
   ```
3. Pick one that has **not** been used, preferring the newest styles that suit the topic (beginner AI tips: clear, readable, screen-friendly styles work best; avoid heavy 3D/WebGL styles in the cloud because rendering is software-only).
4. Read `$LIB/styles/<folder>/STYLE.md` and follow it (look, colour, type, motion, sound palette). Keep its invariants; adapt layout to vertical 1080x1920. Only after deciding your own structure may you fetch the demo for technique reference: `sh $LIB/plugin/skills/lemo-opuscar/scripts/setup.sh demo <folder>`.
5. Record the style and why it was chosen in the notes.

### d. Build the film, timed line by line

- Build an HTML page in `$WORK/projects/<slug>/` that follows the page contract in `$LIB/core/README.md` (`window.render(t)`, `window.DUR = DUR`, `window.READY`, `window.TEXTS(t)` for the read check). Copy any helpers you need from `$LIB/core/` into the project; do not edit `$LIB`.
- Drive every scene change, visual beat and caption from `words.json` times (one `timeline.js` / JSON that the page, captions and music all read).
- Each spoken line gets its own visual: the key idea as a big, simple on-screen element (an icon drawn in code, a step number, a screenshot I provided, a short label).
- Render: `cd "$LIB" && node core/render/video.mjs "$WORK/projects/<slug>" --fps 30 --size 1080x1920 --out "$WORK/projects/<slug>/out/picture.mp4"` (use the fps from STYLE.md if it sets one).

### e. Format

- 1080x1920 vertical, MP4, H.264 (`yuv420p`) + AAC (48 kHz stereo), `+faststart`.
- Under 50 MB (if larger, re-encode with a higher CRF, e.g. `-crf 23..28`).

### f. Shorts requirements

- **Hook:** a text hook on screen within the first 2 s (visible at t = 0.0–0.5 s), using words from the opening line of the audio.
- **Captions:** burned-in, synced to the spoken words (word-by-word or short 2–5-word groups from `words.json`, the current word highlighted). Also write an `.srt` with `core/render/srt.py` for reference (not committed).
- **Safe zones:** no text in the top 10 % (y < 192 px) or bottom 20 % (y > 1536 px). All text sits between y = 192 and y = 1536, with ≥ 60 px side margins. Captions sit in roughly y = 1150–1500.
- **Music:** quiet, original background music made in code (Opuscar `core/audio/sampler.py` / `pluck.py` per TECHNIQUE.md, or synthesized with numpy). No copyrighted tracks, no downloaded songs. Ducked under the voice: about −20 dB under the narration while speaking, may rise a little in pauses and on the end card.
- **End card:** the last 2 s show `Ai Simple with Usama - Subscribe` (the end card stays within the safe zones).
- **Mix:** narration untouched except loudness; final mix about −14 LUFS, true peak ≤ −1 dBTP (`core/render/mux.sh` does this; see `core/README.md`). Example manual mix if needed:
  ```bash
  ffmpeg -i narration.wav -i music.wav -filter_complex "[1:a]volume=0.25[m];[m][0:a]sidechaincompress=threshold=0.02:ratio=8:attack=20:release=400[d];[0:a][d]amix=inputs=2:normalize=0,loudnorm=I=-14:TP=-1.5[a]" -map "[a]" -ar 48000 -ac 2 mix.wav
  ffmpeg -i out/picture.mp4 -i mix.wav -map 0:v -map 1:a -c:v libx264 -pix_fmt yuv420p -crf 20 -c:a aac -b:a 192k -t DUR -movflags +faststart video.mp4
  ```

### g. Content rules

- Show only claims that are spoken in the audio. No extra facts, numbers, prices, features or promises.
- No logos, brand marks, product UI or brand assets, except screenshots I provide. Name products in plain text only if they are spoken.
- No LemoLab credit or watermark (the films are mine).
- Fonts: only those in `$LIB/core/fonts/` (OFL) or other OFL/CC0 fonts.

### h. No questions, no storyboard pause

Do not stop to ask questions and do not pause for a storyboard or treatment approval. Make reasonable decisions and record each one in the "Decisions" section of `metadata.md`. The only reason to stop is a blocked download (report the URL) or missing audio.

---

## 5. Quality check before delivery

1. **Frames:** extract a contact sheet and individual frames across the film:
   ```bash
   ffmpeg -i video.mp4 -vf "fps=1,scale=270:480,tile=6x5" -frames:v 1 sheet.jpg
   for t in 0.5 1.5 <every line start> DUR-1; do ffmpeg -ss $t -i video.mp4 -frames:v 1 f_$t.jpg; done
   ```
   Look at them. Check: hook visible by 2 s; captions readable (size, contrast) and matching the words spoken at that time; nothing cut off; no text in the top 10 % or bottom 20 %; end card correct in the last 2 s.
2. **Read check:** `node core/render/readcheck.mjs <project> --size 1080x1920` (if the page implements `window.TEXTS`).
3. **Duration:** `ffprobe` duration of `video.mp4` equals `DUR` (difference ≤ 1 frame, i.e. ≤ 0.04 s).
4. **Loudness / balance:** `ffmpeg -i video.mp4 -af ebur128 -f null -` ≈ −14 LUFS; compare music alone vs. narration (music at least ~15–20 dB quieter while speaking). Listen-check by running whisper on the final `video.mp4`: the transcript must match `words.json`.
5. **Blank frames:** `ffmpeg -i video.mp4 -vf blackdetect=d=0.1 -an -f null -` reports none (unless the style intends it).
6. **Format:** 1080x1920, h264 + aac, size < 50 MB.

Fix any problems **once**, re-check, then deliver. Note anything left imperfect in the Decisions section.

---

## 6. Output

Write to `output/YYYY-MM-DD-<slug>/` in this repo:

| File | Content |
|---|---|
| `video.mp4` | The finished Short |
| `thumbnail.jpg` | One strong frame, 1080x1920 JPEG (the hook frame or the clearest key-visual frame) |
| `metadata.md` | See template below |

`metadata.md` template:

```markdown
# <Title>

- Title: <under 100 characters, includes the main keyword>
- Duration: <DUR> s
- Style: <English style name> (`<folder>`, Opuscar #<num>)

## Description
<2–4 short sentences, only claims spoken in the audio>
Subscribe to Ai Simple with Usama for more simple AI tips.
#Shorts #AI <2–3 topic hashtags>

## Tags
<comma-separated, 10–15 tags>

## Transcript
<the transcript>

## Decisions
- <every decision made without asking: style choice and reason, hook text, caption corrections, music, anything imperfect>
```

Then:
1. Append one row to the **Style Log** below.
2. Commit **only** `output/YYYY-MM-DD-<slug>/{video.mp4,thumbnail.jpg,metadata.md}` and `README.md`:
   ```bash
   git add output/YYYY-MM-DD-<slug>/video.mp4 output/YYYY-MM-DD-<slug>/thumbnail.jpg output/YYYY-MM-DD-<slug>/metadata.md README.md
   git commit -m "Add Short: <slug>"
   git push -u origin <branch>
   ```
   Push to the branch the session tells you to use (otherwise `main`). If the push to `main` is refused, open a pull request and report its link.
3. Report: the output folder, title, style, duration, size, and the decisions list.

---

## 7. Style Log

One row per video. A style listed here must not be used again.

| date | video slug | style used |
|---|---|---|
