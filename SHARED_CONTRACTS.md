# SHARED_CONTRACTS.md — Common Types, Enums, and API Conventions

## 0) Purpose

This document defines the **single source of truth** for all shared types, enums, error formats, and API conventions used across the SE Shield system. All three services (Attacker, Defender, UI) must conform to these definitions to prevent integration drift.

In a monorepo, this should live at `/packages/shared-types/` and be imported by both backend services. The UI engineer should reference this document directly since Retool does not import TypeScript packages.

---

## 1) Tactic Taxonomy (Canonical Enum)

All services must use these exact string values. Do not invent new tactics without updating this document first.

```typescript
/**
 * Canonical list of social engineering tactics detected and scored by SE Shield.
 * Used by: Attacker scenarios, Defender detection, UI badge rendering.
 */
type Tactic =
  | "authority_impersonation"    // Claiming to be CEO, IT, security, executive
  | "urgency_pressure"           // "Right now", "immediately", "ASAP", time constraints
  | "credential_harvesting"      // Requesting passwords, OTPs, MFA codes, PINs
  | "identity_bypass"            // "Skip verification", "I'll verify later", "just do it"
  | "threat_intimidation"        // "You'll be fired", "I'll report you", legal threats
  | "emotional_manipulation"     // Sympathy plays, guilt trips, personal crises
  | "information_probing"        // Asking about account details, balances, existence
  | "callback_evasion";          // Refusing verified callback, insisting on immediate help
```

---

## 2) Risk Labels

```typescript
/**
 * Risk assessment levels returned by the Defender service.
 */
type RiskLabel = "low" | "medium" | "high" | "critical";
```

Mapping rules (Defender must implement):
- `low`: 0 tactics detected, no sensitive requests
- `medium`: 1-2 tactics detected, or minor sensitive request
- `high`: 3+ tactics detected, or credential harvesting attempt
- `critical`: Active credential disclosure or policy violation in progress

---

## 3) Session Status

```typescript
/**
 * Lifecycle states for a training session.
 */
type SessionStatus = "created" | "live" | "completed" | "abandoned";
```

Transitions:
- `created` → `live`: First event received
- `live` → `completed`: Finalize called OR scenario_complete event received
- `live` → `abandoned`: No activity for 30 minutes (stretch goal)

---

## 4) Event Types

```typescript
/**
 * Types of events ingested by the Defender service.
 */
type EventType = "caller_turn" | "agent_turn" | "scenario_complete";
```

- `caller_turn`: Attacker's line was played/displayed to trainee
- `agent_turn`: Trainee submitted a response
- `scenario_complete`: Attacker returned 409, no more lines available

---

## 5) Standardized Error Response Format

All services must return errors in this shape. This enables consistent UI error handling.

```typescript
/**
 * Canonical error response structure for all SE Shield APIs.
 */
type ErrorResponse = {
  error: {
    code: string;           // Machine-readable error code (e.g., "SCENARIO_DONE", "SESSION_NOT_FOUND")
    message: string;        // Human-readable description
    details?: unknown;      // Optional additional context
  };
  request_id?: string;      // Optional correlation ID for debugging
};
```

### Standard Error Codes

| Code | HTTP Status | Meaning |
|------|-------------|---------|
| `SESSION_NOT_FOUND` | 404 | Session ID does not exist |
| `SCENARIO_NOT_FOUND` | 404 | Scenario ID does not exist |
| `SCENARIO_DONE` | 409 | No more caller lines available |
| `DUPLICATE_EVENT` | 409 | Event with this event_id already processed |
| `INVALID_EVENT_ORDER` | 400 | Event turn_index is invalid or out of order |
| `SESSION_NOT_LIVE` | 400 | Session is completed/abandoned, cannot accept events |
| `UNAUTHORIZED` | 401 | Missing or invalid API key |
| `RATE_LIMITED` | 429 | Too many requests for this session |
| `INTERNAL_ERROR` | 500 | Unexpected server error |

---

## 6) Scenario Schema

Scenarios are stored in `/scenarios/*.json` and must conform to this schema. Both Attacker and Defender should validate scenarios on startup.

```typescript
/**
 * Schema for a training scenario file.
 */
type Scenario = {
  scenario_id: string;              // Unique identifier (e.g., "ceo_impersonation_001")
  title: string;                    // Human-readable name
  description?: string;             // Optional description for UI
  difficulty: "easy" | "medium" | "hard";
  expected_duration_seconds: number;
  lines: ScenarioLine[];
};

type ScenarioLine = {
  text: string;                     // What the caller says
  tactics: Tactic[];                // Tactics employed in this line
  audio_cache_key?: string;         // Pre-generated audio file reference (optional)
};
```

