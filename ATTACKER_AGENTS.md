# ATTACKER_AGENTS.md — Red-Team Scammer Caller (ElevenLabs)

## 0) Owner + Goal

**Owner:** Attacker engineer  
**Goal:** Build the "scammer caller" service that outputs the next caller line (text) and serves **pre-generated ElevenLabs TTS audio** so the UI can play a realistic phone call.

**Key Design Decisions:**
- MVP is fully scripted (no LLM required)
- Audio is pre-generated at startup for all scenario lines (eliminates runtime latency)
- Conforms to SHARED_CONTRACTS.md for types, errors, and conventions

**Before you start:** Read SHARED_CONTRACTS.md for the canonical Tactic enum, error format, and scenario schema.

---

## 1) MVP Deliverables (Target: < 1 hour)

### Must Have
1. Health and version endpoints (`GET /health`, `GET /version`)
2. Scenario listing endpoint (`GET /api/v1/scenarios`)
3. Next caller turn endpoint (`POST /api/v1/sessions/{session_id}/next_caller`)
4. Audio serving endpoint (`GET /api/v1/audio/{audio_id}`)
5. API key validation on all `/api/v1/*` endpoints
6. At least 1 scenario JSON file with 6-10 caller lines
7. Pre-generated audio cache populated at startup

### Nice to Have (Only If MVP Complete)
- Multiple voice options
- LLM-driven adaptive responses
- Difficulty slider

---

## 2) Pre-Generated Audio Strategy

**Why pre-generate?** ElevenLabs TTS takes 1-3 seconds per request. For a demo simulating a phone call, this latency destroys the illusion. Since MVP scenarios are fully scripted, we know all lines ahead of time.

### Startup Behavior

On service startup:
1. Load all scenario JSON files from `/scenarios/`
2. Validate each against the schema in SHARED_CONTRACTS.md (fail fast if malformed)
3. For each line in each scenario:
   - Check if audio already exists in cache (keyed by `{scenario_id}_{line_index}`)
   - If missing: call ElevenLabs TTS and store the result
4. Log startup complete with count of scenarios and audio files

### Audio Cache Structure

```python
# In-memory cache (MVP)
audio_cache: dict[str, bytes] = {}  # key: "{scenario_id}_{line_index}", value: mp3 bytes

# Example key: "ceo_impersonation_001_0" for first line of CEO scenario
```

### Environment Variables

```bash
ELEVENLABS_API_KEY=sk-...          # Required
ELEVENLABS_VOICE_ID=21m00Tcm...    # Default voice ID
ELEVENLABS_MODEL_ID=eleven_flash_v2_5  # Use Flash for lower latency during pre-gen
AUDIO_FORMAT=mp3_44100_128         # Output format
BASE_URL=http://localhost:8001     # For constructing audio URLs
API_KEY=your-shared-secret         # For X-API-Key validation
```

### ElevenLabs API Call (for pre-generation)

```python
import requests

def generate_audio(text: str, voice_id: str) -> bytes:
    """
    Generate TTS audio using ElevenLabs API.
    Called during startup for each scenario line.
    
    Args:
        text: The caller line text to synthesize
        voice_id: ElevenLabs voice ID to use
        
    Returns:
        Raw MP3 audio bytes
        
    Raises:
        RuntimeError: If ElevenLabs API call fails
    """
    response = requests.post(
        f"https://api.elevenlabs.io/v1/text-to-speech/{voice_id}",
        headers={
            "xi-api-key": os.environ["ELEVENLABS_API_KEY"],
            "Content-Type": "application/json",
        },
        json={
            "text": text,
            "model_id": os.environ.get("ELEVENLABS_MODEL_ID", "eleven_flash_v2_5"),
            "output_format": os.environ.get("AUDIO_FORMAT", "mp3_44100_128"),
        },
    )
    if response.status_code != 200:
        raise RuntimeError(f"ElevenLabs API error: {response.status_code} {response.text}")
    return response.content
```

---

## 3) Internal State

