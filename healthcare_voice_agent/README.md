# Healthcare Voice Contact Center (Pipecat + Cartesia + Anthropic)

> A real-time voice AI contact center for a healthcare clinic. **Cartesia** for streaming STT/TTS, **Anthropic Claude** for the LLM with tool calling, orchestrated with **Pipecat**.

The agent handles patient appointment booking, answers FAQ on insurance, billing, and clinic hours, and escalates emergencies or complaints to a "Supervisor" personality (different voice + system prompt) — all on a single Pipecat pipeline.

## Features

- **Streaming voice loop** — Cartesia `ink-whisper` STT and `sonic-3` TTS with Anthropic `claude-haiku-5-5`
- **Function calling** — `check_availability`, `book_appointment`, `lookup_faq`, `escalate_to_human` registered in-process
- **Mock backend** — JSON-backed slot store and FAQ knowledge base under `data/`
- **Personality switching** — On escalation, the TTS voice swaps to the Supervisor and a new system prompt is injected mid-session via `TTSUpdateSettingsFrame`
- **Two transports** — Local **WebRTC** test client (default) or **Daily** rooms
- **Human bridge stub** — Escalations are logged with a ticket ID; a real deployment would bridge to a live operator

## Tech stack

- **Python 3.11+**
- **[Pipecat](https://docs.pipecat.ai)** (`pipecat-ai` with `cartesia`, `anthropic`, `runner`, `webrtc`, `daily`, `silero` extras)
- **[Cartesia](https://cartesia.ai)** — `ink-whisper` STT, `sonic-3` TTS with multiple voice IDs
- **[Anthropic](https://docs.anthropic.com)** — Claude with native tool use
- **FastAPI + uvicorn** — Served by the Pipecat runner (default port **7860**)

## How it works

```
caller audio ──► Cartesia STT ──► user context ──► Claude LLM ──► Cartesia TTS ──► caller audio
                                                       │
                                          tool calls ──┤── check_availability
                                                       ├── book_appointment
                                                       ├── lookup_faq
                                                       └── escalate_to_human  ──► swap voice + supervisor prompt
```

1. Audio arrives through the runner's transport.
2. **CartesiaSTTService** transcribes streaming speech.
3. **AnthropicLLMService** drives the conversation, calling tools when it needs slots, FAQ answers, or escalation.
4. On `escalate_to_human`, a `TTSUpdateSettingsFrame` swaps the Cartesia voice and a Supervisor system prompt is appended to the context.
5. **CartesiaTTSService** speaks the reply back through the transport.

## Assumptions

This is a tutorial/demo project, not a production system:

- All patient, appointment, and FAQ data is mock data in `data/`. There is no real
  patient data, no database, and no PHI.
- There is no authentication, encryption at rest, or HIPAA-grade handling anywhere
  in the pipeline. Do not point this at real patient information.
- `escalate_to_human` only logs a ticket ID — there is no real human bridge.

## Getting started

### Prerequisites

- Python **3.11** or newer
- API keys:
  - [Cartesia](https://cartesia.ai) — `CARTESIA_API_KEY`
  - [Anthropic](https://console.anthropic.com) — `ANTHROPIC_API_KEY`
- For **Daily** transport only: [Daily](https://www.daily.co/) — `DAILY_API_KEY` (optional `DAILY_ROOM_URL`)

### Environment variables

Copy `.env.example` to `.env` in this directory and fill in your keys:

```env
CARTESIA_API_KEY=your_cartesia_api_key
ANTHROPIC_API_KEY=your_anthropic_api_key

# Only if you use Daily (-t daily)
# DAILY_API_KEY=your_daily_api_key
# DAILY_ROOM_URL=https://your-domain.daily.co/your-room
```

### Install and run

```bash
cd voice_agents/healthcare_contact_center
uv sync
```

**WebRTC (default)** — opens a local server and a browser test client:

```bash
uv run python main.py
# or explicitly:
uv run python main.py -t webrtc
```

Open **http://localhost:7860/client** (the runner prints the URL).

**Daily** — bot joins a Daily room (set `DAILY_API_KEY` first):

```bash
uv run python main.py -t daily
```

## Try it

Talk to the agent and try each capability:

- **Book**: *"I'd like to book an appointment with Dr. Patel on April 22."*
- **FAQ**: *"What insurance plans do you accept?"*
- **Escalation (normal)**: *"I want to speak to a manager."*
- **Escalation (emergency)**: *"I'm having chest pain right now."* → voice should switch to the Supervisor.

## Project layout

| File / Dir | Role |
|---|---|
| `main.py` | Pipeline, transport, tool registration, event handlers |
| `personalities.py` | Front-desk and supervisor prompts + Cartesia voice IDs |
| `tools/appointments.py` | `check_availability`, `book_appointment` against `data/slots.json` |
| `tools/knowledge_base.py` | `lookup_faq` keyword match against `data/faq.json` |
| `tools/escalation.py` | `escalate_to_human` — logs a ticket (stub for human bridge) |
| `data/slots.json` | Mock appointment slot store |
| `data/faq.json` | Embedded clinic policies and FAQ |
| `pyproject.toml` | Dependencies and Python version |

## Customizing

- **Voices** — edit `FRONT_DESK_VOICE` / `SUPERVISOR_VOICE` in `personalities.py` with any Cartesia voice ID.
- **LLM model** — change the `model` in `AnthropicLLMService.Settings` (e.g., `claude-sonnet-5-5` for higher quality, `claude-opus-5-5` for the most capable).
- **Knowledge base** — extend `data/faq.json` with more topics; the matcher is keyword-based and easy to swap for embeddings later.
- **Real backend** — replace the JSON file ops in `tools/appointments.py` with calls to your scheduling REST API.
- **Real escalation** — replace the log stub in `tools/escalation.py` with a Daily/Twilio call-bridge or a paging webhook.

## Learn more

- [Pipecat documentation](https://docs.pipecat.ai)
- [Pipecat function calling guide](https://docs.pipecat.ai/pipecat/learn/function-calling)
- [Cartesia voices](https://docs.cartesia.ai)
- [Anthropic API documentation](https://docs.anthropic.com)
