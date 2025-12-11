# Hard-E – AI Sales Director for Residential Contracting

**Hard-E** is a production-deployed AI Sales Director application built for **Patriot Contracting Inc.**, designed to function as an **autonomous AI employee** rather than a traditional assistant. It handles complex sales workflows through natural conversation, generates customer-facing content, and executes multi-step autonomous pipelines—all without requiring users to learn software interfaces.

> **This repository is an architecture showcase.**  
> The production system is available only for scheduled demos. Please visit https://harde.app and book an appointment. This system itself runs on private AWS infrastructure with proprietary integrations, API credentials, and customer data that are not included here.

---

## What Hard-E Does

Hard-E transforms how residential contracting sales teams operate by providing:

- **Conversational Workflow Execution** – Generate sales scripts, create CRM records, search pricing data, and analyze job transcripts through natural language dialogue
- **Autonomous Task Completion** – Email-triggered primer video generation pipeline (script → audio → video → delivery) runs completely unattended
- **Multi-Agent Intelligence** – Specialized agents handle routing, CRM operations, knowledge synthesis, script generation, customer creation, pricing analysis, and web research
- **Voice-First Interface** – Audio playback of generated scripts with two-stage enhancement pipeline (OpenAI TTS → ElevenLabs Speech-to-Speech)
- **Real-Time CRM Integration** – Direct interaction with Leap CRM via undocumented API endpoints discovered through reverse engineering
- **Knowledge Synthesis** – Distills 90+ sales director call transcripts into actionable coaching and script templates

**Philosophy:**  
Hard-E doesn't assist with sales—it **performs** sales operations as an autonomous employee would, freeing human sales professionals to focus on relationship-building and deal-closing.

---

## Current Capabilities (Production, December 2024)

### Core Operations

#### **Script Generation** (Stateful Agent)
- Primer scripts for homeowner education
- Standard sales scripts for qualified leads
- Context-aware generation using CRM data and knowledge bases
- Audio generation with two-stage enhancement (standard: OpenAI TTS, primer: ElevenLabs)

#### **CRM Operations** (Leap Integration)
- Customer search and data retrieval
- Job record creation and updates
- Note attachment to jobs and customers
- Pipeline management support
- @mention support for targeted operations

#### **Customer Creation** (Stateful Agent, 95%+ Accuracy)
- Natural language intake (conversational or bulk text)
- LLM-powered field extraction (GPT-4o-mini)
- Complete customer records with all required fields
- Direct API submission to Leap CRM

#### **Knowledge Synthesis**
- Query 7 specialized S3 knowledge bases
- Streaming responses for long-form content
- Distilled from 90+ Sales Director transcripts
- Pattern recognition and best practice extraction

#### **Pricing Intelligence** (Developing)
- Module-based catalog search (roofing, siding, gutters, windows)
- Area calculation tools for material estimation
- Integration with external pricing APIs
- Margin analysis capabilities

#### **Call Analysis**
- Transcript pattern analysis across 90 distilled conversations
- Objection handling insights
- Sales technique identification
- Performance coaching recommendations

#### **Web Research**
- Perplexity API integration (sonar-pro model)
- Real-time information gathering
- Competitor analysis support

### Autonomous Workflows

#### **Primer Video Pipeline**: Email-triggered, fully autonomous 10-step process
1. Email monitoring (60-second polling)
2. Script generation (context-aware)
3. Audio creation (two-stage enhancement)
4. Video assembly with music overlay (Cloudinary)
5. Google Drive delivery with shareable links
6. CRM notification and client email
7. Automatic cleanup of temporary assets

**Total time:** 18-22 minutes, zero human intervention

---

## Architecture

### Multi-Agent System

Hard-E uses a hybrid architecture combining manual streaming control (voice playback) with the OpenAI Agents SDK for complex workflows:

**Agent Hierarchy:**
```
triage_agent (Router)
├── leap_interaction_agent (CRM Operations)
├── knowledge_agent (S3 Synthesis)
├── scripting_agent (Stateful Script Generation)
├── new_clients_agent (Stateful Customer Creation)
├── analysis_agent (Transcript Analysis)
├── web_search_agent (Perplexity Integration)
└── pricing_agent (Module-Based Pricing)
```

**Key Agents:**

- **Triage Agent** – Entry point, routing only, no tools, handoffs to specialized agents
- **Leap Interaction Agent** – 8 CRM tools, @mention support for targeted operations
- **Knowledge Agent** – S3 knowledge base queries with streaming support
- **Scripting Agent** – Stateful (ScriptDraftState), dual script types (primer/standard), audio generation
- **New Clients Agent** – Stateful (NewClientState), LLM extraction, 95%+ accuracy on customer creation
- **Analysis Agent** – Pattern recognition across 90 distilled transcripts
- **Web Search Agent** – Perplexity sonar-pro integration for real-time research
- **Pricing Agent** – Module-based catalog search, area calculations, developing capability