```python
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class AttackerSessionState:
    """
    Tracks the current position in a scenario for a given session.
    
    Attributes:
        session_id: Unique session identifier (created by Defender, passed to us)
        scenario_id: Which scenario is being played
        line_index: Current position in the scenario (0-indexed)
        completed: Whether we've exhausted all lines
    """
    session_id: str
    scenario_id: str
    line_index: int = 0
    completed: bool = False

# In-memory session state (MVP)
# Key: session_id, Value: AttackerSessionState
sessions: dict[str, AttackerSessionState] = {}

# Pre-generated audio cache
# Key: "{scenario_id}_{line_index}", Value: MP3 bytes
audio_cache: dict[str, bytes] = {}

# Loaded scenarios
# Key: scenario_id, Value: Scenario dict
scenarios: dict[str, dict] = {}
```

**Note on turn_index:** The Attacker service tracks `line_index` internally to know which line to return next. However, the **canonical turn_index** is owned by the Defender service. The Attacker returns `line_index` for informational purposes, but the Defender assigns the official turn number when it receives the event.

---

## 4) API Contract

Base URL: `ATTACKER_BASE_URL` (e.g., `http://localhost:8001`)

All `/api/v1/*` endpoints require the `X-API-Key` header matching the `API_KEY` environment variable.

---

### 4.1 Health Check

**GET /health**

No authentication required.

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

Response 503 (if startup failed):
```json
{
  "status": "degraded",
  "service": "attacker",
  "issues": ["elevenlabs_unreachable", "no_scenarios_loaded"],
  "timestamp": "2024-01-15T10:30:00Z"
}
```

---

### 4.2 Version

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

### 4.3 List Scenarios

**GET /api/v1/scenarios**

