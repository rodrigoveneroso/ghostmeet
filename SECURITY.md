# Security

ghostmeet records private meetings, so the bar is: nothing leaves the machine, and
nothing on the machine other than the extension can read what it recorded.

## What protects a recording

| Threat | Defence | Tested in |
|---|---|---|
| Another device on the network | The backend binds `127.0.0.1` only (Docker publishes on loopback only) | `test_security.py` |
| A web page reading the API | CORS allows the extension and loopback origins only, never `*` | `test_security.py` |
| A web page opening a WebSocket (CORS does not apply) | Origins other than the extension's and loopback are refused, so no page can listen to a live transcript or stream into a session | `test_security.py` |
| A web page sending a "simple" POST, which skips the CORS preflight | Request bodies must be `application/json`, which forces the preflight | `test_security.py` |
| DNS rebinding, a hostile domain resolving to `127.0.0.1` | Requests must address `127.0.0.1` or `localhost` (`GHOSTMEET_ALLOWED_HOSTS` to add more) | `test_security.py` |
| A session id escaping the data folder | Session ids are `[A-Za-z0-9_-]{1,64}` | `test_security.py` |
| The backend sending data out | It has no HTTP or socket client; a test fails if a backend module imports one | `test_security.py` |
| Audio lingering on disk | Only the transcript is stored; the working audio is deleted when a session ends (after the optional second pass) | `test_app.py` |
| Meeting content being executed | `GHOSTMEET_ON_EXPORT` gets the meeting's details in environment variables, never in its command line | `hooks.py` |
| Model output injected into the page | The side panel only ever sets `textContent` | — |

The API has no authentication: anything that runs as your user on your machine can
read your meetings, as it could read any file of yours.

## Reporting a vulnerability

Please do not open a public issue. Use GitHub's private vulnerability reporting on this
repository (Security → Report a vulnerability), with steps to reproduce.
