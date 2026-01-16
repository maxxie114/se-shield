# DEFENDER_AGENTS.md — Blue-Team Copilot (Ray Parallel Analysis + API for Retool)

## 0) Owner + Goal
**Owner:** Defender engineer  
**Goal:** Build the core copilot service that:
- owns the session state
- ingests events from UI (caller and trainee turns)
- runs **parallel analysis using Ray** per turn
- returns live **tactic detection + risk + 3 safe reply suggestions + scoring**
- exposes stable endpoints for Retool to poll

This is the “brain” and integration hub.

---

## 1) MVP in < 1 hour (Defender)
### MVP Objective
Implement a minimal service that supports:
- Create session
- Ingest events
- On each event, compute defender output (tactics/risk/suggestions/score)
- Allow UI polling via `GET /sessions/{id}`
- Finalize report

### MVP Simplifications
- Use in-memory storage (dict)
- Use Ray locally (optional if time), but MVP MUST run without Ray if Ray setup fails
- Use a simple ruleset for detection if LLM isn’t ready (keyword-based)
- Suggestions can be template-based (safe reply patterns)

You can add LLM later; MVP must be deterministic and demo-safe.

---

## 2) Core Data Model (Minimal)
Session state:
{
  "session_id": "sess_...",
  "scenario_id": "ceo_impersonation_001",
  "created_at": "...",
  "events": [
    {"type":"caller_turn","turn_index":1,"timestamp":"...","text":"..."},
    {"type":"agent_turn","turn_index":1,"timestamp":"...","text":"..."}
  ],
  "latest": {
    "risk": {...},
    "tactics_detected": [...],
    "suggestions": [...],
    "score": {...},
    "near_misses": [...]
  },
  "tactic_counts": {"urgency_pressure":2}
}

---

## 3) API Contract (Defender Service)
Base: `DEFENDER_BASE_URL`

### 3.1 Create Session
POST `/api/v1/sessions`

Request:
{
  "scenario_id": "ceo_impersonation_001",
  "demo_mode": true,
  "metadata": {}
}

Response 201:
{
  "session_id": "sess_...",
  "status": "live",
  "scenario_id": "ceo_impersonation_001",
  "created_at": "..."
}

### 3.2 Ingest Events
POST `/api/v1/sessions/{session_id}/events`

Request:
{
  "events": [
    {
      "type": "caller_turn|agent_turn",
      "timestamp": "ISO",
      "text": "string",
      "turn_index": 1,
      "tactics_hint": ["..."]   // optional
    }
  ],
  "trigger_analysis": true
}

Response 202:
{ "accepted": true }

Rules:
- Defender must append events to session log
- If `trigger_analysis=true`, run analysis immediately and update `latest`

### 3.3 Poll Session State (Retool uses this)
GET `/api/v1/sessions/{session_id}`

Response 200 (minimum fields UI expects):
{
  "session_id": "sess_...",
  "scenario_id": "ceo_impersonation_001",
  "risk": {
    "label":"low|medium|high",
    "escalation_score":0.0,
    "reasons":["..."]
  },
  "tactics_detected":["..."],
  "suggestions":[
    {"label":"policy_safe","text":"..."},
    {"label":"deescalate","text":"..."},
    {"label":"boundary_redirect","text":"..."}
  ],
  "score":{
    "overall":72,
    "leak_risk":90,
    "policy_adherence":65,
    "notes":["..."]
  },
  "near_misses":[
    {"turn_index":2,"reason":"Asked for OTP","severity":"high"}
  ]
}

### 3.4 Finalize
POST `/api/v1/sessions/{session_id}/finalize`

Request:
{ "include_report": true }

Response 200:
{
  "session_id":"sess_...",
  "report":{
    "tactics_used_summary":[{"tactic":"authority_impersonation","count":2}],
    "near_misses":[...],
    "score": {...},
    "coach_notes":["..."]
  }
}

---

## 4) Defender Analysis (MVP Rules + Optional LLM)

### 4.1 Tactic Detection (Rules-first MVP)
Detect tactics by keywords/phrases in the latest caller turn (and sometimes agent turn).
Examples:
- authority_impersonation: ["ceo","boss","vp","director","i’m from it","security team"]
- urgency_pressure: ["right now","immediately","urgent","asap","board meeting","time sensitive"]
- credential_harvesting: ["password","otp","code","verification code","2fa","mfa code"]
- identity_bypass: ["skip verification","don’t need that","i’ll verify later","just do it"]
- threat_intimidation: ["you’ll be fired","i’ll report you","lawsuit","compliance will hear"]

Output:
- `tactics_detected[]`
- `risk_label`: low/medium/high based on number + severity of tactics

### 4.2 Safe Reply Suggestions (Template MVP)
Return 3 short suggestions:
1) policy_safe:
   "I can help, but I can’t proceed without verifying your identity. Let’s start with [verification step]."
2) deescalate:
   "I hear this is urgent. I’ll do my best to help quickly, and we’ll follow the required steps to keep your account safe."
3) boundary_redirect:
   "I can’t assist with that request. I can escalate to a supervisor or follow our verified callback process."

### 4.3 Near-miss Detection (Agent reply scanning)
If agent text includes risky patterns:
- "what’s your password" / "tell me the code" => near_miss high
- "yes i see your account" => near_miss medium (account existence confirmation)
- "i can reset without verification" => near_miss high

### 4.4 Scoring (simple)
Start at 100, subtract:
- -30 per high near_miss
- -10 per medium near_miss
Add small bonus if agent uses safe patterns:
- +5 if mentions verification
- +5 if escalates appropriately
Clamp 0..100.
Return dimensioned scores:
- leak_risk, policy_adherence, recognition, overall

### 4.5 Optional LLM (post-MVP)
If time:
- Replace/augment detection with a single LLM call that returns structured JSON:
  - tactics_detected, suspicious_requests, safe_replies, risk, reasons
Still keep rules fallback.

---

## 5) Ray Parallelization (Required for this doc, but keep it safe)
### MVP Ray usage pattern (if possible)
When a new event arrives, run in parallel:
- `detect_tactics_task(latest_context)`
- `suggest_replies_task(latest_context)`
- `score_task(latest_context)`

Then merge results into `session.latest`.

### If Ray setup is slow
MVP must still ship:
- implement the same functions synchronously
- add a flag `USE_RAY=false` to bypass Ray

ENV:
- `USE_RAY=true|false`
- `RAY_ADDRESS` optional

---

## 6) Integration Rules (So everything works together)
- Defender is **source of truth** for session state shown in Retool.
- UI always posts attacker caller text to defender via `/events`.
- Attacker does not talk to defender directly (UI is the bridge).
- Field names in `GET /sessions/{id}` must remain stable (UI depends on them).
- Defender must tolerate missing `tactics_hint` and missing attacker fields.

---

## 7) Stretch Goals (only if MVP done)
- Auth0 role gating
- Policy versions + supervisor publish
- Replay timeline endpoint
- LLM-based tactic classifier + richer coaching notes

END.

