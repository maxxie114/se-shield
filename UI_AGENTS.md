# UI_AGENTS.md — Retool Console (SE Shield)

## 0) Owner + Goal

**Owner:** UI engineer (Retool)  
**Goal:** Build the operator console in **Retool** that drives the demo end-to-end: health check → start session → play attacker audio → capture trainee reply → show defender alerts/suggestions → end call → show report.

This document defines the UI responsibilities and the **exact API calls/fields** needed so it integrates cleanly with the Attacker and Defender services.

**Before you start:** Read SHARED_CONTRACTS.md for the canonical tactic names, error codes, and data shapes. Your UI must handle all error codes defined there.

---

## 1) Architecture Overview

The UI talks to **two backend services** (may be same host or different):

```
┌─────────────────────────────────────────────────────────────────────┐
│                          RETOOL UI                                   │
│                                                                      │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐      │
│  │  Start   │───►│   Next   │───►│  Send    │───►│   End    │      │
│  │ Session  │    │  Caller  │    │  Reply   │    │   Call   │      │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘      │
│       │               │               │               │             │
└───────┼───────────────┼───────────────┼───────────────┼─────────────┘
        │               │               │               │
        ▼               ▼               ▼               ▼
   ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
   │DEFENDER │    │ATTACKER │    │DEFENDER │    │DEFENDER │
   │ create  │    │  next   │    │ events  │    │finalize │
   │ session │    │ caller  │    │         │    │         │
   └─────────┘    └─────────┘    └─────────┘    └─────────┘
                       │
                       ▼
                  ┌─────────┐
                  │DEFENDER │  (UI posts caller event here too)
                  │ events  │
                  └─────────┘
```

**Key Integration Rules:**
1. Defender is the **source of truth** for all session state shown in the UI
2. UI always posts attacker's caller text to defender immediately after receiving it
3. UI uses `tactics` field (not `tactics_used` or `tactics_hint`) — this is standardized
4. UI reads `turn_index` from defender responses, never maintains its own counter
5. UI polls defender every 1-2 seconds only when session is `live`

---

## 2) Environment Configuration

Configure these in Retool as environment variables or resource settings:

```
ATTACKER_BASE_URL = http://localhost:8001
DEFENDER_BASE_URL = http://localhost:8002
API_KEY = your-shared-secret
POLL_INTERVAL_MS = 1500
```

---

## 3) MVP Deliverables (Target: < 1 hour)

### Must Have
1. Health check on page load (both services)
2. Scenario dropdown populated from attacker API
3. Start session button
4. Next caller turn button with audio playback
5. Trainee reply text input and send button
6. Real-time display of defender state (risk, tactics, suggestions, score)
7. Handle 409 SCENARIO_DONE gracefully
8. End call button with report display
9. Transcript display with speaker labels
10. Sync status indicator

### Nice to Have (Stretch Goals)
- Auth0 login
- Multi-page UI (training, replay, policy management)
- Audio for trainee replies
- Detailed timeline visualization

---

## 4) State Variables

Define these as Retool state variables:

```javascript
/**
 * Core session state
 */
const sessionId = null;           // string | null - From defender create response
const scenarioId = "ceo_impersonation_001";  // string - Selected scenario
const sessionStatus = "idle";     // "idle" | "created" | "live" | "completed"

/**
 * Service health
 */
const attackerHealthy = false;    // boolean - From health check
const defenderHealthy = false;    // boolean - From health check
const servicesChecked = false;    // boolean - True after initial health check

/**
 * Transcript (derived from defender events endpoint)
 */
const transcript = [];            // Array<{speaker: "caller"|"agent", text: string, timestamp: string, turnIndex: number}>

/**
 * Defender state (from polling)
 */
const defenderState = {
  risk: null,                     // {label, escalation_score, reasons[]}
  tactics_detected: [],           // string[]
  suggestions: [],                // Array<{label, text}>
  score: null,                    // {overall, leak_risk, policy_adherence, recognition, notes[]}
  near_misses: [],                // Array<{turn_index, reason, severity}>
  current_turn_index: 0,          // number
  updated_at: null                // string (ISO timestamp)
};

/**
 * UI state
 */
const agentReplyText = "";        // string - Current text in reply input
const lastPollTime = null;        // string - For conditional polling
const scenarioComplete = false;   // boolean - True when attacker returns 409
const currentAudioUrl = null;     // string | null - For audio player
const isLoading = false;          // boolean - For button disable states
const syncedToDefender = true;    // boolean - False while events are in flight

/**
 * Report (from finalize)
 */
const report = null;              // object | null - From defender finalize response

/**
 * Available scenarios (from attacker)
 */
const scenarios = [];             // Array<{scenario_id, title, description, difficulty, line_count}>
```

