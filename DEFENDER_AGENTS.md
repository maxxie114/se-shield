# DEFENDER_AGENTS.md — Blue-Team Copilot (Analysis Engine + API for Retool)

## 0) Owner + Goal

**Owner:** Defender engineer  
**Goal:** Build the core copilot service that:
- Owns session state and is the **single source of truth** for the UI
- Owns the **canonical turn_index** (not Attacker, not UI)
- Ingests events from UI (caller turns, agent turns, scenario completion)
- Runs analysis per turn (tactic detection, risk assessment, suggestions, scoring)
- Exposes stable endpoints for Retool to poll
- Generates end-of-session reports

**Before you start:** Read SHARED_CONTRACTS.md for the canonical Tactic enum, error format, event types, and session status definitions.

---

## 1) MVP Deliverables (Target: < 1 hour)

### Must Have
1. Health and version endpoints (`GET /health`, `GET /version`)
2. Create session endpoint (`POST /api/v1/sessions`)
3. Ingest events endpoint (`POST /api/v1/sessions/{session_id}/events`)
4. Poll session state endpoint (`GET /api/v1/sessions/{session_id}`)
5. Get events/transcript endpoint (`GET /api/v1/sessions/{session_id}/events`)
6. Finalize session endpoint (`POST /api/v1/sessions/{session_id}/finalize`)
7. API key validation on all `/api/v1/*` endpoints
8. Rules-based tactic detection (keyword matching)
9. Template-based safe reply suggestions (always exactly 3)
10. Scoring with all four dimensions defined
11. Event idempotency via `event_id`

### Nice to Have (Stretch Goals)
- Ray parallelization for analysis tasks
- LLM-based tactic detection and coaching
- Policy version management
- Auth0 role gating

---

## 2) Core Data Model

```python
from dataclasses import dataclass, field
from datetime import datetime
from typing import Optional
from enum import Enum

class SessionStatus(Enum):
    """Lifecycle states for a training session."""
    CREATED = "created"
    LIVE = "live"
    COMPLETED = "completed"
    ABANDONED = "abandoned"

class EventType(Enum):
    """Types of events the defender can receive."""
    CALLER_TURN = "caller_turn"
    AGENT_TURN = "agent_turn"
    SCENARIO_COMPLETE = "scenario_complete"

@dataclass
class Event:
    """
    A single event in the session timeline.
    
    Attributes:
        event_id: Client-generated UUID for idempotency
        event_type: Type of event (caller_turn, agent_turn, scenario_complete)
        turn_index: Defender-assigned turn number
        timestamp: When the event occurred (client-provided)
        received_at: When the defender received the event
        text: Content of the turn (empty for scenario_complete)
        tactics: Tactics used/detected in this turn
    """
    event_id: str
    event_type: EventType
    turn_index: int
    timestamp: str
    received_at: str
    text: str = ""
    tactics: list[str] = field(default_factory=list)

@dataclass
class NearMiss:
    """
    A moment where the trainee almost violated policy.
    
    Attributes:
        turn_index: Which turn the near-miss occurred on
        event_id: The event that triggered the near-miss
        reason: Human-readable explanation
        severity: How serious the near-miss was
        pattern_matched: The specific pattern that triggered detection
    """
    turn_index: int
    event_id: str
    reason: str
    severity: str  # "low", "medium", "high"
    pattern_matched: str

@dataclass
class Suggestion:
    """
    A safe reply suggestion for the trainee.
    
    Attributes:
        label: Category of suggestion
        text: The suggested response text
    """
    label: str  # "policy_safe", "deescalate", "boundary_redirect"
    text: str

@dataclass
class RiskAssessment:
    """
    Current risk level assessment.
    
    Attributes:
        label: Overall risk level
        escalation_score: Numeric risk score (0.0 to 1.0)
        reasons: List of factors contributing to the risk level
    """
    label: str  # "low", "medium", "high", "critical"
    escalation_score: float
    reasons: list[str] = field(default_factory=list)

@dataclass
class Score:
    """
    Multi-dimensional scoring of trainee performance.
    
    Attributes:
        overall: Combined score (0-100)
        leak_risk: Risk of information disclosure (0-100, higher is better)
        policy_adherence: How well trainee followed policy (0-100)
        recognition: How well trainee recognized manipulation (0-100)
        notes: Explanatory notes for the scores
    """
    overall: int
    leak_risk: int
    policy_adherence: int
    recognition: int
    notes: list[str] = field(default_factory=list)

@dataclass
class Session:
    """
    Complete state for a training session.
    
    Attributes:
        session_id: Unique identifier
        scenario_id: Which scenario is being run
        status: Current lifecycle status
        created_at: When the session was created
        updated_at: When the session was last modified
        events: Ordered list of all events
        seen_event_ids: Set of event_ids for idempotency checking
        current_turn_index: Next turn number to assign
        tactics_detected: All tactics detected so far
        tactic_counts: Count of each tactic type
        near_misses: All near-misses detected
        latest_risk: Most recent risk assessment
        latest_suggestions: Current suggestions (always 3)
        latest_score: Current score
    """
    session_id: str
    scenario_id: str
    status: SessionStatus
    created_at: str
    updated_at: str
    events: list[Event] = field(default_factory=list)
    seen_event_ids: set[str] = field(default_factory=set)
    current_turn_index: int = 0
    tactics_detected: list[str] = field(default_factory=list)
    tactic_counts: dict[str, int] = field(default_factory=dict)
    near_misses: list[NearMiss] = field(default_factory=list)
    latest_risk: Optional[RiskAssessment] = None
    latest_suggestions: list[Suggestion] = field(default_factory=list)
    latest_score: Optional[Score] = None
```

