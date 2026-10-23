# 👻 ghostmeet

[![legacy tests](https://github.com/rodrigoveneroso/ghostmeet/actions/workflows/legacy.yml/badge.svg)](https://github.com/rodrigoveneroso/ghostmeet/actions/workflows/legacy.yml)

**A meeting recorder that never leaves your machine.** A Chrome extension captures the
call and your microphone, a local Whisper transcribes it live, and every meeting ends as
a markdown transcript with who said what, ready for the AI agent of your choice.

No bot joins the call, no account, no cloud, no API key.

> This is a fork of [Higangssh/ghostmeet](https://github.com/Higangssh/ghostmeet) that
> develops its v0.1 (the Chrome extension and the Python backend, in [`legacy/`](legacy/))
> into a daily tool. Upstream is turning into a desktop app, MeetButler, in `app/` and
> `crates/`, which this fork leaves untouched.

## What it does

**Recording**
- Captures any browser tab, Google Meet first, **plus your microphone**: a meeting tab
  never plays your own voice back.
- **No headphones needed.** An echo suppressor uses the tab as its reference and removes
  the others' voices picked up again from your speakers.
- **Invisible.** Nothing opens on its own; a red dot on the icon is the only sign. The
  live captions are one click away, and say who is speaking.
- **Light while you meet.** With the live captions closed, Whisper runs in a batch every
  five minutes instead of every ten seconds, at low priority: measured, the backend went
  from 89 s of CPU during a 90 s meeting to under half a second. Opening the captions
  catches up at once.
- **Follows the call** (optional): starts when you join a Meet call, stops when you leave.
  Muted in Meet means nothing of yours is recorded.
- **Survives a backend restart** mid-meeting: the extension queues audio and reconnects.

**Transcript**
- **Who said what** on Google Meet, from the meeting's own audio indicators. A line whose
  speaker is unclear keeps no name rather than a wrong one.
- **Your vocabulary**, plus the participants' names, handed to Whisper so they come out
  spelled right.
- **Marks and notes** taken during the meeting (popup, or Alt+Shift+M) land in the
  transcript where you took them.
- **Named after the meeting** when it was scheduled.
- **Tidied on export:** words repeated where two transcription windows meet, and the
  subtitle credits Whisper invents in silence ("Legendas pela comunidade Amara.org"),
  are removed.
- Optional **second pass** with a larger model once the meeting is over: with
  `large-v3-turbo`, less than half the errors of the live transcript.

**Afterwards**
- One **markdown file per meeting**, and a command of yours run on it
  (`GHOSTMEET_ON_EXPORT`), e.g. to have your agent summarise it.
- **No junk:** captures under 30 s or without speech are dropped.
- **No audio is kept**, only the text.

## A transcript

What lands in your export folder (made-up meeting):

```markdown
# Weekly sync (05/01/2026 10:00)

- sessão: 20260105-100002
- duração: 32:10
- idioma: pt
- status: stopped
- meet: abc-defg-hij
- falantes: estimados pelo indicador de áudio do Google Meet (aproximado)

## Transcrição

[00:04] Ana Souza: Bom dia, vamos começar pelos números do mês.
[00:41] Bruno Lima: O painel do Metabase está atrasado desde segunda.
[01:02] 📌 Momento marcado
[01:05] Ana Souza: Então a decisão é corrigir a integração antes de sexta.
[01:12] Combinado.
[01:15] 📝 Nota: Bruno fica com a integração
```

The interface and the transcript headings are in Brazilian Portuguese; the interface
text lives in `legacy/extension/_locales/`, so another language is one more file.

## Install

You need Chrome, Python 3.10+ (or [uv](https://docs.astral.sh/uv/), which brings its own)
and Linux, where everything here is tested. macOS should work through `./start.sh`, but
gets no autostart and has not been tried.

```bash
git clone https://github.com/rodrigoveneroso/ghostmeet.git
cd ghostmeet/legacy
./install.sh
```

It asks four questions (meeting language, Whisper model, a folder for the transcripts,
an optional vocabulary file), sets up a virtualenv, downloads the model and, on Linux,
installs a systemd user service that starts the backend at every login and restarts it
if it dies. No sudo. Run it again any time: it only does what is missing.

Then load the extension: `chrome://extensions` → Developer mode → **Load unpacked** →
`legacy/extension`, and pin the 👻.

When something does not work, `./doctor.sh` says what and how to fix it:

```text
ghostmeet doctor
  ✓ virtualenv with the dependencies (Python 3.12.15)
  ✓ Whisper model 'small' is downloaded
  ✓ the service starts the backend at login, and it is running
  ✓ backend answering on 127.0.0.1:8877, Whisper loaded
```

`./start.sh` and `./stop.sh` start and stop the backend by hand. A `docker compose up -d`
setup is in `legacy/` too, without the service and not covered by the CI.

## Use

1. In the meeting tab, click 👻 → **▶ Iniciar gravação**. The first time, a tab asks for
   your microphone.
2. While it records, the popup shows for how long and how much is transcribed, and lets
   you **📌 mark the moment** or type a note. **Alt+Shift+M** marks without opening it.
3. **■ Parar gravação.** The popup says when the transcript is done, and the markdown is
   in your export folder.

To have recording follow the call, tick **Iniciar e parar junto com a chamada**. Chrome
lets an extension capture a tab only after you clicked it there, so open the popup once
per meeting tab (the lobby is fine); if you join without it, the icon shows **REC**.

**With an AI agent.** Point it at the export folder, or let ghostmeet call it when each
transcript is written:

```bash
# in legacy/.env (single quotes: the file is read by a shell)
GHOSTMEET_ON_EXPORT='claude -p "Summarise $GHOSTMEET_TRANSCRIPT" > "${GHOSTMEET_TRANSCRIPT%.md}.summary.md"'
```

`legacy/skill/SKILL.md` is an [OpenClaw](https://github.com/openclaw/openclaw) skill that
reads sessions, transcripts and notes through the local API.

## Configuration

`legacy/.env`, written by the installer and documented in `legacy/.env.example`:

| Variable | Default | |
|---|---|---|
| `GHOSTMEET_LANGUAGE` | detect | Language of your meetings (`pt`, `en`, …) |
| `GHOSTMEET_MODEL` | `base` | Whisper model for the live transcript (`base`, `small`, …) |
| `GHOSTMEET_EXPORT_DIR` | off | Folder for the markdown transcripts |
| `GHOSTMEET_VOCABULARY` | — | Names and terms to expect, comma separated |
| `GHOSTMEET_VOCABULARY_FILE` | — | The same, one per line, re-read every meeting |
| `GHOSTMEET_ON_EXPORT` | — | Command run after each export (`GHOSTMEET_TRANSCRIPT`, `_SESSION_ID`, `_TITLE`, `_DURATION_SEC` in its environment) |
| `GHOSTMEET_FINAL_MODEL` | off | Larger model that transcribes the meeting again after Stop (`large-v3-turbo` recommended) |
| `GHOSTMEET_FINAL_BEAM` | `1` | Beam size of that pass (turbo skipped sentences with 5) |
| `GHOSTMEET_MIN_DURATION_SEC` | `30` | Shorter captures, or silent ones, are dropped (`0` keeps all) |
| `GHOSTMEET_COMPUTE_TYPE` | `auto` | Precision; `auto` picks the fastest the hardware has (int8 on a CPU) |
| `GHOSTMEET_DEVICE` | `auto` | `cpu` or `cuda` |
| `GHOSTMEET_PRELOAD` | `1` | Load Whisper at startup, so the first capture does not wait |
| `GHOSTMEET_CHUNK_INTERVAL` | `10` | Seconds between passes while the live captions are open |
| `GHOSTMEET_IDLE_INTERVAL` | `300` | Seconds between batches while they are closed (`0`: always live) |
| `GHOSTMEET_NICE` | `10` | CPU priority of the backend, so the call comes first (`0`: normal) |
| `GHOSTMEET_DATA_DIR` | `legacy/recordings` | Database and per-session files |
| `GHOSTMEET_HOST`, `GHOSTMEET_PORT` | `127.0.0.1`, `8877` | Where the backend listens |
| `GHOSTMEET_ALLOWED_HOSTS` | — | Extra host names accepted, only for a deliberate exposure |

## How it works

```
Meet tab ──┐                                   ┌─ who is speaking, meeting name,
Microphone ┴─ echo suppressor ─┐               │  participants, in call or muted
                               ▼               │      (Meet content scripts)
            Chrome extension (offscreen recorder) ──┘
                               │ WebSocket, webm/opus, 1 s chunks (queued if the backend drops)
                               ▼
            Local backend (FastAPI, 127.0.0.1)
              ├─ PCM appended to disk, Whisper reads only the newest window
              │  (every 10 s with the captions open, in 5-minute batches otherwise)
              ├─ segments, named by speaker → SQLite + live captions
              └─ on Stop → speaker names, notes → markdown → your command
                           (audio deleted; optional second pass first)
```

Each live pass reads a bounded window of the recording, so a four-hour meeting costs the
same per pass as the first minute. The backend has no HTTP client at all: nothing it
handles can leave the machine.

## Performance

`legacy/tools/bench_whisper.py` on an AMD Ryzen 7 5825U (CPU only), 46 s of Portuguese
meeting speech from a local TTS, through the backend's own settings:

| Model | Precision | Time for 46 s | Word error rate |
|---|---|---|---|
| `base` | int8 (`auto`) | 3.7 s | 12.6% |
| `small` | int8 (`auto`) | 10.1 s | 12.6% |
| `small` + vocabulary | int8 (`auto`) | 10.8 s | 11.2% |
| `small` | float32 (upstream's default) | 18.1 s | 13.3% |
| `medium` | int8 (`auto`) | 24 s | 11.9% |
| `large-v3-turbo`, beam 1 | int8 (`auto`) | 25 s | 7.7% |
| `large-v3-turbo`, beam 5 | int8 (`auto`) | 26 s | 32.9% (skipped three sentences) |

`small` keeps up live with room to spare; `large-v3-turbo` is too slow for that on a CPU
but makes the best second pass: through the backend's own second-pass path, with the
vocabulary and the export cleanup, it got the same speech down to **5.6%** errors, about
half an hour of work for an hour of meeting. Run the tool on your own machine to choose.

## Privacy and security

Only the extension can talk to the backend: it listens on loopback, refuses web pages'
origins (WebSockets included), DNS rebinding and undeclared request bodies, and keeps
session files inside its data folder. Each of these has a test. The threat model, and how
to report a vulnerability, are in [SECURITY.md](SECURITY.md).

## Development

```bash
cd legacy
../.venv/bin/python -m pip install -r requirements-dev.txt
../.venv/bin/python -m pytest tests/ -q            # backend, no model, no network
node --test tests/extension/shared.test.mjs tests/extension/meet-dom.test.mjs \
  tests/extension/echo-gate.test.mjs               # extension logic
```

The real-browser checks drive Chromium with the extension loaded against a real backend
(`npm install playwright ws && npx playwright install chromium` first; the last two need
port 8877 free):

| Script | Checks |
|---|---|
| `verify-capture.mjs` | the recording path (upstream's) |
| `verify-mic-and-reconnect.mjs` | the echo suppressor as an AudioWorklet, the microphone reaching the recording, a backend restart mid-meeting |
| `verify-follow-call.mjs` | joining and leaving a stand-in Meet page, the meeting's name and participants, muting, dropping a short capture |
| `verify-popup.mjs` | the popup through a whole capture, marks and notes |

GitHub Actions runs all of it on every change to `legacy/` ([legacy.yml](.github/workflows/legacy.yml)).

```
legacy/
├── extension/       Chrome MV3 extension: popup, offscreen recorder, echo gate,
│                    Meet content scripts, side panel, _locales/
├── backend/         FastAPI app, streaming decoder, windowed transcription, SQLite,
│                    speakers, vocabulary, export, second pass, on-export hook
├── tests/           pytest suite, extension unit tests, real-browser checks
├── tools/           Whisper benchmark, icon generator
└── install.sh  doctor.sh  run.sh  start.sh  stop.sh
```

## Credits

Built on [ghostmeet](https://github.com/Higangssh/ghostmeet) by Sanghee Son, which
gave it the extension, the streaming backend and the long-meeting design. Icons from
[Noto Color Emoji](https://github.com/googlefonts/noto-emoji) (Apache-2.0).

[MIT](LICENSE)