---

## 5) API Calls (Exact Specifications)

All API calls must include these headers:

```javascript
const headers = {
  "Content-Type": "application/json",
  "X-API-Key": "{{API_KEY}}"
};
```

---

### 5.1 Health Checks (On Page Load)

Run both health checks when the page loads. Show a warning banner if either fails.

**Check Attacker Health:**
```
GET {{ATTACKER_BASE_URL}}/health
```

Response 200:
```json
{
  "status": "ok",
  "service": "attacker",
  "scenarios_loaded": 3,
  "audio_files_cached": 24,
  "timestamp": "2024-01-15T10:30:00Z"
}
```

**Check Defender Health:**
```
GET {{DEFENDER_BASE_URL}}/health
```

Response 200:
```json
{
  "status": "ok",
  "service": "defender",
  "active_sessions": 0,
  "timestamp": "2024-01-15T10:30:00Z"
}
```

**UI Behavior:**
- If either returns non-200 or `status !== "ok"`: Show error banner "Service unavailable: {service name}"
- If both healthy: Set `attackerHealthy = true`, `defenderHealthy = true`, `servicesChecked = true`
- Enable the Start Session button only if both services are healthy

---

### 5.2 Load Available Scenarios

Fetch scenarios after health check passes.

```
GET {{ATTACKER_BASE_URL}}/api/v1/scenarios
```

Response 200:
```json
{
  "scenarios": [
    {
      "scenario_id": "ceo_impersonation_001",
      "title": "CEO Impersonation",
      "description": "A caller claims to be the CEO and demands an immediate MFA reset.",
      "difficulty": "medium",
      "line_count": 6,
      "expected_duration_seconds": 180
    }
  ]
}
```

**UI Behavior:**
- Populate scenario dropdown with `scenarios` array
- Default select the first scenario
- Display `title` in dropdown, store `scenario_id` as value

---

### 5.3 Start Session

Called when user clicks "Start Session" button.

```
POST {{DEFENDER_BASE_URL}}/api/v1/sessions
```

Request:
```json
{
  "scenario_id": "{{scenarioId}}",
  "metadata": {
    "ui": "retool",
    "started_by": "demo_user"
  }
}
```

Response 201:
```json
{
  "session_id": "sess_f47ac10b58cc",
  "scenario_id": "ceo_impersonation_001",
  "status": "created",
  "created_at": "2024-01-15T10:30:00Z"
}
```

