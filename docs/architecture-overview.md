# Hard-E Architecture Overview

Hard-E is an AI "Sales Director" for construction and remodeling companies. It automates and augments:

- Script coaching,
- Client qualification,
- Proposal structuring,
- And pricing and margin analysis.
- Emails, Text, Notes
- Automated Video Production
- Customer specific TTS audio for video

This document describes the **conceptual architecture** of the system.

---

## 1. Layers

Hard-E can be understood as four main layers:

1. **Interface Layer**
   - Voice console (e.g., Streamlit-based UI)
   - Web UI
   - Internal tools (CLI, debugging scripts)

2. **Orchestration & Agents Layer**
   - MotherAgent (orchestrator)
   - Specialized agents (PrimerVideoAgent, NewClientsAgent, etc.)
   - Agent registry and configuration

3. **Tools & Integrations Layer**
   - CRM tools (Leap)
   - Pricing tools (supplier APIs, Foundry)
   - Storage tools (S3 bucket for transcripts)
   - Utility tools (logging, metrics)

4. **Data & Analytics Layer**
   - Transcripts store (S3)
   - Metadata / session store (DB of your choice)
   - Analytics & forecasting (Foundry pipelines, dashboards)

---

## 2. Sequence: Typical Sales Coaching Session

### 2.1. High-level flow

1. **Rep starts a session**
   - Opens the Hard-E console (e.g., at `harde.app`).
   - Presses "Record" and speaks: "Hard-E, help me prep for a siding proposal at 22 Danforth Place."

2. **Audio → Text**
   - Audio is streamed to Whisper.
   - Transcription is returned in chunks.

3. **MotherAgent Receives Input**
   - The full or partial transcript is passed to the MotherAgent along with:
     - Rep identity,
     - Context (customer, property, project),
     - Conversation history.

4. **Agent Selection**
   - MotherAgent decides which agent(s) to call:
     - PrimerVideoAgent (if they need explainer scripts),
     - ScriptCoachAgent (if they’re rehearsing),
     - ClientProfileAgent (if this is a new lead),
     - PriceAnalystAgent (if pricing questions appear).

5. **Tool Calls**
   - Agents may call tools (see `examples/pseudo_tool_functions.py`):
     - `lookup_client(...)`
     - `fetch_pricing(...)`
     - `store_transcript(...)`
     - `generate_primer_video_script(...)`

6. **Response & Coaching**
   - The chosen agent returns:
     - Coaching instructions,
     - Suggested phrasing,
     - Or a script.
   - The response is synthesized via TTS and played back to the rep.

7. **Persistence**
   - The transcript and key entities (budget, timeline, product choices) are logged to S3 + DB.
   - These records can be used later for:
     - Follow-up,
     - Training,
     - Analytics.

---

## 3. Deployment Model

### 3.1. Current / Prototype

- **Compute:** AWS EC2 instance (or small cluster)
- **Storage:** S3 buckets for transcripts and artifacts
- **LLMs & STT/TTS:** OpenAI APIs (Agents SDK + Whisper + TTS)
- **Config:** Environment variables (.env) and YAML config files
- **Monitoring:** CloudWatch / logs (planned more advanced monitoring)

### 3.2. Future / Target

- Break out parts of the system into:
  - **Agents & orchestration** (containerized or as serverless functions),
  - **Voice service** (standalone microservice),
  - **Pricing analytics service** (integrated with Foundry),
  - **Web UI** (Next.js or similar).

- Introduce:
  - Dedicated metrics and tracing,
  - A more formal schema for conversation and session data.

---

## 4. Security Considerations

- All secrets (API keys, tokens) are stored outside the codebase (env variables, secret managers).
- Transcripts may contain sensitive homeowner information; encryption at rest and strict access control are required.
- CRM and supplier APIs should be accessed through well-scoped tokens with minimal privileges.

---

## 5. Why This Architecture Matters

Hard-E is not just a "chatbot." It is:

- A **multi-agent system** that mirrors a real organization (Sales Director + team),
- A **voice-native interface** that fits how reps actually work,
- A **data pipeline** that captures what happens in the field and turns it into institutional knowledge.

This architecture is intended to be:

- **Extensible** – new agents and tools can be added,
- **Auditable** – transcripts and decisions can be reviewed,
- **Practical** – built with widely-used cloud and LLM tools.