### State Management

**Current (Production):**
- `st.session_state` for Streamlit-based conversational memory
- Stateful agents maintain workflow context across multiple turns
- No persistent storage between sessions

**Planned (v3.0):**
- Redis for distributed state management
- Multi-user support with isolated sessions
- Persistent conversation history

### Data Architecture

**S3 Knowledge Bases:**
```
s3://hard-e-transcripts/
├── CORE_KNOWLEDGE/              # Foundational sales methodology
├── SALES_DIRECTOR_CALLS/        # 90 distilled transcripts
├── OBJECTION_HANDLING/          # Response patterns and techniques
├── PRICING_STRATEGIES/          # Margin and pricing guidance
├── PRODUCT_KNOWLEDGE/           # Technical specifications
├── SCRIPT_TEMPLATES/            # Proven script frameworks
└── CUSTOMER_PSYCHOLOGY/         # Behavioral insights
```

**Data Flow:**
- **Live transactional data:** Leap CRM API (customers, jobs, pipeline)
- **Historical knowledge:** S3 buckets (synthesized, searchable)
- **External enrichment:** Perplexity, pricing APIs, HOVER integration (planned)
- **Generated content:** Temporary local storage with automatic cleanup

### Voice Pipeline

**Standard Audio (Script Playback):**
```
Text → OpenAI TTS (tts-1-hd, alloy) → MP3 → Streamlit Audio Widget
```

**Enhanced Audio (Primer Videos):**
```
Text → OpenAI TTS → MP3 
     → ElevenLabs Speech-to-Speech (voice enhancement) 
     → Enhanced MP3 → Video Assembly
```


**v3.0 Vision:**
- Real-time voice streaming (sub-250ms latency)
- WebSocket-based bidirectional audio
- Retell/Vapi integration for voice interface
- Concurrent multi-user support

---

## Technical Stack

### Core Framework
- Python 3.11
- Streamlit (current UI)
- OpenAI Agents SDK >=0.0.17

### AI Models
- **GPT-4o** (all agents except extraction)
- **GPT-4o-mini** (NewClientsAgent field parsing for cost efficiency)
- **Whisper** (future voice input)
- **OpenAI TTS** (tts-1-hd, alloy voice)
- **ElevenLabs Speech-to-Speech** (primer audio enhancement)
- **Grok-3-mini-beta** (optional refinement, configurable)
- **Perplexity sonar-pro** (web research)