Returns all available scenarios. UI uses this to populate the scenario dropdown.

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
    },
    {
      "scenario_id": "tech_support_002",
      "title": "Fake Tech Support",
      "description": "Caller poses as IT helpdesk requesting remote access.",
      "difficulty": "easy",
      "line_count": 8,
      "expected_duration_seconds": 240
    }
  ]
}
```

---

### 4.4 Next Caller Turn

**POST /api/v1/sessions/{session_id}/next_caller**

Returns the next caller line and audio URL. If this is the first call for this session_id, creates internal tracking state.

Request:
```json
{
  "scenario_id": "ceo_impersonation_001"
}
```

Response 200 (success):
```json
{
  "session_id": "sess_abc123",
  "scenario_id": "ceo_impersonation_001",
  "line_index": 0,
  "caller_text": "Hi, this is the CEO. I need you to reset my MFA right now.",
  "tactics": ["authority_impersonation", "urgency_pressure"],
  "audio_url": "http://localhost:8001/api/v1/audio/ceo_impersonation_001_0",
  "lines_remaining": 5
}
```

Response 409 (scenario exhausted):
```json
{
  "error": {
    "code": "SCENARIO_DONE",
    "message": "No more caller lines available in this scenario"
  }
}
```

Response 404 (scenario not found):
```json
{
  "error": {
    "code": "SCENARIO_NOT_FOUND",
    "message": "Scenario 'unknown_scenario' does not exist"
  }
}
```

**Behavior:**
1. If `session_id` not seen before: create new `AttackerSessionState` with `line_index=0`
2. If `session_id` exists but `scenario_id` differs: return 400 (cannot change scenario mid-session)
3. If `line_index >= len(lines)`: return 409 `SCENARIO_DONE`
4. Otherwise: return current line, increment `line_index`

---

### 4.5 Serve Audio

**GET /api/v1/audio/{audio_id}**

Returns pre-generated audio bytes. The `audio_id` is `{scenario_id}_{line_index}`.

Response 200:
- Content-Type: `audio/mpeg`
- Body: Raw MP3 bytes

Response 404:
```json
{
  "error": {
    "code": "AUDIO_NOT_FOUND",
    "message": "Audio file 'unknown_id' not found in cache"
  }
}
```

**Note:** Audio is served directly from the in-memory cache. There is no TTL because audio is pre-generated at startup and persists for the lifetime of the service.

---

## 5) Scenario File Format

Store scenarios in `/scenarios/*.json`. Validated at startup.

Example (`/scenarios/ceo_impersonation_001.json`):
```json
{
  "scenario_id": "ceo_impersonation_001",
  "title": "CEO Impersonation",
  "description": "A caller claims to be the CEO and demands an immediate MFA reset without proper verification.",
  "difficulty": "medium",
  "expected_duration_seconds": 180,
  "lines": [
    {
      "text": "Hi, this is the CEO. I need you to reset my MFA right now.",
      "tactics": ["authority_impersonation", "urgency_pressure"]
    },
    {
      "text": "I'm in a board meeting and I can't access my account. This is extremely time-sensitive.",
      "tactics": ["urgency_pressure", "authority_impersonation"]
    },
    {
      "text": "I don't have time for verification. If you can't help me, I'll have your manager handle it.",
      "tactics": ["threat_intimidation", "identity_bypass"]
    },
    {
      "text": "Just tell me what you see on my account. I need to know if someone else has been accessing it.",
      "tactics": ["information_probing", "urgency_pressure"]
    },
    {
      "text": "Look, I can verify later. Just reset it now, it's critical for the board meeting.",
      "tactics": ["identity_bypass", "urgency_pressure"]
    },
    {
      "text": "Fine. What's the callback number? Actually, I can't take calls right now. Just email me the reset link.",
      "tactics": ["callback_evasion"]
    }
  ]
}
```

---

## 6) Implementation Skeleton

```python
"""
SE Shield Attacker Service
Provides scripted social engineering scenarios with pre-generated TTS audio.
"""

import os
import json
import uuid
from pathlib import Path
from dataclasses import dataclass
from datetime import datetime, timezone
from typing import Optional

from fastapi import FastAPI, HTTPException, Header, Depends
from fastapi.responses import Response
from pydantic import BaseModel

app = FastAPI(title="SE Shield Attacker Service", version="0.1.0")

# ============================================================================
# Configuration
# ============================================================================

API_KEY = os.environ.get("API_KEY", "dev-api-key")
BASE_URL = os.environ.get("BASE_URL", "http://localhost:8001")
SCENARIOS_DIR = Path(os.environ.get("SCENARIOS_DIR", "./scenarios"))

# ============================================================================
# State (in-memory for MVP)
# ============================================================================

@dataclass
class AttackerSessionState:
    """Tracks position in scenario for a session."""
    session_id: str
    scenario_id: str
    line_index: int = 0

sessions: dict[str, AttackerSessionState] = {}
scenarios: dict[str, dict] = {}
audio_cache: dict[str, bytes] = {}

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
# Startup: Load Scenarios and Pre-Generate Audio
# ============================================================================

@app.on_event("startup")
async def startup_load_scenarios():
    """
    Load all scenario files and pre-generate TTS audio.
    This runs once when the service starts.
    """
    if not SCENARIOS_DIR.exists():
        print(f"WARNING: Scenarios directory {SCENARIOS_DIR} does not exist")
        return
    
    for scenario_file in SCENARIOS_DIR.glob("*.json"):
        try:
            with open(scenario_file) as f:
                scenario = json.load(f)
            
            # Validate required fields
            assert "scenario_id" in scenario, "Missing scenario_id"
            assert "lines" in scenario, "Missing lines"
            
            scenarios[scenario["scenario_id"]] = scenario
            print(f"Loaded scenario: {scenario['scenario_id']} ({len(scenario['lines'])} lines)")
            
            # Pre-generate audio for each line
            for idx, line in enumerate(scenario["lines"]):
                cache_key = f"{scenario['scenario_id']}_{idx}"
                if cache_key not in audio_cache:
                    try:
                        audio_bytes = generate_elevenlabs_audio(line["text"])
                        audio_cache[cache_key] = audio_bytes
                        print(f"  Generated audio: {cache_key}")
                    except Exception as e:
                        print(f"  WARNING: Failed to generate audio for {cache_key}: {e}")
                        
        except Exception as e:
            print(f"ERROR loading {scenario_file}: {e}")
    
    print(f"Startup complete: {len(scenarios)} scenarios, {len(audio_cache)} audio files")

def generate_elevenlabs_audio(text: str) -> bytes:
    """
    Call ElevenLabs API to generate TTS audio.
    Returns raw MP3 bytes.
    """
    import requests
    
    voice_id = os.environ.get("ELEVENLABS_VOICE_ID", "21m00Tcm4TlvDq8ikWAM")
    api_key = os.environ.get("ELEVENLABS_API_KEY")
    
    if not api_key:
        raise RuntimeError("ELEVENLABS_API_KEY not configured")
    
    response = requests.post(
        f"https://api.elevenlabs.io/v1/text-to-speech/{voice_id}",
        headers={
            "xi-api-key": api_key,
            "Content-Type": "application/json",
        },
        json={
            "text": text,
            "model_id": os.environ.get("ELEVENLABS_MODEL_ID", "eleven_flash_v2_5"),
        },
    )
    
    if response.status_code != 200:
        raise RuntimeError(f"ElevenLabs error: {response.status_code}")
    
    return response.content

# ============================================================================
# API Endpoints
# ============================================================================

@app.get("/health")
async def health_check():
    """Health check endpoint (no auth required)."""
    return {
        "status": "ok" if scenarios else "degraded",
        "service": "attacker",
        "scenarios_loaded": len(scenarios),
        "audio_files_cached": len(audio_cache),
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

@app.get("/api/v1/scenarios", dependencies=[Depends(verify_api_key)])
async def list_scenarios():
    """List all available scenarios."""
    return {
        "scenarios": [
            {
                "scenario_id": s["scenario_id"],
                "title": s.get("title", s["scenario_id"]),
                "description": s.get("description", ""),
                "difficulty": s.get("difficulty", "medium"),
                "line_count": len(s["lines"]),
                "expected_duration_seconds": s.get("expected_duration_seconds", 180),
            }
            for s in scenarios.values()
        ]
    }

class NextCallerRequest(BaseModel):
    """Request body for next_caller endpoint."""
    scenario_id: str

@app.post("/api/v1/sessions/{session_id}/next_caller", dependencies=[Depends(verify_api_key)])
async def next_caller(session_id: str, request: NextCallerRequest):
    """
    Get the next caller line for a session.
    Creates session state on first call; returns 409 when scenario exhausted.
    """
    # Validate scenario exists
    if request.scenario_id not in scenarios:
        raise HTTPException(status_code=404, detail={
            "error": {
                "code": "SCENARIO_NOT_FOUND",
                "message": f"Scenario '{request.scenario_id}' does not exist"
            }
        })
    
    scenario = scenarios[request.scenario_id]
    
    # Get or create session state
    if session_id not in sessions:
        sessions[session_id] = AttackerSessionState(
            session_id=session_id,
            scenario_id=request.scenario_id,
        )
    
    state = sessions[session_id]
    
    # Validate scenario hasn't changed
    if state.scenario_id != request.scenario_id:
        raise HTTPException(status_code=400, detail={
            "error": {
                "code": "SCENARIO_MISMATCH",
                "message": "Cannot change scenario mid-session"
            }
        })
    
    # Check if scenario is exhausted
    if state.line_index >= len(scenario["lines"]):
        raise HTTPException(status_code=409, detail={
            "error": {
                "code": "SCENARIO_DONE",
                "message": "No more caller lines available in this scenario"
            }
        })
    
    # Get current line
    line = scenario["lines"][state.line_index]
    audio_key = f"{request.scenario_id}_{state.line_index}"
    
    response = {
        "session_id": session_id,
        "scenario_id": request.scenario_id,
        "line_index": state.line_index,
        "caller_text": line["text"],
        "tactics": line.get("tactics", []),
        "audio_url": f"{BASE_URL}/api/v1/audio/{audio_key}" if audio_key in audio_cache else None,
        "lines_remaining": len(scenario["lines"]) - state.line_index - 1,
    }
    
    # Advance to next line
    state.line_index += 1
    
    return response

@app.get("/api/v1/audio/{audio_id}", dependencies=[Depends(verify_api_key)])
async def serve_audio(audio_id: str):
    """Serve pre-generated audio file."""
    if audio_id not in audio_cache:
        raise HTTPException(status_code=404, detail={
            "error": {
                "code": "AUDIO_NOT_FOUND",
                "message": f"Audio file '{audio_id}' not found in cache"
            }
        })
    
    return Response(
        content=audio_cache[audio_id],
        media_type="audio/mpeg",
    )
```

---

## 7) Generating Scenarios with Tonic Fabricate

Instead of manually authoring scenario JSON files, you can use [Tonic Fabricate](https://www.tonic.ai/products/fabricate) to generate diverse, realistic scenarios via natural language prompts.

### 7.1 Why Use Fabricate?

- **Speed:** Generate 10+ scenarios in minutes instead of hours
- **Diversity:** AI creates varied attack patterns you might not think of
- **Edge Cases:** Request specific difficult scenarios for advanced training
- **Consistency:** Output matches your schema automatically

### 7.2 Example Prompts

**Basic Scenario Generation:**
```
Generate a JSON file for a social engineering training scenario where a caller 
impersonates the CEO of a company and pressures a helpdesk agent to reset MFA 
without proper verification. Include 6 caller lines with escalating tactics.
Use this schema: {scenario_id, title, description, difficulty, expected_duration_seconds, lines: [{text, tactics}]}
Tactics must be from: authority_impersonation, urgency_pressure, credential_harvesting, 
identity_bypass, threat_intimidation, emotional_manipulation, information_probing, callback_evasion
```

**Batch Generation:**
```
Generate 5 different social engineering scenarios for a bank customer support training program:
1. A caller claiming to be from IT support needing remote access
2. A "customer" who lost their card and is extremely emotional
3. Someone claiming to be a vendor demanding payment information
4. A fake police officer requesting account information for an "investigation"
5. An "executive assistant" requesting an urgent wire transfer

For each, generate 6-8 caller lines with appropriate tactics from the provided enum.
Output as a JSON array matching the schema.
```

**Edge Case Generation:**
```
Generate a difficult social engineering scenario where the caller uses subtle 
manipulation without obvious red flags. The caller should:
- Build rapport slowly before making requests
- Use insider terminology that sounds legitimate
- Apply gentle pressure rather than aggressive urgency
- Make reasonable-sounding justifications for each request

This should be marked as "hard" difficulty.
```

### 7.3 Fabricate Output → Scenario File

Fabricate will output JSON that you may need to lightly edit:

1. Copy Fabricate output to `/scenarios/{scenario_id}.json`
2. Verify `scenario_id` is unique and follows naming convention
3. Validate tactics are from the canonical enum (SHARED_CONTRACTS.md)
4. Test by running the attacker service and hitting `/api/v1/scenarios`

### 7.4 Mock API for UI Development

Fabricate can also generate mock API endpoints. If the Attacker service isn't ready, the UI engineer can:

1. Use Fabricate to create a mock `/api/v1/sessions/{id}/next_caller` endpoint
2. Configure it to return realistic response shapes
3. Develop the UI against the mock while backend is built
4. Switch to real endpoint when ready

This enables parallel development across all three workstreams.

---

## 8) Stretch Goals (Only If MVP Complete)

1. **Multiple voices:** Allow scenario to specify different `voice_id` per line (e.g., different accents)
2. **LLM-driven adaptive mode:** Replace scripted lines with Claude/GPT generating responses based on trainee's replies
3. **Difficulty slider:** Modify line selection or add/remove tactics based on difficulty setting
4. **Streaming audio:** Use ElevenLabs streaming endpoint for LLM-driven mode where pre-generation isn't possible

---

END OF ATTACKER SPEC.