---

## 3) API Contract

Base URL: `DEFENDER_BASE_URL` (e.g., `http://localhost:8002`)

All `/api/v1/*` endpoints require the `X-API-Key` header matching the `API_KEY` environment variable.

---

### 3.1 Health Check

**GET /health**

No authentication required.

Response 200:
```json
{
  "status": "ok",
  "service": "defender",
  "active_sessions": 3,
  "timestamp": "2024-01-15T10:30:00Z"
}
```

---

### 3.2 Version

**GET /version**

No authentication required.

Response 200:
```json
{
  "version": "0.1.0",
  "commit": "abc123f",
  "built_at": "2024-01-15T10:00:00Z"
}
```

---

### 3.3 Create Session

**POST /api/v1/sessions**

Creates a new training session. Returns the `session_id` that must be used for all subsequent calls.

Request:
```json
{
  "scenario_id": "ceo_impersonation_001",
  "metadata": {
    "trainee_name": "John Doe",
    "department": "Customer Support"
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

---

### 3.4 Ingest Events

**POST /api/v1/sessions/{session_id}/events**

Submit one or more events to the session. The Defender will:
1. Validate each event
2. Check for duplicate `event_id` (return 409 if seen)
3. Assign `turn_index` for caller_turn events
4. Run analysis and update session state

Request:
```json
{
  "events": [
    {
      "event_id": "evt_a1b2c3d4",
      "type": "caller_turn",
      "timestamp": "2024-01-15T10:30:05Z",
      "text": "Hi, this is the CEO. I need you to reset my MFA right now.",
      "tactics": ["authority_impersonation", "urgency_pressure"]
    }
  ]
}
```

**Event fields:**
- `event_id` (required): Client-generated UUID for idempotency
- `type` (required): One of `caller_turn`, `agent_turn`, `scenario_complete`
- `timestamp` (required): ISO8601 timestamp of when the event occurred
- `text` (required for caller_turn/agent_turn): Content of the turn
- `tactics` (optional): Tactics hint from attacker (used for scoring, not detection)

Response 202 (accepted):
```json
{
  "accepted": true,
  "events_processed": 1,
  "session_status": "live",
  "updated_at": "2024-01-15T10:30:05Z"
}
```

Response 404 (session not found):
```json
{
  "error": {
    "code": "SESSION_NOT_FOUND",
    "message": "Session 'sess_unknown' does not exist"
  }
}
```

Response 409 (duplicate event):
```json
{
  "error": {
    "code": "DUPLICATE_EVENT",
    "message": "Event 'evt_a1b2c3d4' has already been processed"
  }
}
```

Response 400 (session not live):
```json
{
  "error": {
    "code": "SESSION_NOT_LIVE",
    "message": "Session is completed and cannot accept new events"
  }
}
```

**Turn Index Assignment Rules:**
1. When a `caller_turn` event is received, increment `current_turn_index` and assign it to the event
2. When an `agent_turn` event is received, assign the current `current_turn_index` (agent replies to the most recent caller turn)
3. When a `scenario_complete` event is received, do not change turn_index; mark session for finalization

---

### 3.5 Poll Session State

**GET /api/v1/sessions/{session_id}**

Returns the current session state. This is what Retool polls every 1-2 seconds.

Optional query parameter:
- `since`: ISO8601 timestamp. If `updated_at` is not newer than `since`, returns 304 Not Modified (saves bandwidth).

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
    "reasons": [
      "Authority impersonation detected",
      "Urgency pressure detected",
      "Identity bypass attempted"
    ]
  },
  "tactics_detected": [
    "authority_impersonation",
    "urgency_pressure",
    "identity_bypass"
  ],
  "suggestions": [
    {
      "label": "policy_safe",
      "text": "I can help, but I need to verify your identity first. Can you confirm your employee ID and answer your security question?"
    },
    {
      "label": "deescalate",
      "text": "I understand this is urgent. I want to help you as quickly as possible while keeping your account secure. Let's start the verification process."
    },
    {
      "label": "boundary_redirect",
      "text": "I'm not able to bypass verification, but I can escalate this to a supervisor who may have additional options. Would you like me to do that?"
    }
  ],
  "score": {
    "overall": 72,
    "leak_risk": 85,
    "policy_adherence": 65,
    "recognition": 70,
    "notes": [
      "Good: Did not confirm account existence",
      "Warning: Consider asking for callback number"
    ]
  },
  "near_misses": [
    {
      "turn_index": 2,
      "event_id": "evt_x1y2z3",
      "reason": "Almost confirmed account details without verification",
      "severity": "medium",
      "pattern_matched": "account_existence_confirmation"
    }
  ]
}
```

Response 304 (not modified, when using `since` parameter):
No body.

Response 404 (session not found):
```json
{
  "error": {
    "code": "SESSION_NOT_FOUND",
    "message": "Session 'sess_unknown' does not exist"
  }
}
```

---

### 3.6 Get Events (Transcript Recovery)

**GET /api/v1/sessions/{session_id}/events**