### Infrastructure
- **AWS EC2** t3.small (2 vCPU, 2 GiB RAM, 20 GiB EBS)
- Amazon Linux 2
- Nginx reverse proxy with SSL (Let's Encrypt)
- systemd service management
- AWS S3 for knowledge storage and artifacts

### Integrations
- Leap CRM (V3 API + undocumented endpoints)
- Perplexity API (web search)
- ElevenLabs API (voice enhancement)
- OpenAI API (agents, TTS, models)
- Cloudinary (video assembly)
- Google Drive API (video delivery)
- Gmail API (email monitoring for autonomous workflows)

### Deployment
- **Production:** https://streaming.harde.app (port 8504)
- **Service:** harde-streaming.service (systemd)
- **Process:** Managed by systemd with automatic restart
- **Logs:** /var/log/harde-streaming.log and systemd journal

---

## Project Evolution

### Development Timeline

#### Early Stages (2024 Q1-Q2)
- Initial prototype with basic conversational interface
- Manual CRM operations and script generation
- Learning OpenAI Agents SDK patterns

#### Production Deployment (2024 Q3)
- AWS infrastructure setup with Nginx and SSL
- Leap CRM integration (reverse-engineered API)
- S3 knowledge base architecture
- Streaming version deployed to production

#### Current State (2024 Q4)
- Dual environments operational (development + production)
- Autonomous workflows (primer video pipeline)
- Stateful agents with high-accuracy customer creation
- Multi-agent coordination with specialized roles
- 8+ operational workflows across 30+ tools

#### Active Development
- Pricing system refinement
- Additional autonomous workflow patterns
- Knowledge base expansion
- Tool optimization and error handling

### Next Generation (v3.0 - Planning Phase)

**Target Architecture:**
- **Frontend:** React (replacing Streamlit)
- **Backend:** FastAPI (replacing Streamlit server)
- **State:** Redis (replacing session state)
- **Voice:** Real-time streaming with Retell/Vapi webhooks
- **Deployment:** New EC2 instance at https://harde.app

**Key Improvements:**
- Sub-250ms voice latency
- Multi-user concurrent sessions
- Modern responsive UI
- Horizontal scaling capability
- Enhanced real-time features
- Better mobile experience

**Migration Strategy:**
- Parallel operation (streaming.harde.app continues in production)
- Gradual feature porting and testing
- Zero downtime transition
- Comprehensive regression testing before cutover

---

## Why This Matters

### The Problem

Residential contracting sales teams spend 3+ hours per day on administrative tasks:

- Manually writing scripts for every customer type and project scope
- Searching CRM for customer and job history
- Copying and pasting data between systems
- Creating customer records with 12+ required fields
- Researching pricing and product specifications
- Generating proposals and customer education materials

**Result:** Sales professionals spend most of their time on data entry instead of selling.

### The Solution

Hard-E operates as an AI employee that handles operational workflows through conversation:

- *"Generate a primer script for John Smith's roof replacement job"*
- *"Create a new customer: Sarah Johnson, 555-0123, interested in siding..."*
- *"What's our standard objection response for price concerns?"*
- *"Search for gutter pricing in the Englert catalog"*

**No software to learn. No interfaces to navigate. Just conversation.**

### The Breakthrough: Autonomous Operation

The primer video pipeline represents a fundamental shift from AI assistance to AI autonomy:

1. Sales Director sends email: *"Please produce Primer Video for [customer]"*
2. Hard-E daemon detects email (60-second polling)
3. System executes 10-step workflow without human intervention
4. Notifications sent to CRM and customer
5. All temporary files cleaned up automatically

**This is not automation. This is autonomous execution.**

### Market Opportunity

- 250,000+ residential contracting companies in the US
- Average 5-10 sales reps per company
- $50-$500M annual revenue in target segment
- Template for broader AI employee deployment across industries

Hard-E proves that AI can perform complex multi-step business workflows with zero human intervention—not just provide suggestions or draft content, but actually execute operations end-to-end.

---


## Key Design Principles

### 1. Conversational Interface as Primary UX
No forms, no buttons, no navigation—just natural language. Users describe what they need, and Hard-E executes.

### 2. Agents as Specialists, Not Generalists
Each agent has a narrow, well-defined responsibility with appropriate tools. Routing logic keeps workflows clean.

### 3. State When It Matters
Stateful agents (scripting, customer creation) maintain context across multiple conversation turns for complex workflows.

### 4. Autonomous by Default
Workflows should execute without human intervention once triggered. Manual steps are optimization opportunities.

### 5. Fail Gracefully, Log Everything
Every autonomous workflow includes comprehensive error handling and logging. Failures should never be silent.

### 6. Real Data, Real Operations
Hard-E performs actual business operations (CRM writes, file generation, API calls), not simulations or suggestions.

---

### Planned Improvements

**Short Term (v2.x - Current Streamlit):**
- Enhanced pricing catalog integration
- Additional autonomous workflow patterns
- Improved error handling and recovery
- Knowledge base expansion and refinement
- Tool performance optimization

**Long Term (v3.0 - React/FastAPI):**
- Real-time voice streaming with sub-250ms latency
- Multi-user concurrent sessions
- Redis-based distributed state
- Modern responsive UI
- Horizontal scaling architecture
- Mobile-optimized experience
- Comprehensive testing framework
- API for external integrations

---

**Note:** This repository does not include runnable code. The production system requires:

- Private AWS infrastructure access
- Leap CRM API credentials
- OpenAI API keys with Agents SDK access
- S3 buckets with proprietary knowledge bases
- ElevenLabs, Perplexity, and other API credentials
- Cloudinary and Google Drive API access

---

## About the Creator

Hard-E was built by an independent developer with extensive experience in:

- Software development and AI integration
- Political campaigns and community organizing
- Residential construction and sales operations
- Building AI agents as force multipliers for individual developers

**Philosophy:**  
AI should amplify individual capability, not require enterprise resources. One developer with the right architecture can build systems that previously required entire teams.

**Vision:**  
Hard-E is a template for AI employees across industries. The patterns proven here—conversational interfaces, autonomous workflows, multi-agent coordination—apply to any domain where professionals spend more time on software than their core expertise.

---

## Contact & Contributions

This is a showcase repository—not open for external contributions. The production system contains proprietary integrations and customer data.

For inquiries about Hard-E's architecture, AI agent design patterns, or collaboration opportunities: fotiosmpouris@gmail.com

---

## License

This repository's documentation is provided for educational and reference purposes. The actual Hard-E production system, including all code, integrations, and data, is proprietary and not licensed for reuse.

See LICENSE file for detailed terms.

---

**Hard-E: AI that doesn't just assist—it works.**
