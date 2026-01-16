# UI_AGENTS.md — Retool Console (SE Shield)

## 0) Owner + Goal
**Owner:** UI engineer (Retool)  
**Goal:** Build the entire operator console in **Retool** that drives the demo end-to-end: start session → play attacker audio → capture trainee reply → show defender alerts/suggestions → end call → show report.

This doc defines ONLY the UI responsibilities and the exact API calls/fields needed so it plugs into the Attacker + Defender services.

---

## 1) What the UI Talks To (Contracts)
You will call **two backend services** (can be separate URLs or same API gateway):
- **Attacker Service** (scammer simulator): `ATTACKER_BASE_URL`
- **Defender Service** (copilot + scoring): `DEFENDER_BASE_URL`

### Required headers for all requests
- `Content-Type: application/json`
- Optional (stretch): `Authorization: Bearer <token>` (Auth0). **MVP can skip auth.**

### Shared identifier
- `session_id` (string). UI must store this and send to both services.

---

## 2) MVP in < 1 hour (UI)
### MVP Objective
A single Retool page that can:
1) Create session
2) Get next attacker line + play audio
3) Send trainee reply text
4) Display defender outputs (tactics, risk, 3 suggestions)
5) End session + display report

### MVP Components (single page)
**State variables**
- `session_id` (string)
- `scenario_id` (string, default `"ceo_impersonation_001"`)
- `turn_index` (number, starts at 0)
- `transcript` (array of `{speaker, text, ts}`) — UI-local is fine
- `latest_defense` (object) — from defender
- `report` (object) — from defender finalize

**UI widgets**
- Dropdown: Scenario (`scenario_id`)
- Button: Start Session
- Button: Next Caller Turn
- Audio Player: plays `attacker.audio_url` (or hide if none)
- Transcript display (Table/List)
- Text Input: `agent_reply_text`
- Button: Send Reply
- Alerts panel: tactics + risk label + reasons
- Suggestions panel: 3 cards with Copy buttons
- Button: End Call (Finalize)
- Report display area

---

## 3) API Calls (Exact)
### 3.1 Start Session
UI action on Start button:
- POST `${DEFENDER_BASE_URL}/api/v1/sessions`
Request:
{
  "scenario_id": "<scenario_id>",
  "demo_mode": true,
  "metadata": { "ui": "retool" }
}
Response:
{
  "session_id": "sess_...",
  "status": "live|created",
  "created_at": "...",
  "scenario_id": "..."
}
UI sets `session_id`, clears transcript/report.

(If defender doesn’t create sessions and attacker does: swap to attacker create, but keep `session_id` identical. Recommended: defender owns session creation.)

### 3.2 Next Caller Turn (attacker)
- POST `${ATTACKER_BASE_URL}/api/v1/sessions/${session_id}/next_caller`
Request:
{
  "scenario_id": "<scenario_id>"
}
Response:
{
  "session_id": "sess_...",
  "turn_index": 1,
  "caller_text": "string",
  "audio_url": "https://...",
  "tactics_used": ["authority_impersonation", "urgency_pressure"]
}

UI behavior:
- Append to transcript: `{speaker:"caller", text:caller_text, ts:now}`
- Play `audio_url` if present
- Immediately send this event to defender (next section)

### 3.3 Send Event to Defender (caller turn)
- POST `${DEFENDER_BASE_URL}/api/v1/sessions/${session_id}/events`
Request:
{
  "events": [
    {
      "type": "caller_turn",
      "timestamp": "<ISO8601>",
      "text": "<caller_text>",
      "turn_index": <turn_index>,
      "tactics_hint": ["..."]   // optional
    }
  ],
  "trigger_analysis": true
}
Response 202:
{
  "accepted": true
}

### 3.4 Send Trainee Reply
On Send Reply button:
- Append UI transcript item `{speaker:"agent", text:agent_reply_text, ts:now}`
- POST `${DEFENDER_BASE_URL}/api/v1/sessions/${session_id}/events`
Request:
{
  "events": [
    {
      "type": "agent_turn",
      "timestamp": "<ISO8601>",
      "text": "<agent_reply_text>",
      "turn_index": <turn_index>
    }
  ],
  "trigger_analysis": true
}

### 3.5 Poll Defender State (every 1–2s)
- GET `${DEFENDER_BASE_URL}/api/v1/sessions/${session_id}`
Response (minimum fields UI expects):
{
  "session_id":"sess_...",
  "risk": {
    "label":"low|medium|high",
    "escalation_score":0.0,
    "reasons":["..."]
  },
  "tactics_detected":["urgency_pressure","..."],
  "suggestions":[
    {"label":"policy_safe","text":"..."},
    {"label":"deescalate","text":"..."},
    {"label":"boundary_redirect","text":"..."}
  ],
  "score": {
    "overall": 72,
    "leak_risk": 90,
    "policy_adherence": 65,
    "notes":["..."]
  },
  "near_misses":[
    {"turn_index":2,"reason":"Asked for OTP","severity":"high"}
  ]
}

UI should render:
- Risk label + meter
- Tactics list badges
- Suggestion cards
- Scoreboard (big number)

### 3.6 Finalize / End Call
- POST `${DEFENDER_BASE_URL}/api/v1/sessions/${session_id}/finalize`
Request:
{
  "include_report": true
}
Response:
{
  "session_id":"sess_...",
  "report":{
    "tactics_used_summary":[{"tactic":"authority_impersonation","count":2}],
    "near_misses":[...],
    "score": {...},
    "coach_notes":["..."]
  }
}
UI shows report.

---

## 4) UI Reliability Rules (So integration doesn’t break)
- UI never invents server state; it always displays defender’s `GET /sessions/{id}` as source of truth.
- UI uses `session_id` returned from defender create.
- UI always calls attacker `next_caller` then posts caller event to defender.
- UI does not assume there are always 3 suggestions (render dynamic array).
- UI polls every 1–2 seconds using simple GET.

---

## 5) Stretch Goals (only if MVP is done)
- Auth0 login gating
- Multi-page UI (Policy, Replay)
- Audio for trainee reply (ElevenLabs “agent voice”)
- Replay timeline visualization

END.