Returns the full event log for a session. Used to reconstruct the transcript if the UI loses state (e.g., page refresh).

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
      "text": "Hello! I'd be happy to help. Before I can make any changes, I'll need to verify your identity. Can you provide your employee ID?",
      "tactics": []
    }
  ]
}
```

---

### 3.7 Finalize Session

**POST /api/v1/sessions/{session_id}/finalize**

Marks the session as completed and generates the final report.

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
    "scenario_title": "CEO Impersonation",
    "duration_seconds": 185,
    "total_turns": 6,
    "tactics_used_summary": [
      {"tactic": "authority_impersonation", "count": 3},
      {"tactic": "urgency_pressure", "count": 4},
      {"tactic": "identity_bypass", "count": 2},
      {"tactic": "threat_intimidation", "count": 1}
    ],
    "near_misses": [
      {
        "turn_index": 2,
        "reason": "Almost confirmed account details without verification",
        "severity": "medium"
      }
    ],
    "score": {
      "overall": 72,
      "leak_risk": 85,
      "policy_adherence": 65,
      "recognition": 70
    },
    "coach_notes": [
      "Strong start: Immediately asked for verification",
      "Good recovery after pressure escalated",
      "Consider: Offer callback option earlier when caller refuses verification",
      "Watch for: Account existence confirmation (almost slipped on turn 2)"
    ],
    "grade": "B",
    "passed": true
  }
}
```

---

## 4) Analysis Engine (MVP: Rules-Based)

### 4.1 Tactic Detection

Detect tactics by scanning the latest caller turn text for keywords/phrases. This runs **in addition to** any `tactics` hint provided by the Attacker—we detect independently to verify.

```python
# Tactic detection rules (case-insensitive matching)
TACTIC_PATTERNS: dict[str, list[str]] = {
    "authority_impersonation": [
        "ceo", "cfo", "cto", "coo", "president", "vice president", "vp",
        "director", "boss", "executive", "c-suite", "board member",
        "i'm from it", "security team", "compliance", "legal department",
        "this is the", "i am the"
    ],
    "urgency_pressure": [
        "right now", "immediately", "urgent", "asap", "emergency",
        "time sensitive", "board meeting", "critical", "can't wait",
        "need this done", "hurry", "quickly"
    ],
    "credential_harvesting": [
        "password", "otp", "one-time", "verification code", "2fa", "mfa",
        "authenticator", "security code", "pin", "passcode", "token"
    ],
    "identity_bypass": [
        "skip verification", "don't need that", "verify later",
        "just do it", "bypass", "make an exception", "this once",
        "trust me", "you know who i am"
    ],
    "threat_intimidation": [
        "fired", "report you", "your manager", "hr will hear",
        "lawsuit", "compliance", "consequences", "trouble",
        "disciplinary", "write you up"
    ],
    "emotional_manipulation": [
        "please help", "desperate", "family emergency", "sick",
        "dying", "hospital", "crying", "begging", "only one who can"
    ],
    "information_probing": [
        "what do you see", "tell me about my account", "how much",
        "balance", "transactions", "activity", "who accessed"
    ],
    "callback_evasion": [
        "can't take calls", "don't call back", "just email",
        "no callback", "i'll call you", "not available by phone"
    ]
}

def detect_tactics(text: str) -> list[str]:
    """
    Scan text for tactic patterns and return list of detected tactics.
    
    Args:
        text: The text to analyze (usually caller turn)
        
    Returns:
        List of tactic identifiers that were detected
    """
    text_lower = text.lower()
    detected = []
    
    for tactic, patterns in TACTIC_PATTERNS.items():
        for pattern in patterns:
            if pattern in text_lower:
                detected.append(tactic)
                break  # Only count each tactic once per turn
    
    return detected
```

### 4.2 Risk Assessment

```python
def assess_risk(
    tactics_detected: list[str],
    tactic_counts: dict[str, int],
    near_misses: list[NearMiss]
) -> RiskAssessment:
    """
    Calculate current risk level based on detected tactics and near-misses.
    
    Args:
        tactics_detected: All unique tactics detected in this session
        tactic_counts: Count of each tactic across all turns
        near_misses: List of near-miss incidents
        
    Returns:
        RiskAssessment with label, score, and reasons
    """
    reasons = []
    score = 0.0
    
    # High-severity tactics add more to score
    high_severity = ["credential_harvesting", "identity_bypass", "threat_intimidation"]
    medium_severity = ["authority_impersonation", "urgency_pressure", "information_probing"]
    
    for tactic in tactics_detected:
        if tactic in high_severity:
            score += 0.25
            reasons.append(f"{tactic.replace('_', ' ').title()} detected")
        elif tactic in medium_severity:
            score += 0.15
            reasons.append(f"{tactic.replace('_', ' ').title()} detected")
        else:
            score += 0.10
    
    # Near-misses increase risk significantly
    high_misses = sum(1 for nm in near_misses if nm.severity == "high")
    medium_misses = sum(1 for nm in near_misses if nm.severity == "medium")
    
    score += high_misses * 0.20
    score += medium_misses * 0.10
    
    if high_misses > 0:
        reasons.append(f"{high_misses} high-severity near-miss(es)")
    if medium_misses > 0:
        reasons.append(f"{medium_misses} medium-severity near-miss(es)")
    
    # Clamp score
    score = min(score, 1.0)
    
    # Determine label
    if score >= 0.75:
        label = "critical"
    elif score >= 0.50:
        label = "high"
    elif score >= 0.25:
        label = "medium"
    else:
        label = "low"
    
    return RiskAssessment(label=label, escalation_score=score, reasons=reasons)
```

### 4.3 Safe Reply Suggestions