**UI Behavior:**
- Set `sessionId = response.session_id`
- Set `sessionStatus = "created"`
- Clear `transcript`, `report`, `defenderState`
- Set `scenarioComplete = false`
- Enable "Next Caller Turn" button
- Disable scenario dropdown (can't change mid-session)
- Start polling timer (see section 5.7)

---

### 5.4 Next Caller Turn

Called when user clicks "Next Caller Turn" button.

**Step 1: Get caller line from Attacker**

```
POST {{ATTACKER_BASE_URL}}/api/v1/sessions/{{sessionId}}/next_caller
```

Request:
```json
{
  "scenario_id": "{{scenarioId}}"
}
```

Response 200 (success):
```json
{
  "session_id": "sess_f47ac10b58cc",
  "scenario_id": "ceo_impersonation_001",
  "line_index": 0,
  "caller_text": "Hi, this is the CEO. I need you to reset my MFA right now.",
  "tactics": ["authority_impersonation", "urgency_pressure"],
  "audio_url": "http://localhost:8001/api/v1/audio/ceo_impersonation_001_0",
  "lines_remaining": 5
}
```

Response 409 (scenario complete):
```json
{
  "error": {
    "code": "SCENARIO_DONE",
    "message": "No more caller lines available in this scenario"
  }
}
```

**Step 2: Post caller event to Defender**

Only do this if Step 1 returned 200 (not 409).

```
POST {{DEFENDER_BASE_URL}}/api/v1/sessions/{{sessionId}}/events
```

Request:
```json
{
  "events": [
    {
      "event_id": "{{generateUUID()}}",
      "type": "caller_turn",
      "timestamp": "{{new Date().toISOString()}}",
      "text": "{{attacker_response.caller_text}}",
      "tactics": {{attacker_response.tactics}}
    }
  ]
}
```

Response 202:
```json
{
  "accepted": true,
  "events_processed": 1,
  "session_status": "live",
  "updated_at": "2024-01-15T10:30:05Z"
}
```

**UI Behavior on 200:**
- Set `syncedToDefender = false` before posting to defender
- Set `currentAudioUrl = response.audio_url`
- Play audio if `audio_url` is not null
- After posting to defender: Set `syncedToDefender = true`
- Set `sessionStatus = "live"` (from defender response)
- Wait for next poll to update transcript (defender is source of truth)

**UI Behavior on 409 SCENARIO_DONE:**
- Set `scenarioComplete = true`
- Disable "Next Caller Turn" button
- Show banner: "Scenario complete! Click End Call to see your report."
- Post `scenario_complete` event to defender:

```
POST {{DEFENDER_BASE_URL}}/api/v1/sessions/{{sessionId}}/events
```

Request:
```json
{
  "events": [
    {
      "event_id": "{{generateUUID()}}",
      "type": "scenario_complete",
      "timestamp": "{{new Date().toISOString()}}",
      "text": ""
    }
  ]
}
```

---

### 5.5 Send Trainee Reply

Called when user clicks "Send Reply" button or presses Enter in the text input.

```
POST {{DEFENDER_BASE_URL}}/api/v1/sessions/{{sessionId}}/events
```

Request:
```json
{
  "events": [
    {
      "event_id": "{{generateUUID()}}",
      "type": "agent_turn",
      "timestamp": "{{new Date().toISOString()}}",
      "text": "{{agentReplyText}}"
    }
  ]
}
```

Response 202:
```json
{
  "accepted": true,
  "events_processed": 1,
  "session_status": "live",
  "updated_at": "2024-01-15T10:30:15Z"
}
```

**UI Behavior:**
- Set `syncedToDefender = false` before posting
- Clear `agentReplyText` input
- After response: Set `syncedToDefender = true`
- Wait for next poll to update transcript and defender state

---

### 5.6 Recover Transcript (On Refresh or Initial Load)

If the page is refreshed mid-session, recover the transcript from defender.

```
GET {{DEFENDER_BASE_URL}}/api/v1/sessions/{{sessionId}}/events
```

Response 200:
```json
{
  "session_id": "sess_f47ac10b58cc",
  "events": [
    {
      "event_id": "evt_a1b2c3d4",
      "type": "caller_turn",
      "turn_index": 1,
      "timestamp": "2024-01-15T10:30:05Z",
      "text": "Hi, this is the CEO. I need you to reset my MFA right now.",
      "tactics": ["authority_impersonation", "urgency_pressure"]
    },
    {
      "event_id": "evt_e5f6g7h8",
      "type": "agent_turn",
      "turn_index": 1,
      "timestamp": "2024-01-15T10:30:25Z",
      "text": "Hello! I'd be happy to help. Before I can make any changes, I'll need to verify your identity.",
      "tactics": []
    }
  ]
}
```

**UI Behavior:**
- Transform events into transcript format:
```javascript
const transcript = response.events
  .filter(e => e.type !== "scenario_complete")
  .map(e => ({
    speaker: e.type === "caller_turn" ? "caller" : "agent",
    text: e.text,
    timestamp: e.timestamp,
    turnIndex: e.turn_index
  }));
```

---

### 5.7 Poll Defender State

Poll every 1-2 seconds **only when `sessionStatus === "live"`**.

```
GET {{DEFENDER_BASE_URL}}/api/v1/sessions/{{sessionId}}?since={{lastPollTime}}
```

Response 200:
```json
{
  "session_id": "sess_f47ac10b58cc",
  "scenario_id": "ceo_impersonation_001",
  "status": "live",
  "updated_at": "2024-01-15T10:30:15Z",
  "current_turn_index": 2,
  "risk": {
    "label": "high",
    "escalation_score": 0.75,
    "reasons": ["Authority impersonation detected", "Urgency pressure detected"]
  },
  "tactics_detected": ["authority_impersonation", "urgency_pressure"],
  "suggestions": [
    {"label": "policy_safe", "text": "I can help, but I need to verify your identity first..."},
    {"label": "deescalate", "text": "I understand this is urgent..."},
    {"label": "boundary_redirect", "text": "I'm not able to bypass verification..."}
  ],
  "score": {
    "overall": 72,
    "leak_risk": 85,
    "policy_adherence": 65,
    "recognition": 70,
    "notes": ["Good: Did not confirm account existence"]
  },
  "near_misses": []
}
```

Response 304 (not modified):
No body. Do not update UI.

**UI Behavior:**
- Only poll if `sessionStatus === "live"`
- Update `defenderState` with response data
- Set `lastPollTime = response.updated_at`
- If `response.status === "completed"`: Set `sessionStatus = "completed"` and stop polling
- Update transcript by calling the events endpoint periodically (or diff against local state)

**Polling Logic (Retool Timer):**
```javascript
// In a Retool timer or interval query
if (sessionStatus === "live" && sessionId) {
  await pollDefenderState.trigger();
}
```

---

### 5.8 End Call / Finalize

Called when user clicks "End Call" button.

```
POST {{DEFENDER_BASE_URL}}/api/v1/sessions/{{sessionId}}/finalize
```

Request:
```json
{
  "include_report": true
}
```

Response 200:
```json
{
  "session_id": "sess_f47ac10b58cc",
  "status": "completed",
  "report": {
    "scenario_id": "ceo_impersonation_001",
    "duration_seconds": 185,
    "total_turns": 6,
    "tactics_used_summary": [
      {"tactic": "authority_impersonation", "count": 3},
      {"tactic": "urgency_pressure", "count": 4}
    ],
    "near_misses": [
      {"turn_index": 2, "reason": "Almost confirmed account details", "severity": "medium"}
    ],
    "score": {
      "overall": 72,
      "leak_risk": 85,
      "policy_adherence": 65,
      "recognition": 70
    },
    "coach_notes": [
      "Strong start: Immediately asked for verification",
      "Good recovery after pressure escalated"
    ],
    "grade": "B",
    "passed": true
  }
}
```

**UI Behavior:**
- Set `sessionStatus = "completed"`
- Set `report = response.report`
- Stop polling timer
- Show report panel
- Hide live defender state panel
- Show "Start New Session" button

---

## 6) UI Layout Specification

### 6.1 Header Bar

```
┌────────────────────────────────────────────────────────────────────┐
│  🛡️ SE Shield Training Console                    [Services: ✓ ✓]  │
│  Scenario: [CEO Impersonation    ▼]  [Start Session]              │
└────────────────────────────────────────────────────────────────────┘
```

Components:
- Logo and title
- Service health indicators (green checkmarks or red X for Attacker/Defender)
- Scenario dropdown (disabled during active session)
- Start Session button (disabled if services unhealthy or session active)

---

### 6.2 Main Panel (During Session)

```
┌─────────────────────────────────┬──────────────────────────────────┐
│                                 │  RISK: ████████░░ HIGH           │
│  TRANSCRIPT                     │                                  │
│  ─────────────────────────────  │  Detected Tactics:               │
│  📞 Caller (Turn 1):            │  [authority_impersonation]       │
│  "Hi, this is the CEO..."       │  [urgency_pressure]              │
│                                 │                                  │
│  👤 You (Turn 1):               │  ─────────────────────────────   │
│  "I'd be happy to help..."      │  SUGGESTED RESPONSES:            │
│                                 │  ┌─────────────────────────────┐ │
│  📞 Caller (Turn 2):            │  │ Policy Safe           [📋] │ │
│  "I'm in a board meeting..."    │  │ "I can help, but I need..." │ │
│                                 │  └─────────────────────────────┘ │
│  [Synced ✓]                     │  ┌─────────────────────────────┐ │
│                                 │  │ De-escalate           [📋] │ │
│  ─────────────────────────────  │  │ "I understand this is..."   │ │
│                                 │  └─────────────────────────────┘ │
│  🔊 [▶ Play Audio]              │  ┌─────────────────────────────┐ │
│                                 │  │ Boundary              [📋] │ │
│  Your Reply:                    │  │ "I'm not able to bypass..." │ │
│  ┌─────────────────────────────┐│  └─────────────────────────────┘ │
│  │                             ││                                  │
│  └─────────────────────────────┘│  ─────────────────────────────   │
│  [Send Reply]  [Next Caller]    │  SCORE: 72                       │
│                [End Call]       │  Leak Risk: 85 │ Policy: 65      │
│                                 │  Recognition: 70                 │
└─────────────────────────────────┴──────────────────────────────────┘
```

**Left Panel - Conversation:**
- Transcript list with speaker icons and turn numbers
- Sync indicator showing "Synced ✓" or "Syncing..." 
- Audio player for current caller line
- Text input for trainee reply
- Action buttons: Send Reply, Next Caller Turn, End Call

**Right Panel - Defender Dashboard:**
- Risk meter with label and escalation score
- Tactics detected as badges
- Three suggestion cards with copy-to-clipboard buttons
- Score display with all four dimensions
- Near-miss warnings (if any)

---

### 6.3 Report Panel (After Finalize)

```
┌────────────────────────────────────────────────────────────────────┐
│                    TRAINING REPORT                                  │
│                                                                     │
│  Scenario: CEO Impersonation                                        │
│  Duration: 3:05                                                     │
│  Total Turns: 6                                                     │
│                                                                     │
│  ┌──────────────────────┐  ┌──────────────────────┐                │
│  │  OVERALL SCORE       │  │  GRADE: B            │                │
│  │       72             │  │  PASSED ✓            │                │
│  └──────────────────────┘  └──────────────────────┘                │
│                                                                     │
│  Score Breakdown:                                                   │
│  • Leak Risk:        85/100  ████████░░                            │
│  • Policy Adherence: 65/100  ██████░░░░                            │
│  • Recognition:      70/100  ███████░░░                            │
│                                                                     │
│  Tactics Used Against You:                                          │
│  • authority_impersonation (3 times)                               │
│  • urgency_pressure (4 times)                                      │
│  • identity_bypass (2 times)                                       │
│                                                                     │
│  Near Misses:                                                       │
│  ⚠️ Turn 2: Almost confirmed account details (medium severity)      │
│                                                                     │
│  Coach Notes:                                                       │
│  ✓ Strong start: Immediately asked for verification                │
│  ✓ Good recovery after pressure escalated                          │
│  → Consider: Offer callback option earlier                         │
│                                                                     │
│  [Start New Session]  [Download Report]                            │
└────────────────────────────────────────────────────────────────────┘
```

---

## 7) Error Handling

Handle all error codes from SHARED_CONTRACTS.md:

| Error Code | HTTP Status | UI Behavior |
|------------|-------------|-------------|
| `SESSION_NOT_FOUND` | 404 | Show error, reset to idle state |
| `SCENARIO_NOT_FOUND` | 404 | Show error, let user pick different scenario |
| `SCENARIO_DONE` | 409 | Set `scenarioComplete = true`, show completion banner |
| `DUPLICATE_EVENT` | 409 | Ignore (idempotency working as expected) |
| `INVALID_EVENT_ORDER` | 400 | Show error, suggest refresh |
| `SESSION_NOT_LIVE` | 400 | Transition to completed state |
| `UNAUTHORIZED` | 401 | Show "Check API key configuration" error |
| `RATE_LIMITED` | 429 | Show "Slow down" message, increase poll interval |
| `INTERNAL_ERROR` | 500 | Show "Server error, please try again" |

**Error Display:**
```javascript
// Show error toast/banner
function showError(error) {
  const message = error?.error?.message || "Unknown error";
  const code = error?.error?.code || "UNKNOWN";
  // Display in Retool notification component
}
```

---

## 8) Helper Functions

### 8.1 Generate UUID

```javascript
/**
 * Generate a UUID v4 for event_id.
 * @returns {string} UUID in format "evt_xxxxxxxxxxxx"
 */
function generateEventId() {
  const uuid = 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, function(c) {
    const r = Math.random() * 16 | 0;
    const v = c === 'x' ? r : (r & 0x3 | 0x8);
    return v.toString(16);
  });
  return `evt_${uuid.replace(/-/g, '').slice(0, 12)}`;
}
```

### 8.2 Format Timestamp

```javascript
/**
 * Format ISO timestamp for display.
 * @param {string} isoTimestamp - ISO8601 timestamp
 * @returns {string} Formatted time like "10:30:05 AM"
 */
function formatTime(isoTimestamp) {
  return new Date(isoTimestamp).toLocaleTimeString();
}
```

### 8.3 Risk Level Color

```javascript
/**
 * Get color for risk level display.
 * @param {string} label - Risk label from defender
 * @returns {string} Hex color code
 */
function getRiskColor(label) {
  const colors = {
    low: "#22c55e",      // green
    medium: "#f59e0b",   // amber
    high: "#ef4444",     // red
    critical: "#7c2d12"  // dark red
  };
  return colors[label] || "#6b7280";
}
```

### 8.4 Tactic Display Name

```javascript
/**
 * Convert tactic ID to display name.
 * @param {string} tactic - Tactic identifier
 * @returns {string} Human-readable name
 */
function formatTactic(tactic) {
  return tactic
    .split('_')
    .map(word => word.charAt(0).toUpperCase() + word.slice(1))
    .join(' ');
}
```

---

## 9) Retool-Specific Implementation Notes

### 9.1 Query Configuration

Create these queries in Retool:

1. **healthCheckAttacker** - GET, runs on page load
2. **healthCheckDefender** - GET, runs on page load
3. **loadScenarios** - GET, runs after health checks pass
4. **createSession** - POST, triggered by Start button
5. **nextCaller** - POST, triggered by Next Caller button
6. **postEventToDefender** - POST, called after nextCaller and sendReply
7. **sendReply** - Transformer that calls postEventToDefender with agent_turn
8. **pollDefender** - GET, runs on 1500ms interval when session is live
9. **recoverTranscript** - GET, called on page load if sessionId exists
10. **finalizeSession** - POST, triggered by End Call button

### 9.2 Timer for Polling

```javascript
// In Retool, create a Timer component with:
// Interval: 1500ms
// Condition: {{sessionStatus === "live" && sessionId !== null}}
// On trigger: pollDefender.trigger()
```

### 9.3 Conditional Button States

```javascript
// Start Session button
disabled: {{!servicesChecked || !attackerHealthy || !defenderHealthy || sessionStatus !== "idle"}}

// Next Caller button
disabled: {{sessionStatus !== "live" || scenarioComplete || isLoading}}

// Send Reply button
disabled: {{sessionStatus !== "live" || !agentReplyText.trim() || isLoading}}

// End Call button
disabled: {{sessionStatus !== "live" || isLoading}}
```

### 9.4 Suggestions Copy Button

Each suggestion card should have a copy button that:
1. Copies the suggestion text to clipboard
2. Optionally auto-fills the reply input

```javascript
// On copy button click
navigator.clipboard.writeText(suggestion.text);
// Optionally: agentReplyText.setValue(suggestion.text);
```

---

## 10) Testing Checklist

Before demo, verify:

- [ ] Health checks run on page load and show correct status
- [ ] Scenario dropdown populates from attacker API
- [ ] Start Session creates session and enables controls
- [ ] Next Caller plays audio and shows text in transcript
- [ ] Caller turn appears in defender state within 2 seconds
- [ ] Send Reply clears input and appears in transcript
- [ ] Defender suggestions update after each turn
- [ ] Score changes after trainee replies
- [ ] 409 SCENARIO_DONE shows completion banner and disables Next Caller
- [ ] End Call shows report with all sections
- [ ] Page refresh mid-session recovers transcript
- [ ] Service failure shows appropriate error message

---

## 11) External Tool Integrations

### 11.1 Tonic Fabricate for Parallel Development

If the Attacker or Defender services aren't ready yet, use [Tonic Fabricate](https://www.tonic.ai/products/fabricate) to create mock APIs so UI development can proceed independently.

**Creating Mock Endpoints:**

1. Go to [fabricate.tonic.ai](https://fabricate.tonic.ai) (free tier available)
2. Create a new project and generate a database with session/scenario data
3. Use Fabricate's Mock API feature to expose endpoints

**Example: Mock Attacker Service**

Prompt for Fabricate:
```
Create a mock API for a social engineering attack simulator with these endpoints:

GET /api/v1/scenarios
- Returns array of scenarios with: scenario_id, title, description, difficulty, line_count

POST /api/v1/sessions/{session_id}/next_caller
- Request body: { scenario_id: string }
- Returns: { session_id, scenario_id, line_index, caller_text, tactics: string[], 
  audio_url: null, lines_remaining: number }
- After 6 calls with same session_id, return 409 with error.code = "SCENARIO_DONE"

Generate realistic social engineering caller lines for the responses.
```

**Example: Mock Defender Service**

Prompt for Fabricate:
```
Create a mock API for a social engineering defense copilot with these endpoints:

POST /api/v1/sessions
- Returns: { session_id, scenario_id, status: "created", created_at }

POST /api/v1/sessions/{session_id}/events  
- Always returns: { accepted: true, events_processed: 1, session_status: "live" }

GET /api/v1/sessions/{session_id}
- Returns realistic defender state with risk assessment, tactics_detected, 
  3 suggestions, score object, and near_misses array
- Vary the responses to simulate different states

POST /api/v1/sessions/{session_id}/finalize
- Returns complete report with tactics_used_summary, score, coach_notes, grade
```

**Switching from Mock to Real:**

In Retool, use environment variables to switch:
```javascript
// Development (mock)
const ATTACKER_BASE_URL = "https://your-fabricate-mock.tonic.ai";
const DEFENDER_BASE_URL = "https://your-fabricate-mock.tonic.ai";

// Production (real services)
const ATTACKER_BASE_URL = "http://localhost:8001";
const DEFENDER_BASE_URL = "http://localhost:8002";
```

This enables the UI engineer to build and test the entire flow before backend services are ready.

### 11.2 Modulate.ai Enhanced Features (Future)

If Modulate.ai integration is enabled in the Defender service (stretch goal), the UI should display additional data:

**Extended Risk Panel:**
```
┌──────────────────────────────────────────────────────────────────┐
│  RISK: ████████░░ HIGH                                           │
│                                                                  │
│  Modulate Analysis:                                              │
│  • Fraud Probability: 78%                                        │
│  • Deception Indicators: evasive responses, inconsistent story   │
│  • Emotion: High stress (0.82), Urgency (0.91)                   │
│                                                                  │
│  ⚠️ SYNTHETIC VOICE DETECTED                                     │
│  This caller may be using AI-generated audio.                    │
└──────────────────────────────────────────────────────────────────┘
```

**Conditional Rendering:**

```javascript
// Only show Modulate section if data is present
{defenderState.modulate_indicators && (
  <ModulateAnalysisPanel 
    fraudProbability={defenderState.fraud_probability}
    deceptionIndicators={defenderState.modulate_indicators}
    emotion={defenderState.emotion}
    isSyntheticVoice={defenderState.synthetic_voice_detected}
  />
)}
```

**Note for MVP:** The Modulate integration is a stretch goal. For hackathon MVP, the Defender will only return rules-based analysis. The UI should gracefully handle missing Modulate fields by simply not rendering those sections.

---

## 12) Stretch Goals (Only If MVP Complete)

1. **Auth0 login:** Gate access to training console, show user info
2. **Multi-page UI:** Separate pages for training, replay review, policy management
3. **Trainee audio:** Generate TTS for trainee replies using ElevenLabs
4. **Timeline visualization:** Graphical replay of the session with tactic markers
5. **Leaderboard:** Track and display trainee scores across sessions
6. **Export:** Download report as PDF or share via link

---

END OF UI SPEC.
