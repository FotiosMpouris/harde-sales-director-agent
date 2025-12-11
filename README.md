# Hard-E – AI Sales Director Agent (Architecture Showcase)

Hard-E is an AI "Sales Director" clone for construction and remodeling companies.

It is designed to:
- Capture and extend the expertise of a real Sales Director,
- Coach sales reps in real time,
- Analyze proposal transcripts and customer interactions,
- And (in future phases) integrate live material pricing and forecasting.

> This repository is an **architecture and pattern showcase**.  
> The production implementation runs on private infrastructure and includes proprietary integrations, secrets, and customer data which are **not** included here.

---

## High-Level Overview

**Hard-E** combines:

- **Multi-agent orchestration** using the OpenAI Agents SDK  
- **Voice interface** (Whisper for speech-to-text + TTS for responses)  
- **Transcript analysis** stored in S3 for ongoing training and coaching  
- **Sales workflow integration**, including CRM and supplier pricing APIs  
- A planned **PriceAnalystAgent** that uses real pricing data (e.g., ABC Supply) and Palantir Foundry pipelines.

The philosophy behind Hard-E:

> Take the *real* behavior of a Sales Director  
> → encode the workflows, prompts, and data flows  
> → provide every rep with an always-available expert coach.

---

## Core Components

Hard-E is composed of several logical components:

1. **MotherAgent (Orchestrator)**  
   - Receives user input (voice or text).  
   - Decides which specialized agent should handle the task.  
   - Maintains conversation-wide context and state.

2. **Specialized Agents** (see `docs/agent-roles.md`)  
   - `PrimerVideoAgent` – generates explainer/primer scripts for homeowners.  
   - `NewClientsAgent` – handles intake, qualification, and initial call flow.  
   - `ClientProfileAgent` – builds structured client profiles from transcripts & CRM data.  
   - `ScriptCoachAgent` – helps reps rehearse or refine pitch scripts.  
   - `PriceAnalystAgent` (planned) – looks at material pricing trends and margin impact.

3. **Voice I/O Pipeline**  
   - Whisper STT → text → MotherAgent + specialized agents → text response → TTS → audio stream to the user.  
   - Designed with a two-phase streaming plan:
     - **Fire & Stitch** (chunked TTS with fast first audio),
     - **Live Stream** (full duplex, websocket-based streaming).

4. **Transcript & Data Layer**  
   - Audio and transcripts stored in S3.  
   - Metadata (client ID, property, project scope) persisted for reuse.  
   - Planned integration with Palantir Foundry for analytics and pricing.

5. **Integrations**  
   - **Leap CRM** (planned/experimental): sync client records and pipeline data.  
   - **Supplier APIs / ABC Supply**: real-time and historical material pricing.  
   - **HOVER / proposal tools**: structured data from project scans and line items.

---

## Technical Stack (Conceptual)

**Languages & Frameworks**
- Python (core services, agents, tooling)
- Streamlit (prototype voice console)
- JavaScript/TypeScript (web front-ends, dashboards)

**AI / LLM**
- OpenAI Agents SDK (multi-agent orchestration)
- Whisper (speech-to-text)
- TTS (text-to-speech, multiple providers configurable via env)

**Cloud & Data**
- AWS EC2 (main app hosting)
- AWS S3 (transcripts, audio, artifacts)
- AWS Lambda (optional/offloaded tasks)
- Palantir Foundry (planned, for pricing and analytics)

**CRM & External APIs**
- Leap CRM API (pipeline & client data)
- Supplier pricing APIs (e.g., ABC Supply)
- HOVER or similar property-report providers

---

## Repository Contents

This repo is intentionally **safe and non-sensitive**. It contains:

- `README.md` – this overview.
- `.env.example` – documented environment variables (no real secrets).
- `docs/architecture-overview.md` – deeper architecture narrative and diagrams (text-based).
- `docs/agent-roles.md` – detailed responsibilities and prompts for each agent.
- `docs/voice-pipeline.md` – explanation of the streaming audio pipeline.
- `examples/sample_agent_config.yaml` – redacted sample config for agent wiring.
- `examples/transcript_snippet.txt` – synthetic transcript data used in examples.
- `examples/pseudo_tool_functions.py` – pseudo-code showing how tools might be wired.

None of these files contain:

- Real API keys,  
- Real customer data, or  
- Proprietary secrets from partner companies.

---

## Quick Mental Model

You can think of Hard-E as:

```text
[Sales Rep Voice]
       ↓ (Whisper)
[Transcribed Utterance]
       ↓
[MotherAgent]
       ↓            ↘
[Specialized Agent]  [State / Memory]
       ↓
[Proposed Response + Actions]
       ↓
[TTS → Audio Output]   +   [CRM / Pricing / Notes Update]
