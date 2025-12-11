# Hard-E Agent Roles

This document describes the main agent personas in Hard-E and what each one does.

> Note: This is a conceptual design. The actual prompts and configurations are maintained in private repositories.

---

## 1. MotherAgent (Orchestrator)

**Purpose:**  
Route user requests to the right specialized agent and maintain high-level session context.

**Responsibilities:**

- Interpret the rep's intent from each utterance.
- Decide which agent(s) to call.
- Merge outputs when multiple agents contribute.
- Maintain memory of:
  - The homeowner,
  - Property details,
  - Stage in the sales cycle.

**Example behaviors:**

- If the rep says: "Help me practice the pitch for tomorrow's siding job," → call `ScriptCoachAgent`.
- If the rep says: "What should I quote for fiber cement vs vinyl here?" → call `PriceAnalystAgent`.

---

## 2. PrimerVideoAgent

**Purpose:**  
Generate scripts for primer videos that reps send to homeowners before/after appointments.

**Inputs:**

- Basic project info (property type, product, pain points).
- Any specific themes the rep wants covered (warranty, company story, process).

**Outputs:**

- Script for a 60–120 second video in the rep’s voice agent of choice (ElevenLabs).
- Optional sections:
  - Intro,
  - Problem framing,
  - Solution explanation,
  - Call to action.

---

## 3. NewClientsAgent

**Purpose:**  
Handle new lead intake and early discovery.

**Inputs:**

- Lead source (web, referral, canvassing).
- First call or meeting transcript segments.

**Outputs:**

- Structured client profile fields:
  - Name, address, contact info,
  - Project type and urgency,
  - Budget signals,
  - Objections / concerns.
- Suggested next steps (schedule measurement, send primer, etc.).

---

## 4. ClientProfileAgent (In Progress)

**Purpose:**  
Maintain and update a persistent profile for each client.

**Inputs:**

- Transcripts from multiple calls/meetings.
- CRM records (when available).

**Outputs:**

- Updated client profile object (JSON).
- Clear summary of:
  - Project scope,
  - Decision-makers,
  - Constraints (budget, HOA, timing),
  - Emotional drivers.

---

## 5. ScriptCoachAgent

**Purpose:**  
Help reps practice and refine their pitch.

**Inputs:**

- Rep’s draft script or rehearsal audio.
- Context: product, homeowner type, stage in the process.

**Outputs:**

- Feedback on clarity, pacing, and empathy.
- Alternative phrasing for key lines.
- Roleplay prompts ("I’m the homeowner; ask me XYZ").

---

## 6. PriceAnalystAgent 

**Purpose:**  
Bridge real pricing data with what reps say in the field.

**Inputs:**

- Project configuration (materials, square footage, complexity).
- Pricing feeds from suppliers (e.g., ABC Supply).
- Historical pricing trends (via Foundry or other analytics).

**Outputs:**

- Pricing per Square/Square Feet.
- Specific Product Details.
- Labor Costs based on SKU.

---

## 7. Cross-cutting Concerns

All agents share some common patterns:

- **Grounding:** Agents should defer to company policies and local pricing rules when relevant.
- **Safety & Compliance:** Avoid hallucinating numbers; for specific prices, call tools and return ranges or “I need to look this up.”
- **Traceability:** All key outputs are linked back to the transcript segments and tools used.

---

This design allows Hard-E to grow:

- You can add new agents for:
  - Hover Agent,
  - Visual Safety Agent,
  - Proposal Generation Agent.
