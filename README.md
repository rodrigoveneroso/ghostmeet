<p align="center">
  <img src="assets/logo.jpg" width="200" alt="ghostmeet logo">
</p>

# ghostmeet

**Your invisible meeting recorder** — Live captions and local transcripts, right in your browser.

[Quick Start](#quick-start) • [How It Works](#how-it-works) • [Features](#features) • [API](#api)

> **About this fork.** This is a fork of [Higangssh/ghostmeet](https://github.com/Higangssh/ghostmeet)
> focused on the v0.1 Chrome extension + Python backend in [`legacy/`](legacy/), kept in
> sync with upstream. It is tuned for everyday use with Google Meet and plain-text
> transcripts you can hand to any AI agent:
>
> - **Your voice too.** The meeting tab only carries the other participants, so the
>   microphone is mixed into the recording (asked once, on an extension page).
> - **Who said what, on Google Meet.** Each exported line is credited to the participant
>   whose audio indicator was lit while it was said (`[02:26] Ana: …`). Lines where that
>   is unclear stay without a name rather than being guessed.
> - **Follows the call.** Optionally, recording starts when you join a Google Meet call
>   and stops when you leave it (one click on the extension per meeting tab, which Chrome
>   requires before any tab can be captured).
> - **Named after the meeting.** A scheduled meeting's name and the Meet code go into the
>   session and the exported transcript.
> - **No junk sessions.** Captures shorter than 30 s or without any speech are dropped
>   when they stop (`GHOSTMEET_MIN_DURATION_SEC`).
> - **Survives a backend restart.** If the backend goes away mid-meeting, the extension
>   keeps recording into a queue, reconnects on its own and the session continues on the
>   same timeline.
> - **No audio is kept.** Only the transcript is stored; the working PCM file is deleted at
>   the end of every session.
> - **Markdown export.** Every finished meeting can be written to a folder as a `.md` file
>   (`GHOSTMEET_EXPORT_DIR`), ready to be read by Claude Code, Codex or any other agent.
> - **Out of the way.** The side panel only opens when you ask for it; the popup shows
>   whether it is recording, for how long and how much is transcribed, then how the
>   transcription ended. No Summarize, Clear or language buttons: summarise the exported
>   markdown with the agent of your choice, and set the meeting language once in `.env`
>   (`GHOSTMEET_LANGUAGE`, e.g. `pt`).
> - **In Portuguese.** The interface comes from `extension/_locales/` (Brazilian
>   Portuguese today; a language is one more `messages.json`).
> - **More reliable capture.** The capture state survives Chrome putting the service
>   worker to sleep during long meetings, Stop always stops the recorder, and a recorder
>   left over from a previous session no longer corrupts the next one.
> - **`start.sh` / `stop.sh`** to run the backend in the background, and
>   `GHOSTMEET_DATA_DIR` to keep the database outside the checkout.
> - **New icon**, the 👻 from the side panel instead of the old logo.
> - **Tested in a real browser.** Besides the unit tests, Playwright scripts check the
>   microphone, a backend restart mid-meeting, following a Meet call and the popup (see
>   [Development](#development)).
>
> Summaries are still in the backend (`POST /api/sessions/{id}/summarize`, with
> `GHOSTMEET_ANTHROPIC_KEY`), but the extension no longer has a button for them.

> **Renewing as MeetButler.** ghostmeet is becoming MeetButler, a desktop app for Windows
> and macOS that records in-person meetings as well as web calls, fully on your machine.
> It is under development in `app/` and `crates/` and not ready to use yet. The Chrome
> extension and Python backend described below are v0.1, now in [`legacy/`](legacy/), and
> still work as documented.

---

<p align="center">
  <img src="assets/demo.gif" alt="ghostmeet demo" width="800">
</p>

## What is ghostmeet?

ghostmeet silently captures audio from any browser tab — Google Meet, Zoom, Teams, or anything with sound — together with your microphone, and transcribes it in real-time using Whisper. When the meeting ends you have a transcript, as markdown if you like, ready for the AI agent of your choice to summarise.

It runs as a **Chrome extension**: a popup to start and stop, and a side panel with live captions only if you open it. Other participants can't see it. Like a ghost in your meeting. 👻

- **100% local** — audio never leaves your machine
- **No accounts** — no sign-up, no cloud, no subscriptions
- **Works everywhere** — any tab that plays audio

## Features

- 🎙️ **Real-time transcription** — Whisper STT, updates every 10 seconds
- ⏱️ **Built for long meetings** — cost per pass stays flat, so a 4-hour session behaves like a 4-minute one
- 🗣️ **Who said what** — on Google Meet, each line is credited to the participant speaking
- 💾 **Nothing is lost** — transcripts are written to SQLite as they happen and survive a restart
- 📝 **Markdown export** — one `.md` per meeting, named after it when it was scheduled
- 🌏 **Your language** — set it once with `GHOSTMEET_LANGUAGE`, or let Whisper detect it
- 🔒 **Self-hosted** — your audio stays on your machine, and the server listens on loopback only
- 🐳 **One-command setup** — `docker compose up` and you're ready
- 👻 **Invisible** — nothing opens on its own, no one in the meeting knows

## How It Works

```
Browser Tab (Zoom / Meet / Teams)          Microphone
    │ audio                                     │
    ▼                                           ▼
Chrome Extension
    ├── service worker  — asks Chrome for a tab stream id, follows the call
    ├── offscreen page  — mixes tab + mic, records, reconnects if the backend drops
    └── Meet scripts    — who is speaking, the meeting's name, in call or not
    │
    ▼  WebSocket (webm/opus, 1s chunks)
Local Backend (FastAPI)
    ├── one demuxer per session  ──→ PCM appended to disk
    ├── Whisper reads only the newest window, never the whole recording
    ├── segments ──→ SQLite  +  live captions in the side panel
    └── on stop ──→ speaker names ──→ markdown export (or dropped if too short)
```

Everything runs on your machine. Nothing is sent anywhere, unless you call the backend's
optional summary endpoint, which uses the Claude API.

Audio is decoded once as it arrives and kept on disk, and each transcription pass reads
only a bounded window of it. That is what keeps a long meeting from getting slower and
slower, and keeps memory flat no matter how long you record.

## Quick Start

### Prerequisites

- **Docker** (recommended) or Python 3.10+
- **Chrome** browser

### 1) Start the backend

```bash
git clone https://github.com/rodrigoveneroso/ghostmeet.git
cd ghostmeet/legacy

# Copy and edit config (meeting language, export folder, Whisper model)
cp .env.example .env

# Start with Docker
docker compose up -d
```

Backend is ready when you see `http://0.0.0.0:8877` in the logs. To get the markdown
export out of the container, mount a folder and point `GHOSTMEET_EXPORT_DIR` at it in
`docker-compose.yml`; `./start.sh` (manual install) reads `.env` instead.

<details>
<summary><strong>Manual install (without Docker)</strong></summary>

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
python -m backend
```

Note: First run downloads the Whisper model (~150MB for `base`).
</details>

### 2) Install Chrome Extension

1. Open **chrome://extensions** in Chrome
2. Enable **Developer mode** (toggle in top-right)
3. Click **Load unpacked**
4. Select the `legacy/extension/` folder from this repo
5. Pin the 👻 icon in your toolbar

### 3) Use it

1. **Join a meeting** — Open Google Meet, Zoom, Teams (or any tab with audio)
2. **Click 👻** — The popup says whether it is recording
3. **Click ▶ Iniciar gravação** — Recording starts quietly: a red dot on the icon is the
   only sign, and the tab stays audible. **Ver transcrição ao vivo** opens the side panel
   with live captions if you want to watch them.
   The first time, a tab asks for your microphone so your own voice is transcribed too.
   Headphones keep the others from being picked up a second time through your speakers.
   To have recording follow the call instead, tick **Start and stop with the call** in the
   popup: from then on, opening the popup once on a Meet tab (lobby or call) is enough;
   if you join without it, the icon shows **REC** to ask for that click.
4. **Click ■ Parar gravação** — The popup says the transcription is being finished, then
   shows its length and how many segments it has
5. **Read the transcript** — in the side panel, through the API, or as markdown in
   `GHOSTMEET_EXPORT_DIR` if you set it. Summarise it with the AI agent of your choice.

Reopening the side panel mid-meeting brings the transcript so far back with it.

That's it. No sign-up, no config, no cloud.

## Configuration

Set these in `.env` or `docker-compose.yml`:

| Variable | Default | Description |
|----------|---------|-------------|
| `GHOSTMEET_MODEL` | `base` | Whisper model size (`tiny` / `base` / `small` / `medium` / `large`) |
| `GHOSTMEET_DEVICE` | `auto` | Compute device (`auto` / `cpu` / `cuda`) |
| `GHOSTMEET_COMPUTE_TYPE` | `float32` | Precision (`int8` is much faster on CPU) |
| `GHOSTMEET_LANGUAGE` | auto-detect | Language of your meetings (`pt` / `en` / `ko` / etc.). There is no picker in the side panel |
| `GHOSTMEET_EXPORT_DIR` | off | Folder that receives a markdown copy of every finished transcript |
| `GHOSTMEET_MIN_DURATION_SEC` | `30` | Captures shorter than this, or without speech, are dropped when they stop (`0` keeps everything) |
| `GHOSTMEET_DATA_DIR` | `legacy/recordings` | Where the SQLite database and per-session files live |
| `GHOSTMEET_CHUNK_INTERVAL` | `10` | Seconds between transcription updates |
| `GHOSTMEET_ANTHROPIC_KEY` | — | Only for the optional summary endpoint |
| `GHOSTMEET_HOST` | `127.0.0.1` | Server bind address (loopback — the API has no auth) |
| `GHOSTMEET_PORT` | `8877` | Server port |

**Model size guide:**
- `tiny` — fastest, least accurate (~75MB)
- `base` — good balance (recommended, ~150MB)
- `small` — better accuracy, slower (~500MB)
- `medium` / `large` — best accuracy, needs GPU

## API

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/health` | GET | Health check + model info |
| `/api/sessions` | GET | List all sessions |
| `/api/sessions/{id}` | GET | Session details |
| `/api/sessions/{id}/transcript` | GET | Full transcript |
| `/api/sessions/{id}/summarize` | POST | Generate AI summary |
| `/api/sessions/{id}/summary` | GET | Get generated summary |
| `/api/sessions/{id}/meeting` | POST | Meeting name and code from the meeting page (`{"title", "code"}`) |
| `/api/sessions/{id}/speaker-events` | POST | Google Meet participant UI changes, used to name speakers |
| `/ws/audio` | WS | Audio ingest (binary chunks; send `stop` as text to finish; `resume=<epoch ms>` continues a dropped session) |
| `/ws/transcript/{id}` | WS | Live transcript stream |

Sessions, transcripts and summaries are stored in `recordings/ghostmeet.db` (or under
`GHOSTMEET_DATA_DIR`), so every endpoint above keeps working after the backend restarts.

## Development

MeetButler (Rust, from the repository root):

```bash
cargo test --workspace                               # no model, no network
```

v0.1 (from `legacy/`):

```bash
cd legacy
python -m venv .venv
./.venv/Scripts/python.exe -m pip install -r requirements-dev.txt

./.venv/Scripts/python.exe -m pytest tests/ -q       # backend suite
node --test tests/extension/shared.test.mjs          # extension helpers
```

The suite needs no Whisper model and no network — real opus audio goes through the real
decoder, and only inference is stubbed.

There is one thing tests cannot reach: `chrome.tabCapture.getMediaStreamId()` needs the
`activeTab` grant that only a real toolbar click produces. Everything after that point is
covered by `node tests/extension/verify-capture.mjs`, which drives the extension in a real
browser against a running backend (needs `npm install playwright && npx playwright install
chromium`). `node tests/extension/verify-mic-and-reconnect.mjs` starts its own backend on
a throwaway data folder and checks, with Chromium's fake microphone, that your voice
reaches the recording and that a capture survives the backend restarting mid-meeting.
`node tests/extension/verify-follow-call.mjs` serves a stand-in Meet page and checks that
joining and leaving are noticed, that the session takes the meeting's name and that a
short capture is dropped. `node tests/extension/verify-popup.mjs` drives the popup through
a whole capture. These two need port 8877 free: stop your own backend while they run.
To check the last step by hand: start a capture on a tab with audio and confirm
`audio_bytes` climbs in `/api/sessions` — and that you can still hear the tab.

## Project Structure

```
ghostmeet/
├── app/                    # MeetButler desktop app (Tauri 2), in development
├── crates/                 # MeetButler engine and audio capture (Rust)
├── bench/                  # Korean STT engine benchmarks
└── legacy/                 # v0.1, laid out as below

legacy/
├── extension/              # Chrome MV3 Extension
│   ├── manifest.json       # permissions + side panel config
│   ├── background.js       # service worker: gets a tab stream id, drives capture
│   ├── offscreen.html/js   # hidden page that actually records → WebSocket
│   ├── sidepanel.html/js   # live captions
│   ├── meet-dom.js         # pure readers of the Meet page (in call? name? code?)
│   ├── meet-call.js        # Meet content script: join/leave and the meeting's name
│   ├── meet-speakers.js    # Meet content script: participant audio indicators
│   ├── mic-permission.*    # the page where the microphone is granted once
│   ├── popup.*             # what the capture is doing, and Start or Stop
│   ├── i18n.js, _locales/  # interface text (pt_BR)
│   ├── shared.js           # pure helpers shared by the above
│   └── icons/              # drawn by tools/make_icons.py (Noto Color Emoji, Apache-2.0)
├── backend/                # Python backend (FastAPI)
│   ├── app.py              # HTTP + WebSocket server
│   ├── decoder.py          # streaming webm/opus → PCM (one demuxer per session)
│   ├── pcm_store.py        # append-only audio on disk, windowed reads
│   ├── incremental.py      # bounded-window transcription, absolute timestamps
│   ├── pipeline.py         # receive / decode / transcribe, decoupled
│   ├── transcriber.py      # shared Whisper model + per-session language
│   ├── store.py            # SQLite: sessions, segments, summaries
│   ├── exporter.py         # markdown copy of finished transcripts
│   ├── speakers.py         # names each segment from the Meet audio indicators
│   ├── retention.py        # which finished captures are worth keeping
│   ├── summarizer.py       # Claude API integration
│   └── models.py           # session model
├── tests/                  # pytest suite + extension tests
├── docker-compose.yml      # one-command deployment
├── Dockerfile              # backend container
└── requirements.txt        # Python dependencies
```

## OpenClaw Integration

ghostmeet works as an [OpenClaw](https://github.com/openclaw/openclaw) skill. Control your meetings from chat.

```bash
# Install the skill
clawhub install ghostmeet
```

Then just ask your AI assistant:

- **"Summarize my last meeting"** → generates AI summary from latest session
- **"How many meetings did I have today?"** → lists all sessions
- **"What was discussed?"** → fetches full transcript
- **"Extract action items"** → pulls tasks from the summary

> The skill handles session listing, transcript retrieval, and summary generation via the ghostmeet API. Recording start/stop is done through the Chrome Extension.

## Roadmap

- [x] Real-time transcription (Whisper)
- [x] Chrome Extension side panel UI
- [x] AI meeting summaries (Claude)
- [x] Long meetings — flat cost per pass, tested to 4h+ of audio
- [x] Transcripts survive a restart (SQLite)
- [x] Per-session language (`GHOSTMEET_LANGUAGE`, or `lang=` on `/ws/audio`)
- [x] Your own voice in web calls (microphone mixed with the tab)
- [x] Recording follows a Google Meet call
- [ ] Meeting context input + file attach
- [x] Speaker names on Google Meet (from the meeting's own audio indicators)
- [ ] Speaker diarization from the audio itself (other meeting apps)
- [ ] In-person meetings (microphone capture)
- [x] Export to Markdown
- [ ] Agent Mode — AI speaks in the meeting for you

## License

[MIT](LICENSE)