**Always return exactly 3 suggestions.** The UI layout depends on this.

```python
def generate_suggestions(
    tactics_detected: list[str],
    latest_caller_text: str
) -> list[Suggestion]:
    """
    Generate exactly 3 safe reply suggestions based on current context.
    
    Args:
        tactics_detected: Tactics detected in the current/recent turns
        latest_caller_text: The most recent caller statement
        
    Returns:
        Exactly 3 Suggestion objects
    """
    # Default templates (always return these 3 categories)
    suggestions = [
        Suggestion(
            label="policy_safe",
            text="I can help, but I need to verify your identity first. "
                 "Can you confirm your employee ID and answer your security question?"
        ),
        Suggestion(
            label="deescalate",
            text="I understand this is urgent. I want to help you as quickly as possible "
                 "while keeping your account secure. Let's start the verification process."
        ),
        Suggestion(
            label="boundary_redirect",
            text="I'm not able to bypass verification, but I can escalate this to a supervisor "
                 "who may have additional options. Would you like me to do that?"
        ),
    ]
    
    # Customize based on detected tactics
    if "threat_intimidation" in tactics_detected:
        suggestions[1] = Suggestion(
            label="deescalate",
            text="I hear that this is frustrating. I want to help resolve this for you. "
                 "Our verification process protects your account—let's work through it together."
        )
    
    if "callback_evasion" in tactics_detected:
        suggestions[2] = Suggestion(
            label="boundary_redirect",
            text="For security, I can arrange a verified callback using the phone number on file. "
                 "This protects both of us. Would you like me to set that up?"
        )
    
    if "credential_harvesting" in tactics_detected:
        suggestions[0] = Suggestion(
            label="policy_safe",
            text="I can't accept verification codes over the phone. "
                 "Let me walk you through the self-service reset process instead."
        )
    
    return suggestions
```

### 4.4 Near-Miss Detection

Scan agent replies for patterns that indicate policy violations or close calls.

```python
NEAR_MISS_PATTERNS: dict[str, dict] = {
    "credential_disclosure": {
        "patterns": [
            "your password is", "the code is", "i'll tell you the",
            "here's your otp", "the verification code"
        ],
        "severity": "high",
        "reason": "Disclosed or offered to disclose credentials"
    },
    "account_existence_confirmation": {
        "patterns": [
            "yes i see your account", "i can see you have",
            "your account shows", "looking at your profile",
            "i found your account"
        ],
        "severity": "medium",
        "reason": "Confirmed account existence without verification"
    },
    "verification_bypass_agreement": {
        "patterns": [
            "i can make an exception", "just this once",
            "i'll skip verification", "without verifying",
            "i'll reset it without"
        ],
        "severity": "high",
        "reason": "Agreed to bypass verification process"
    },
    "sensitive_info_disclosure": {
        "patterns": [
            "your balance is", "last transaction was",
            "your address on file", "the card ending in"
        ],
        "severity": "high",
        "reason": "Disclosed sensitive account information"
    },
    "excessive_trust": {
        "patterns": [
            "i believe you", "i trust that you're",
            "that makes sense", "of course you are"
        ],
        "severity": "low",
        "reason": "Expressed trust without verification"
    }
}

def detect_near_misses(
    agent_text: str,
    turn_index: int,
    event_id: str
) -> list[NearMiss]:
    """
    Scan agent reply for near-miss patterns.
    
    Args:
        agent_text: The trainee's response text
        turn_index: Which turn this occurred on
        event_id: The event ID for reference
        
    Returns:
        List of NearMiss objects for any patterns found
    """
    text_lower = agent_text.lower()
    near_misses = []
    
    for pattern_name, config in NEAR_MISS_PATTERNS.items():
        for pattern in config["patterns"]:
            if pattern in text_lower:
                near_misses.append(NearMiss(
                    turn_index=turn_index,
                    event_id=event_id,
                    reason=config["reason"],
                    severity=config["severity"],
                    pattern_matched=pattern_name
                ))
                break  # Only one near-miss per pattern type
    
    return near_misses
```

### 4.5 Scoring

All four dimensions must be calculated and returned.