Example:
```json
{
  "scenario_id": "ceo_impersonation_001",
  "title": "CEO Impersonation",
  "description": "A caller claims to be the CEO and demands an immediate MFA reset.",
  "difficulty": "medium",
  "expected_duration_seconds": 180,
  "lines": [
    {
      "text": "Hi, this is the CEO. I need you to reset my MFA right now.",
      "tactics": ["authority_impersonation", "urgency_pressure"]
    },
    {
      "text": "I'm in a board meeting. If you can't help, I'll have your manager handle it.",
      "tactics": ["threat_intimidation", "urgency_pressure"]
    },
    {
      "text": "Look, I can verify later. Just do it, it's critical.",
      "tactics": ["identity_bypass", "urgency_pressure"]
    }
  ]
}
```

---

## 7) Common Request Headers

All API requests must include:

```
Content-Type: application/json
X-API-Key: <api_key>
```

The `X-API-Key` header is required even in MVP. Use a shared secret configured via environment variable. This prevents accidental abuse of ElevenLabs credits and session spam.

Optional (stretch goal):
```
Authorization: Bearer <auth0_token>
```

---

## 8) Health and Version Endpoints

Both Attacker and Defender services must implement:

### GET /health

Response 200:
```json
{
  "status": "ok",
  "service": "attacker|defender",
  "timestamp": "2024-01-15T10:30:00Z"
}
```

Response 503 (if dependencies unavailable):
```json
{
  "status": "degraded",
  "service": "attacker",
  "issues": ["elevenlabs_unreachable"],
  "timestamp": "2024-01-15T10:30:00Z"
}
```

### GET /version

Response 200:
```json
{
  "version": "0.1.0",
  "commit": "abc123f",
  "built_at": "2024-01-15T10:00:00Z"
}
```

---

## 9) Deployment Constraints (MVP)

**Critical:** The MVP uses in-memory state for sessions, events, and audio. This architecture requires:

1. **Single-instance deployment per service.** Do not deploy to serverless (Vercel Functions, AWS Lambda) or enable auto-scaling.
2. **Long-lived processes.** Use a VM, container, or local development server.
3. **Accepted limitation:** Service restart clears all sessions. This is acceptable for a hackathon demo.

If you need multi-instance or serverless deployment, you must add Redis for session state and S3 for audio storage. This is a stretch goal, not MVP.

---

## 10) Turn Index Ownership Rules

**The Defender service owns the canonical turn index.** Here's how it works:

1. When Defender creates a session, `current_turn_index` starts at 0.
2. When Defender receives a `caller_turn` event, it increments `current_turn_index` and assigns that value to the event.
3. When Defender receives an `agent_turn` event, it uses the same `current_turn_index` (agent replies to the most recent caller turn).
4. The Attacker service tracks its own line pointer internally, but this is not the canonical turn index.
5. The UI should **not** maintain its own turn counter. It reads `turn_index` from Defender responses.

This prevents off-by-one errors and ensures the final report's "near miss on turn 3" references are accurate.

---

## 11) Idempotency

To prevent duplicate events from network retries, every event must include a client-generated `event_id` (UUID). The Defender will:

1. Check if `event_id` has been seen for this session.
2. If seen: return 409 `DUPLICATE_EVENT` (safe to ignore on client).
3. If new: process the event and store the `event_id`.

This makes event submission safe to retry on network failure.

---

---

## 12) External Tool Integrations

### 12.1 Tonic Fabricate (Test Data Generation)

