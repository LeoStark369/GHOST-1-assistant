# GHOST-1
**Global Heuristic Operating System Terminal-1** — a futuristic, holographic-HUD desktop AI assistant.

Created by **Surafel Gashaw** (*Leo Stark*).

This repository is a **working architectural foundation**, not a polished, fully
QA'd shrink-wrapped app. Every module is real, runnable code — but a system this
large (voice, PC automation, multi-agent LLMs, custom Qt rendering) needs to be
run, tested, and tuned on your actual machine with a real microphone and a real
API key. Treat this as v0.1: the skeleton is solid and every seam is documented;
expect to iterate.

## Quick start

```bash
python -m venv venv
source venv/bin/activate        # venv\Scripts\activate on Windows
pip install -r requirements.txt

cp .env.example .env
# edit .env and add your Gemini API key: https://aistudio.google.com/apikey

python main.py
```

Windows-only extras used by `system/pc_control.py` (install if you want volume
and brightness control):
```bash
pip install pycaw comtypes screen-brightness-control playsound
```

## Architecture

```
main.py                  Entry point (QApplication + main window)
config.py                Loads .env, central Settings object, masks secrets in logs

core/
  gemini_agents.py        GeminiCore / GeminiResearch / GeminiCommand (3 isolated agents)
  orchestrator.py          Routes input -> agent -> (confirmation) -> execution
  memory.py                SQLite conversation + command audit log (+ optional FAISS)

system/
  pc_control.py            Real OS actions: apps, files, volume, brightness, wifi, bluetooth
  monitor.py                CPU/RAM/GPU/network polling for the HUD

voice/
  speech_engine.py          Wake-word ("Hello GHOST") + faster-whisper transcription
  tts_engine.py              edge-tts synthesis + playback

research/
  web_research.py           DuckDuckGo search -> fetch -> GeminiResearch synthesis + citations

ui/
  themes.py                 Dark/light futuristic palettes + QSS
  main_window.py             Assembles the full HUD, wires voice + orchestrator
  widgets/
    ai_core_widget.py         Animated rotating AI core (QPainter, no image assets)
    waveform_widget.py         Real-time-look audio spectrum bars
    particle_background.py     Drifting connected-particle field
    system_monitor_widget.py    CPU/RAM/GPU/network gauges
    side_widgets.py             Clock, weather, toast notifications, command history log
```

## Multi-agent design

- **GeminiCore** — conversation, reasoning, and a cheap intent classifier that decides
  whether a message should go to Core / Research / Command.
- **GeminiResearch** — takes search results + fetched page text and produces a
  cited, original-wording report (never copies large verbatim source text).
- **GeminiCommand** — turns natural language into one structured action
  (`open_app`, `set_volume`, `toggle_wifi`, etc.) and flags whether it needs
  user confirmation. `Orchestrator` enforces that confirmation gate before
  `PCController` ever touches the real system — Command never executes
  anything itself.

Each agent can use its own API key (billing/rate-limit isolation) and its own
model tier via `.env` — by default all three fall back to `GEMINI_API_KEY`.

## Known limitations / next steps

- **Voice pipeline** needs a real mic to test; `faster-whisper` model files
  download on first run (requires network). The wake-word detector currently
  re-transcribes short clips rather than using a dedicated lightweight KWS
  model — swap in Porcupine/OpenWakeWord for lower latency and CPU use.
- **Bluetooth/Wi-Fi toggles on Windows** require admin privileges and, for
  Bluetooth, PowerShell's `PnpDevice` cmdlets — test in an elevated terminal.
- **Text-to-speech playback** on Windows needs `playsound` (or swap in
  `pygame.mixer` / `sounddevice` for more control over interrupting speech).
- **UI calls the orchestrator synchronously** on the Qt main thread for
  clarity; move `Orchestrator.handle_user_input` onto a `QThread`/worker
  before shipping, so slow API/network calls never freeze the HUD.
- **FAISS semantic memory** is stubbed (index created, but nothing embeds
  into it yet) — wire in a sentence-embedding call (e.g. a small local model,
  or Gemini's embedding endpoint) to make old turns semantically searchable.
- **Automations** are deliberately whitelist-only (`automations/<name>.py`)
  rather than arbitrary code execution, for safety — add new scripts there.

## Security notes

- API keys live only in `.env` (gitignored), loaded once via `config.py`, and
  are masked (`ABCD...WXYZ`) in every log line — never printed in full.
- Every command GHOST-1 executes is written to the `commands` table in
  `data/ghost_memory.db` (action, args, whether it was confirmed, result) —
  a full audit trail of everything GHOST-1 has done to your PC.
- Anything the Command agent marks sensitive (closing apps, toggling
  Wi-Fi/Bluetooth, running automations) is held for an explicit yes/no before
  `PCController` runs it.

## Author

**Surafel Gashaw** (*Leo Stark*) — designer and developer of GHOST-1.