```python
def calculate_score(
    events: list[Event],
    near_misses: list[NearMiss],
    tactics_detected: list[str]
) -> Score:
    """
    Calculate multi-dimensional score for trainee performance.
    
    Dimensions:
    - overall: Combined score (weighted average)
    - leak_risk: How well they protected information (higher = better)
    - policy_adherence: How well they followed procedures (higher = better)
    - recognition: How well they recognized manipulation (higher = better)
    
    Args:
        events: All events in the session
        near_misses: All detected near-misses
        tactics_detected: All tactics that appeared in the scenario
        
    Returns:
        Score object with all dimensions
    """
    notes = []
    
    # Start all dimensions at 100
    leak_risk = 100
    policy_adherence = 100
    recognition = 100
    
    # === Leak Risk ===
    # Penalize for near-misses related to information disclosure
    leak_related = ["credential_disclosure", "sensitive_info_disclosure", "account_existence_confirmation"]
    for nm in near_misses:
        if nm.pattern_matched in leak_related:
            if nm.severity == "high":
                leak_risk -= 30
                notes.append(f"Major leak risk: {nm.reason}")
            elif nm.severity == "medium":
                leak_risk -= 15
                notes.append(f"Moderate leak risk: {nm.reason}")
            else:
                leak_risk -= 5
    
    # === Policy Adherence ===
    # Penalize for verification bypass
    policy_related = ["verification_bypass_agreement", "excessive_trust"]
    for nm in near_misses:
        if nm.pattern_matched in policy_related:
            if nm.severity == "high":
                policy_adherence -= 35
                notes.append(f"Policy violation: {nm.reason}")
            elif nm.severity == "medium":
                policy_adherence -= 15
            else:
                policy_adherence -= 5
    
    # Bonus for using safe patterns
    agent_events = [e for e in events if e.event_type == EventType.AGENT_TURN]
    for event in agent_events:
        text_lower = event.text.lower()
        if any(phrase in text_lower for phrase in ["verify", "verification", "confirm your"]):
            policy_adherence = min(100, policy_adherence + 5)
            if "Asked for verification" not in notes:
                notes.append("Good: Asked for verification")
        if any(phrase in text_lower for phrase in ["supervisor", "escalate", "manager"]):
            policy_adherence = min(100, policy_adherence + 5)
            if "Offered escalation" not in notes:
                notes.append("Good: Offered escalation path")
    
    # === Recognition ===
    # Based on how quickly agent responds appropriately to high-risk tactics
    high_risk_tactics = ["credential_harvesting", "identity_bypass", "threat_intimidation"]
    high_risk_present = any(t in tactics_detected for t in high_risk_tactics)
    
    if high_risk_present:
        # Check if agent maintained boundaries
        maintained_boundaries = not any(
            nm.pattern_matched == "verification_bypass_agreement" 
            for nm in near_misses
        )
        if maintained_boundaries:
            notes.append("Strong: Maintained boundaries under pressure")
        else:
            recognition -= 25
    
    # === Overall (weighted average) ===
    overall = int(
        (leak_risk * 0.35) +
        (policy_adherence * 0.40) +
        (recognition * 0.25)
    )
    
    # Clamp all values
    leak_risk = max(0, min(100, leak_risk))
    policy_adherence = max(0, min(100, policy_adherence))
    recognition = max(0, min(100, recognition))
    overall = max(0, min(100, overall))
    
    return Score(
        overall=overall,
        leak_risk=leak_risk,
        policy_adherence=policy_adherence,
        recognition=recognition,
        notes=notes
    )
```

---

## 5) Ray Parallelization (STRETCH GOAL)

**Ray is NOT required for MVP.** Implement synchronous analysis first. Only add Ray if MVP is complete and working.

### If You Add Ray

Use Ray to parallelize the three analysis tasks:

```python
import ray
from typing import Optional

# Initialize Ray (if enabled)
USE_RAY = os.environ.get("USE_RAY", "false").lower() == "true"

if USE_RAY:
    ray.init(ignore_reinit_error=True)
    
    @ray.remote
    def detect_tactics_task(text: str) -> list[str]:
        return detect_tactics(text)
    
    @ray.remote
    def generate_suggestions_task(tactics: list[str], text: str) -> list[Suggestion]:
        return generate_suggestions(tactics, text)
    
    @ray.remote
    def calculate_score_task(events: list, near_misses: list, tactics: list) -> Score:
        return calculate_score(events, near_misses, tactics)

def run_analysis(session: Session, latest_caller_text: str) -> None:
    """
    Run all analysis tasks (parallel if Ray enabled, sequential otherwise).
    Updates session state in place.
    """
    if USE_RAY:
        # Parallel execution
        tactics_future = detect_tactics_task.remote(latest_caller_text)
        
        new_tactics = ray.get(tactics_future)
        
        suggestions_future = generate_suggestions_task.remote(new_tactics, latest_caller_text)
        score_future = calculate_score_task.remote(session.events, session.near_misses, session.tactics_detected)
        
        session.latest_suggestions = ray.get(suggestions_future)
        session.latest_score = ray.get(score_future)
    else:
        # Sequential execution (MVP)
        new_tactics = detect_tactics(latest_caller_text)
        session.latest_suggestions = generate_suggestions(new_tactics, latest_caller_text)
        session.latest_score = calculate_score(session.events, session.near_misses, session.tactics_detected)
    
    # Update session tactics
    for tactic in new_tactics:
        if tactic not in session.tactics_detected:
            session.tactics_detected.append(tactic)
        session.tactic_counts[tactic] = session.tactic_counts.get(tactic, 0) + 1
    
    # Update risk assessment
    session.latest_risk = assess_risk(session.tactics_detected, session.tactic_counts, session.near_misses)
```

### Hard Timeouts

If using Ray, add timeouts to prevent hanging:

```python
try:
    result = ray.get(future, timeout=5.0)  # 5 second timeout
except ray.exceptions.GetTimeoutError:
    # Fall back to synchronous
    result = synchronous_fallback()
```

---

## 6) Environment Variables

```bash
API_KEY=your-shared-secret         # Required: For X-API-Key validation
USE_RAY=false                      # Optional: Enable Ray parallelization
RAY_ADDRESS=auto                   # Optional: Ray cluster address
```

---

## 7) Implementation Skeleton