[Tonic Fabricate](https://docs.tonic.ai/fabricate) is an AI-powered synthetic data generation platform that can accelerate development by generating realistic test data from natural language prompts.

**Relevance to SE Shield:**
- Generate diverse scenario libraries without manual authoring
- Create realistic caller personas, account details, and conversation contexts
- Mock API endpoints for frontend development before backend is ready
- Generate edge-case scenarios for stress testing the Defender

**Recommended Uses:**

1. **Scenario Generation (Attacker Service)**
   ```
   Prompt: "Generate 10 social engineering scenarios for a bank helpdesk, 
   including CEO fraud, tech support scams, and account takeover attempts. 
   Each scenario should have 6-8 caller lines with escalating manipulation tactics."
   ```
   
2. **Mock Session Data (Defender Service)**
   ```
   Prompt: "Generate 50 training session records with transcripts, scores, 
   near-misses, and tactic detection results for a customer support 
   social engineering training platform."
   ```

3. **Development Mock APIs**
   - Use Fabricate's mock API feature to simulate Attacker/Defender endpoints
   - Allows UI development to proceed independently of backend

**Integration Points:**
- Export scenarios as JSON matching our `Scenario` schema
- Generate test data for CI/CD pipeline validation
- Create demo datasets for sales presentations

**Pricing Note:** Fabricate has a free tier suitable for hackathon use. No integration code required—use it as a development tool to generate static JSON files.

---

### 12.2 Modulate.ai (Voice Fraud Detection)

[Modulate.ai](https://www.modulate.ai/solutions/commercial-fraud) provides enterprise-grade real-time voice fraud detection that directly overlaps with—and could enhance—the Defender service's capabilities.

**What Modulate Offers:**
- Real-time conversational intelligence detecting manipulation, deception, and fraud
- Emotion, prosody, and intent analysis (not just keyword matching)
- Synthetic voice / deepfake detection
- Processes millions of conversations at scale
- 18+ language support

**Competitive Positioning:**

SE Shield and Modulate.ai serve different purposes:

| Aspect | SE Shield | Modulate.ai |
|--------|-----------|-------------|
| Primary Use | Training simulation | Production fraud detection |
| Target User | Support rep trainees | Security/fraud teams |
| Audio Source | Synthetic (ElevenLabs) | Real customer calls |
| Analysis | Rules-based (MVP) | ML-based, battle-tested |
| Cost | Self-hosted, hackathon | Enterprise pricing |

**SE Shield is a training tool**—it creates a safe environment to practice. **Modulate is a production tool**—it monitors real calls. These are complementary, not competitive.

**Potential Integration (Stretch Goal):**

If SE Shield evolves beyond hackathon MVP, Modulate integration could provide:

1. **Enhanced Defender Analysis**
   - Replace rules-based tactic detection with Modulate's ML models
   - More accurate detection of manipulation patterns
   - Emotion and prosody analysis beyond text content

2. **Synthetic Voice Detection Training**
   - Modulate can detect that the attacker audio is AI-generated
   - Train reps to recognize deepfake voice characteristics
   - Add "deepfake awareness" as a training dimension

3. **Realism Validation**
   - Use Modulate to score how realistic our ElevenLabs voices are
   - Ensure training scenarios match real-world attack sophistication

**Integration Architecture (If Pursued):**

```
┌─────────────────────────────────────────────────────────────────┐
│                        DEFENDER SERVICE                          │
│                                                                  │
│  ┌────────────────┐     ┌────────────────┐     ┌──────────────┐ │
│  │  Rules-Based   │     │   Modulate.ai  │     │   Results    │ │
│  │   Analysis     │────►│   API (opt)    │────►│   Merger     │ │
│  │  (always runs) │     │  (if enabled)  │     │              │ │
│  └────────────────┘     └────────────────┘     └──────────────┘ │
│                                                                  │
│  Feature Flag: ENABLE_MODULATE_INTEGRATION=true|false           │
│  Env Var: MODULATE_API_KEY=...                                  │
└─────────────────────────────────────────────────────────────────┘
```

**API Contract (Hypothetical):**

```typescript
// If Modulate integration is enabled, Defender would call:
type ModulateAnalysisRequest = {
  audio_url?: string;           // URL to audio file (optional)
  transcript: string;           // Text transcript
  session_context?: {
    caller_claimed_identity: string;
    call_reason: string;
  };
};

type ModulateAnalysisResponse = {
  fraud_probability: number;    // 0.0 to 1.0
  deception_indicators: string[];
  emotion_analysis: {
    stress: number;
    urgency: number;
    anger: number;
  };
  is_synthetic_voice: boolean;
  confidence: number;
};
```

**Decision for MVP:**
- **Do NOT integrate Modulate for hackathon MVP.** It adds complexity and cost.
- **Do mention Modulate in the pitch** as a potential production integration path.
- **Do keep the Defender architecture modular** so Modulate could be added later.

---

## 13) Tool Decision Matrix

| Tool | Use in MVP? | Purpose | Integration Effort |
|------|-------------|---------|-------------------|
| **ElevenLabs** | ✅ Yes | TTS for attacker voice | Low (API calls) |
| **Tonic Fabricate** | ✅ Yes (dev tool) | Generate test scenarios | None (export JSON) |
| **Modulate.ai** | ❌ No (stretch) | Enhanced fraud detection | Medium (API + merge logic) |
| **Ray** | ❌ No (stretch) | Parallel analysis | Medium (cluster setup) |
| **Auth0** | ❌ No (stretch) | Role-based access | Medium (OAuth flow) |

---

END OF SHARED CONTRACTS.
