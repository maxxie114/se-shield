# ATTACKER_AGENTS.md — Red-Team Scammer Caller (ElevenLabs)

## 0) Owner + Goal
**Owner:** Attacker engineer  
**Goal:** Build the “scammer caller” service that outputs the next caller line (text) and generates **ElevenLabs TTS audio** so the UI can play a realistic phone call.

This service should be deterministic and demo-safe. MVP is scripted (no LLM required).

---

## 1) MVP in < 1 hour (Attacker)
### MVP Objective
A tiny API that supports:
- Create or accept `session_id` (can be created by Defender; attacker only needs it)
- `next_caller` returns the next line from a pre-written script
- ElevenLabs TTS generates `audio_url` (or return raw audio bytes if easier)

### MVP Deliverables
- 1 scenario JSON with 6–10 caller lines
- Endpoint: `POST /api/v1/sessions/{session_id}/next_caller`
- In-memory session pointer: `turn_index` per session
- ElevenLabs integration for TTS

If ElevenLabs is slow/unreliable, MVP can return `audio_url: null` and still pass; but voice adds huge demo value.

---

## 2) Scenario Format (Canonical)
Store scenarios in `/scenarios/*.json`.

Example schema:
{
  "scenario_id": "ceo_impersonation_001",
  "title": "CEO Impersonation",
  "lines": [
    {"text":"Hi, this is the CEO. I need you to reset my MFA right now.", "tactics":["authority_impersonation","urgency_pressure"]},
    {"text":"I’m in a board meeting. If you can’t help, I’ll have your manager handle it.", "tactics":["threat_intimidation","time_pressure"]},
    {"text":"Look, I can verify later. Just do it, it’s critical.", "tactics":["identity_bypass","urgency_pressure"]}
  ]
}

---

## 3) API Contract (Attacker Service)
Base: `ATTACKER_BASE_URL`

### 3.1 Next Caller Turn
POST `/api/v1/sessions/{session_id}/next_caller`

Request:
{
  "scenario_id": "ceo_impersonation_001",
  "voice": {
    "provider": "elevenlabs",
    "voice_id": "<optional>"
  },
  "format": "mp3"
}

Response 200:
{
  "session_id": "sess_...",
  "scenario_id": "ceo_impersonation_001",
  "turn_index": 1,
  "caller_text": "string",
  "tactics_used": ["authority_impersonation","urgency_pressure"],
  "audio_url": "https://... or null"
}

Rules:
- `turn_index` increments per call.
- When out of lines: return 409 with:
  { "error": { "code":"SCENARIO_DONE", "message":"No more caller lines" } }

---

## 4) ElevenLabs Integration (MVP)
### Inputs
- `text` (caller_text)
- `voice_id` (env or request)
- output format mp3

### Outputs
- For fastest MVP:
  - return `audio_base64` (optional) OR `audio_url` if you can upload to S3 quickly.
- Recommended: generate audio and return a temporary URL:
  - Option A: store in S3 + pre-signed URL
  - Option B: host a simple `/audio/{id}` endpoint that serves bytes

MVP fastest path:
- Generate audio -> store bytes in memory map `{audio_id: bytes}` -> return `audio_url` as `ATTACKER_BASE_URL/api/v1/audio/{audio_id}`

### Required ENV
- `ELEVENLABS_API_KEY`
- `ELEVENLABS_VOICE_ID` (default)
- `BASE_URL` (for constructing audio_url)

---

## 5) Internal State (Keep simple)
In-memory maps:
- `session_turn_index[session_id] = int`
- `session_scenario_id[session_id] = scenario_id`
- `audio_store[audio_id] = bytes`

No database needed for hack MVP.

---

## 6) Stretch Goals (only if MVP done)
- LLM-driven adaptive scammer (hard mode)
- Difficulty slider modifies lines/tactics
- Caller “memory” (reuses trainee replies to pivot)

END.