```python
"""
SE Shield Defender Service
Provides session management, analysis, and coaching for SE training.
"""

import os
import uuid
from datetime import datetime, timezone
from typing import Optional
from dataclasses import dataclass, field, asdict

from fastapi import FastAPI, HTTPException, Header, Depends, Query
from fastapi.responses import Response
from pydantic import BaseModel

app = FastAPI(title="SE Shield Defender Service", version="0.1.0")

# ============================================================================
# Configuration
# ============================================================================

API_KEY = os.environ.get("API_KEY", "dev-api-key")

# ============================================================================
# State (in-memory for MVP)
# ============================================================================

sessions: dict[str, Session] = {}

# ============================================================================
# Auth Dependency
# ============================================================================

def verify_api_key(x_api_key: str = Header(...)) -> str:
    """Validate the X-API-Key header against configured secret."""
    if x_api_key != API_KEY:
        raise HTTPException(status_code=401, detail={
            "error": {"code": "UNAUTHORIZED", "message": "Invalid API key"}
        })
    return x_api_key

# ============================================================================
# API Endpoints
# ============================================================================

@app.get("/health")
async def health_check():
    """Health check endpoint (no auth required)."""
    return {
        "status": "ok",
        "service": "defender",
        "active_sessions": len([s for s in sessions.values() if s.status == SessionStatus.LIVE]),
        "timestamp": datetime.now(timezone.utc).isoformat(),
    }

@app.get("/version")
async def version():
    """Version endpoint (no auth required)."""
    return {
        "version": "0.1.0",
        "commit": os.environ.get("GIT_COMMIT", "unknown"),
        "built_at": os.environ.get("BUILD_TIME", "unknown"),
    }

class CreateSessionRequest(BaseModel):
    """Request body for session creation."""
    scenario_id: str
    metadata: dict = {}

@app.post("/api/v1/sessions", status_code=201, dependencies=[Depends(verify_api_key)])
async def create_session(request: CreateSessionRequest):
    """Create a new training session."""
    session_id = f"sess_{uuid.uuid4().hex[:12]}"
    now = datetime.now(timezone.utc).isoformat()
    
    session = Session(
        session_id=session_id,
        scenario_id=request.scenario_id,
        status=SessionStatus.CREATED,
        created_at=now,
        updated_at=now,
    )
    
    # Initialize with default values
    session.latest_risk = RiskAssessment(label="low", escalation_score=0.0, reasons=[])
    session.latest_suggestions = generate_suggestions([], "")
    session.latest_score = Score(overall=100, leak_risk=100, policy_adherence=100, recognition=100, notes=[])
    
    sessions[session_id] = session
    
    return {
        "session_id": session_id,
        "scenario_id": request.scenario_id,
        "status": session.status.value,
        "created_at": now,
    }

class EventInput(BaseModel):
    """Single event in the events request."""
    event_id: str
    type: str
    timestamp: str
    text: str = ""
    tactics: list[str] = []

class IngestEventsRequest(BaseModel):
    """Request body for event ingestion."""
    events: list[EventInput]

@app.post("/api/v1/sessions/{session_id}/events", status_code=202, dependencies=[Depends(verify_api_key)])
async def ingest_events(session_id: str, request: IngestEventsRequest):
    """Ingest events and trigger analysis."""
    if session_id not in sessions:
        raise HTTPException(status_code=404, detail={
            "error": {"code": "SESSION_NOT_FOUND", "message": f"Session '{session_id}' does not exist"}
        })
    
    session = sessions[session_id]
    
    if session.status not in [SessionStatus.CREATED, SessionStatus.LIVE]:
        raise HTTPException(status_code=400, detail={
            "error": {"code": "SESSION_NOT_LIVE", "message": "Session is completed and cannot accept new events"}
        })
    
    now = datetime.now(timezone.utc).isoformat()
    processed = 0
    
    for event_input in request.events:
        # Check idempotency
        if event_input.event_id in session.seen_event_ids:
            raise HTTPException(status_code=409, detail={
                "error": {"code": "DUPLICATE_EVENT", "message": f"Event '{event_input.event_id}' has already been processed"}
            })
        
        # Parse event type
        try:
            event_type = EventType(event_input.type)
        except ValueError:
            raise HTTPException(status_code=400, detail={
                "error": {"code": "INVALID_EVENT_TYPE", "message": f"Unknown event type '{event_input.type}'"}
            })
        
        # Assign turn index
        if event_type == EventType.CALLER_TURN:
            session.current_turn_index += 1
        
        turn_index = session.current_turn_index
        
        # Create event
        event = Event(
            event_id=event_input.event_id,
            event_type=event_type,
            turn_index=turn_index,
            timestamp=event_input.timestamp,
            received_at=now,
            text=event_input.text,
            tactics=event_input.tactics,
        )
        
        session.events.append(event)
        session.seen_event_ids.add(event_input.event_id)
        
        # Update status
        if session.status == SessionStatus.CREATED:
            session.status = SessionStatus.LIVE
        
        # Handle scenario_complete
        if event_type == EventType.SCENARIO_COMPLETE:
            session.status = SessionStatus.COMPLETED
        
        # Run analysis
        if event_type == EventType.CALLER_TURN:
            # Detect tactics in caller text
            detected = detect_tactics(event.text)
            for tactic in detected:
                if tactic not in session.tactics_detected:
                    session.tactics_detected.append(tactic)
                session.tactic_counts[tactic] = session.tactic_counts.get(tactic, 0) + 1
            
            # Update suggestions and risk
            session.latest_suggestions = generate_suggestions(session.tactics_detected, event.text)
            session.latest_risk = assess_risk(session.tactics_detected, session.tactic_counts, session.near_misses)
        
        elif event_type == EventType.AGENT_TURN:
            # Check for near-misses
            new_near_misses = detect_near_misses(event.text, turn_index, event.event_id)
            session.near_misses.extend(new_near_misses)
            
            # Recalculate score
            session.latest_score = calculate_score(session.events, session.near_misses, session.tactics_detected)
            session.latest_risk = assess_risk(session.tactics_detected, session.tactic_counts, session.near_misses)
        
        processed += 1
    
    session.updated_at = now
    
    return {
        "accepted": True,
        "events_processed": processed,
        "session_status": session.status.value,
        "updated_at": now,
    }

@app.get("/api/v1/sessions/{session_id}", dependencies=[Depends(verify_api_key)])
async def get_session(
    session_id: str,
    since: Optional[str] = Query(None, description="Return 304 if not modified since this timestamp")
):
    """Get current session state (for polling)."""
    if session_id not in sessions:
        raise HTTPException(status_code=404, detail={
            "error": {"code": "SESSION_NOT_FOUND", "message": f"Session '{session_id}' does not exist"}
        })
    
    session = sessions[session_id]
    
    # Check if modified since
    if since and session.updated_at <= since:
        return Response(status_code=304)
    
    return {
        "session_id": session.session_id,
        "scenario_id": session.scenario_id,
        "status": session.status.value,
        "updated_at": session.updated_at,
        "current_turn_index": session.current_turn_index,
        "risk": {
            "label": session.latest_risk.label,
            "escalation_score": session.latest_risk.escalation_score,
            "reasons": session.latest_risk.reasons,
        } if session.latest_risk else None,
        "tactics_detected": session.tactics_detected,
        "suggestions": [
            {"label": s.label, "text": s.text}
            for s in session.latest_suggestions
        ],
        "score": {
            "overall": session.latest_score.overall,
            "leak_risk": session.latest_score.leak_risk,
            "policy_adherence": session.latest_score.policy_adherence,
            "recognition": session.latest_score.recognition,
            "notes": session.latest_score.notes,
        } if session.latest_score else None,
        "near_misses": [
            {
                "turn_index": nm.turn_index,
                "event_id": nm.event_id,
                "reason": nm.reason,
                "severity": nm.severity,
                "pattern_matched": nm.pattern_matched,
            }
            for nm in session.near_misses
        ],
    }

@app.get("/api/v1/sessions/{session_id}/events", dependencies=[Depends(verify_api_key)])
async def get_events(session_id: str):
    """Get all events for transcript recovery."""
    if session_id not in sessions:
        raise HTTPException(status_code=404, detail={
            "error": {"code": "SESSION_NOT_FOUND", "message": f"Session '{session_id}' does not exist"}
        })
    
    session = sessions[session_id]
    
    return {
        "session_id": session.session_id,
        "events": [
            {
                "event_id": e.event_id,
                "type": e.event_type.value,
                "turn_index": e.turn_index,
                "timestamp": e.timestamp,
                "text": e.text,
                "tactics": e.tactics,
            }
            for e in session.events
        ],
    }

class FinalizeRequest(BaseModel):
    """Request body for session finalization."""
    include_report: bool = True

@app.post("/api/v1/sessions/{session_id}/finalize", dependencies=[Depends(verify_api_key)])
async def finalize_session(session_id: str, request: FinalizeRequest):
    """Finalize session and generate report."""
    if session_id not in sessions:
        raise HTTPException(status_code=404, detail={
            "error": {"code": "SESSION_NOT_FOUND", "message": f"Session '{session_id}' does not exist"}
        })
    
    session = sessions[session_id]
    session.status = SessionStatus.COMPLETED
    session.updated_at = datetime.now(timezone.utc).isoformat()
    
    # Calculate duration
    if session.events:
        first_ts = session.events[0].timestamp
        last_ts = session.events[-1].timestamp
        # Simple duration calculation (would need proper parsing in production)
        duration = len(session.events) * 15  # Rough estimate: 15 seconds per turn
    else:
        duration = 0
    
    # Generate coach notes
    coach_notes = session.latest_score.notes.copy() if session.latest_score else []
    
    if session.latest_score:
        if session.latest_score.overall >= 80:
            coach_notes.insert(0, "Excellent performance! Strong resistance to social engineering tactics.")
        elif session.latest_score.overall >= 60:
            coach_notes.insert(0, "Good performance with some areas for improvement.")
        else:
            coach_notes.insert(0, "Needs additional training on recognizing and resisting manipulation.")
    
    # Determine grade and pass/fail
    overall = session.latest_score.overall if session.latest_score else 0
    grade = "A" if overall >= 90 else "B" if overall >= 80 else "C" if overall >= 70 else "D" if overall >= 60 else "F"
    passed = overall >= 60
    
    response = {
        "session_id": session_id,
        "status": "completed",
    }
    
    if request.include_report:
        response["report"] = {
            "scenario_id": session.scenario_id,
            "duration_seconds": duration,
            "total_turns": session.current_turn_index,
            "tactics_used_summary": [
                {"tactic": tactic, "count": count}
                for tactic, count in sorted(session.tactic_counts.items(), key=lambda x: -x[1])
            ],
            "near_misses": [
                {
                    "turn_index": nm.turn_index,
                    "reason": nm.reason,
                    "severity": nm.severity,
                }
                for nm in session.near_misses
            ],
            "score": {
                "overall": session.latest_score.overall,
                "leak_risk": session.latest_score.leak_risk,
                "policy_adherence": session.latest_score.policy_adherence,
                "recognition": session.latest_score.recognition,
            } if session.latest_score else None,
            "coach_notes": coach_notes,
            "grade": grade,
            "passed": passed,
        }
    
    return response
```

---

## 8) External Tool Integrations

### 8.1 Modulate.ai Integration (STRETCH GOAL)

[Modulate.ai](https://www.modulate.ai/solutions/commercial-fraud) provides enterprise-grade voice fraud detection that could significantly enhance the Defender's analysis capabilities. Their platform analyzes content, intent, emotion, and behavior in real-time.

**Why Consider Modulate?**

The MVP rules-based detection has limitations:
- Keyword matching misses sophisticated manipulation
- No prosody/tone analysis (only text content)
- No emotion detection (stress, urgency, deception)
- Cannot detect synthetic/deepfake voices

Modulate addresses all of these with ML models trained on hundreds of millions of hours of audio.

**Integration Architecture:**

```python
# Feature flag to enable/disable Modulate integration
ENABLE_MODULATE = os.environ.get("ENABLE_MODULATE", "false").lower() == "true"
MODULATE_API_KEY = os.environ.get("MODULATE_API_KEY")

async def analyze_with_modulate(
    transcript: str,
    audio_url: str | None = None
) -> dict | None:
    """
    Call Modulate.ai API for enhanced fraud/manipulation detection.
    
    Args:
        transcript: Text transcript of the conversation
        audio_url: Optional URL to audio file for prosody analysis
        
    Returns:
        Modulate analysis results or None if disabled/failed
    """
    if not ENABLE_MODULATE or not MODULATE_API_KEY:
        return None
    
    try:
        # Hypothetical Modulate API call
        response = await httpx.post(
            "https://api.modulate.ai/v1/analyze",
            headers={"Authorization": f"Bearer {MODULATE_API_KEY}"},
            json={
                "transcript": transcript,
                "audio_url": audio_url,
                "detection_types": ["fraud", "manipulation", "deception", "synthetic_voice"]
            },
            timeout=5.0  # Hard timeout to not block the response
        )
        return response.json()
    except Exception as e:
        # Log but don't fail - Modulate is enhancement, not critical path
        print(f"Modulate API error (non-fatal): {e}")
        return None
```

**Merging Results:**

When Modulate is enabled, merge its results with rules-based detection:

```python
def merge_analysis_results(
    rules_result: dict,
    modulate_result: dict | None
) -> dict:
    """
    Combine rules-based and Modulate ML-based analysis.
    Modulate results enhance but don't replace rules-based detection.
    
    Args:
        rules_result: Output from rules-based tactic detection
        modulate_result: Output from Modulate API (may be None)
        
    Returns:
        Merged analysis with combined confidence
    """
    if modulate_result is None:
        return rules_result
    
    # Enhance risk assessment with Modulate's fraud probability
    if modulate_result.get("fraud_probability", 0) > 0.7:
        rules_result["risk"]["label"] = "critical"
        rules_result["risk"]["reasons"].append(
            f"Modulate: High fraud probability ({modulate_result['fraud_probability']:.0%})"
        )
    
    # Add emotion analysis if available
    if "emotion_analysis" in modulate_result:
        rules_result["emotion"] = modulate_result["emotion_analysis"]
    
    # Flag synthetic voice detection
    if modulate_result.get("is_synthetic_voice"):
        rules_result["synthetic_voice_detected"] = True
        rules_result["risk"]["reasons"].append("Synthetic/deepfake voice detected")
    
    # Add Modulate-specific deception indicators
    if modulate_result.get("deception_indicators"):
        rules_result["modulate_indicators"] = modulate_result["deception_indicators"]
    
    return rules_result
```

**When to Enable Modulate:**

| Scenario | Enable Modulate? |
|----------|-----------------|
| Hackathon demo | ❌ No - adds complexity and cost |
| Internal pilot | ⚠️ Maybe - if budget allows |
| Production deployment | ✅ Yes - significant accuracy improvement |
| Enterprise sales demo | ✅ Yes - demonstrates integration capability |

**Cost Considerations:**

Modulate pricing is enterprise-tier (contact for quote). For hackathon:
- MVP works fine without it
- Mention it in pitch as "production integration path"
- Architecture is designed to add it later without refactoring

### 8.2 Tonic Fabricate for Test Data

Use [Tonic Fabricate](https://www.tonic.ai/products/fabricate) to generate test session data for development and demos:

```
Generate 20 training session records for a social engineering defense platform.
Each session should include:
- session_id (UUID format)
- scenario_id (one of: ceo_impersonation_001, tech_support_002, account_takeover_003)
- 4-8 events alternating between caller_turn and agent_turn
- tactics_detected array with realistic combinations
- score object with overall (40-95), leak_risk, policy_adherence, recognition
- 0-3 near_misses with varying severity
- status: "completed"

Make some sessions show excellent trainee performance (score 85+) and 
some show poor performance (score below 60) with multiple near-misses.
```

This is useful for:
- Populating the UI with realistic demo data
- Testing the report generation without running full scenarios
- Generating edge cases for Defender validation

---

## 9) Stretch Goals (Only If MVP Complete)

1. **Auth0 role gating:** Different views for trainees vs supervisors
2. **LLM-based detection:** Replace keyword matching with Claude/GPT for more nuanced analysis
3. **Policy versions:** Allow supervisors to publish custom detection rules and suggestions
4. **Replay timeline:** Endpoint that returns events with analysis results per-turn for visualization
5. **Ray parallelization:** As documented in section 5

---

END OF DEFENDER SPEC.
