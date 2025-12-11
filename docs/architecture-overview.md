# Hard-E Architecture Overview

Hard-E is an AI-powered Sales Director for residential contracting companies (specifically Patriot Contracting Inc.). It automates and augments critical sales workflows through natural conversation:

- **Script Generation** - Proposal and primer video scripts with job context
- **Customer Creation** - CRM integration with 95%+ accuracy from voice input
- **Knowledge Access** - Instant answers from company training materials
- **Pricing Calculations** - Material and labor cost lookups with area-based math
- **CRM Operations** - Job stages, notes, customer lookup, @mention notifications
- **Autonomous Workflows** - Complete primer video generation and delivery
- **Web Search** - External product information and specifications
- **Transcript Analysis** - Pattern recognition across historical sales conversations

This document describes the **production architecture** of the system as of November 2025.

---

## 1. Architecture Layers

Hard-E operates as a four-layer system with hybrid processing capabilities:

### 1.1. Interface Layer

**Current (Production):**
- **Streamlit Web UI** - Browser-based interface at https://streaming.harde.app
- **Voice Input** - Real-time speech-to-text via OpenAI Whisper
- **Audio Output** - Text-to-speech with session-based playback
- **Mobile Support** - Android validated, iOS with autoplay limitations

**Future (v3.0 - In Development):**
- **React SPA** - Modern TypeScript frontend with Vite
- **Real-Time Voice** - Sub-250ms latency via Retell or Vapi webhooks
- **WebSocket/SSE** - Bi-directional streaming for instant responses
- **Accessibility** - ARIA-compliant, keyboard navigation

### 1.2. Orchestration & Agent Layer

**Hybrid Processing Model:**

Hard-E uses intelligent routing to determine processing path:
```
User Query → Router Analysis
    ↓
    ├─→ Simple Query? → Manual Streaming Flow
    │   • Direct OpenAI API (stream=True)
    │   • Real-time text display (200-500ms first token)
    │   • Direct tool responses or model composition
    │   • Use cases: CRM lookups, knowledge queries, web search
    │
    └─→ Complex Workflow? → SDK-Based Processing
        • OpenAI Agents SDK with multi-agent handoffs
        • Robust state management via context passing
        • Multi-turn conversation support
        • Use cases: Script generation, customer creation
```

**Agent Registry:**

1. **TriageAgent** (Router)
   - Primary routing logic based on keywords, conversation history, active state
   - Coordinates handoffs to specialist agents
   - Model: GPT-4.1

2. **LeapInteractionAgent** (CRM Operations)
   - Job stages, customer data, notes management
   - Undocumented API integration (@mentions, stage updates)
   - Smart fallback: RA Token → V3 API
   - Model: GPT-4.1

3. **KnowledgeAgent** (Internal Q&A)
   - Searches S3 knowledge bases (training/, Yarmouth/, meeting_summaries/)
   - GPT-4 synthesis for grounded answers
   - Streaming-enabled via SDK model composition
   - Model: GPT-4.1

4. **ScriptingAgent** (Video Scripts)
   - **Stateful**: Uses ScriptDraftState dataclass
   - Multi-turn conversation for data collection
   - Generates proposal scripts and primer scripts
   - Fetches job context from Leap CRM automatically
   - Idempotency guards prevent regeneration
   - Model: GPT-4.1

5. **NewClientsAgent** (Customer Creation)
   - **Stateful**: Uses NewClientState dataclass
   - LLM-powered field extraction (GPT-4.1-mini with function calling)
   - Handles speech artifacts and flexible data entry
   - Supports "skip" commands for optional fields
   - 95%+ success rate from voice input
   - Models: GPT-4.1 (orchestration), GPT-4.1-mini (extraction)

6. **AnalysisAgent** (Transcript Patterns)
   - Analyzes 90+ distilled historical transcripts
   - Identifies common objections, script structures, best practices
   - Model: GPT-4.1

7. **WebSearchAgent** (External Information)
   - Perplexity API (sonar-pro) for product specs and warranties
   - Synthesized answers with citations
   - Model: GPT-4.1 (orchestration), Perplexity sonar-pro (search)

8. **PricingAgent** (Cost Calculations)
   - **Module-based** (not SDK Agent)
   - Searches compiled S3 pricing catalog (JSONL format)
   - Token-based search with fuzzy fallback, SKU priority
   - Area-based calculations (squares, sq_ft, lin_ft)
   - Basis extraction from descriptions
   - **Status**: Developing (data quality improvements needed)

**State Management:**
```python
# Current (Streamlit)
st.session_state = {
    "script_state": ScriptDraftState(),     # Multi-turn script generation
    "new_client_state": NewClientState(),   # Customer creation workflow
    "conversation_history": [],             # Full conversation
    "timer_state": {},                      # Performance metrics
    "last_primer_audio_path": str,          # Audio caching
    "last_primer_audio_meta": dict          # Cache validation
}

# Planned (v3.0 - Redis)
redis_keys = {
    "session:{session_id}:script": ScriptDraftState,
    "session:{session_id}:client": NewClientState,
    "session:{session_id}:history": List[Message]
}
```

### 1.3. Tools & Integrations Layer

**CRM Integration (Leap/JobProgress):**

- **Standard V3 API:**
  - Customer creation/retrieval
  - Job listing and details
  - Basic note management
  - Authentication: Bearer token

- **Undocumented Public API** (Reverse-engineered via browser DevTools):
  - Job stage updates: `POST /api/v1/jobs/stage`
  - @Mention notifications: `@[u:USER_ID]` format
  - Email sending with template injection
  - Authentication: RA Token (manually refreshed)

**Knowledge Base (S3):**
```
s3://hard-e-transcripts/
├── training/                    # General sales training (~6 KB)
├── Yarmouth/                    # Project-specific knowledge
├── distilled_transcripts/       # 90 structured JSONs
├── meeting_summaries/           # Planned: sales meeting notes
├── price_book/
│   ├── raw/                     # Vendor price sheets
│   └── compiled/                # Normalized JSONL catalog
└── logs/
    └── query_log.txt            # All user queries (timestamped)
```

**External APIs:**

- **OpenAI**: GPT-4.1, GPT-4.1-mini, Whisper (speech-to-text), TTS (text-to-speech)
- **xAI Grok**: grok-3-mini-beta (response refinement, configurable)
- **ElevenLabs**: Speech-to-Speech (eleven_multilingual_sts_v2) for primer voiceovers
- **Perplexity**: sonar-pro (web search with synthesized answers)
- **Cloudinary**: Video rendering and CDN delivery
- **Google Drive**: Video delivery via service account with shareable links
- **Gmail IMAP**: Email monitoring for autonomous workflow triggers

**Audio Pipeline:**
```
Standard Responses:
User Query → GPT-4.1 Response → OpenAI TTS (tts-1, shimmer) → MP3 → Playback

Primer Videos (Two-Stage):
Script → OpenAI TTS (tts-1, onyx) → intermediate.mp3
      → ElevenLabs S2S (eleven_multilingual_sts_v2) → final_primer_audio.mp3
      → Fallback to OpenAI-only if ElevenLabs unavailable
```

**Video Rendering (Primer Videos):**
```
Cloudinary Pipeline:
1. Upload audio with unique ID (primer/audio/{slug}-{job_id}-{timestamp})
2. Build transformation URL (base video + audio overlay)
   - Base music volume: 30%
   - Narration start: +8 seconds
3. Download rendered MP4
4. Upload to Google Drive
5. Generate shareable link
6. Cleanup local files and Cloudinary assets
```

### 1.4. Data & Analytics Layer

**Structured Data:**

- **State Objects:**
  - `ScriptDraftState`: job_id, client_name, project_type, focus_material, flags
  - `NewClientState`: contact info, address, job_description, completion flags

- **Distilled Transcripts** (S3):
  - Format: JSON with section_summary structure
  - Count: 90 validated files
  - Use: Script inspiration, pattern analysis

- **Pricing Catalog** (S3):
  - Raw: Vendor JSON files (labor_rates, materials)
  - Compiled: Normalized JSONL with effective pricing
  - Fields: type, name, price, unit, SKU, vendor, description, basis extraction

**Query Logging:**
```
Implementation: log_user_query_to_s3()
Format: "YYYY-MM-DD HH:MM:SS - QUERY: [user_text]"
Location: s3://hard-e-transcripts/logs/query_log.txt
Purpose: Usage analysis, feature prioritization
Status: Operational since July 2025
```

**Performance Metrics:**
```python
timer_state = {
    "start_time": timestamp,
    "first_text_time": timestamp,      # First streaming token
    "full_text_time": timestamp,       # Complete text response
    "audio_done_time": timestamp       # TTS generation complete
}
# Displayed in UI as three-column latency breakdown
```

**Analytics (Planned):**

- Query volume by agent/feature
- Response time trends
- Tool success/failure rates
- User satisfaction indicators
- Error pattern recognition

---

## 2. Sequence: Typical Interactions

### 2.1. Simple Knowledge Query
```
1. User: "What's our process for qualifying budget-conscious homeowners?"
2. Interface: Text/voice input captured
3. Router: Detects knowledge query → Manual streaming flow
4. KnowledgeAgent: 
   - Calls tool_get_training_answer()
   - Reads S3 training documents
   - GPT-4 synthesizes grounded answer
5. Response: Streams word-by-word to UI (300-500ms first token)
6. Audio: OpenAI TTS generates response.mp3 → Autoplay
7. Logging: Query logged to S3 with timestamp
```

### 2.2. Multi-Turn Script Generation
```
1. User: "Generate a proposal script for the Johnson project"
2. Router: Detects "generate script" → SDK processing
3. TriageAgent: Handoff to ScriptingAgent
4. ScriptingAgent (Turn 1):
   - Checks ScriptDraftState (empty)
   - Response: "I'll need the Leap Job ID"
5. User: "Job 6142599"
6. ScriptingAgent (Turn 2):
   - Calls tool_set_script_field(job_id="6142599")
   - Calls tool_fetch_leap_context() → Gets description + notes
   - Response: "Got it. Now I need client name, project type, focus material"
7. User: "Client is Foti, siding replacement, James Hardie"
8. ScriptingAgent (Turn 3):
   - Saves all fields to state
   - Detects all required data present
   - Calls tool_generate_final_script()
   - Loads representative transcript from S3 for inspiration
   - GPT-4 generates personalized script with job context
9. Response: Full script displayed, audio generated
10. State: script_state.completed = True (prevents regeneration)
```

### 2.3. Customer Creation from Voice
```
1. User (voice): "Add new customer Jane Smith, phone 617-555-1234, 
                  email jane at demo dot com, skip the address"
2. Whisper: Transcribes to text (handles "at" speech pattern)
3. Router: Detects "add customer" → SDK processing
4. TriageAgent: Handoff to NewClientsAgent
5. NewClientsAgent:
   - Calls llm_extract_client_fields() with GPT-4.1-mini
   - Function calling extracts:
     * first_name: "Jane"
     * last_name: "Smith"  
     * phone: "617-555-1234"
     * email: "jane@demo.com" (normalized from "jane at demo dot com")
     * address: None (user said "skip")
   - Updates NewClientState with parsed data
   - Marks address as "skipped" (intentionally blank)
   - Response: "I have Jane Smith, 617-555-1234, jane@demo.com. 
                Address skipped. Create this customer?"
6. User: "Yes"
7. NewClientsAgent:
   - Calls tool_create_leap_customer()
   - Leap API creates customer with available fields
   - Response: "Customer created successfully. ID: 12345"
8. Audio: Confirmation played via TTS
```

### 2.4. Autonomous Primer Video Workflow
```
Trigger: Job stage changed to "Permission to Measure" in Leap CRM

1. Leap Automation: Sends email to poorpeoplepundit@gmail.com
   Subject: "Please produce Primer Video"
   Body: HTML with customer name in db-element span

2. Email Listener Daemon (email_listener.py):
   - Polling every 60 seconds
   - Detects new unread email with target subject
   - Parses HTML → Extracts "John Doe"
   - Resolves customer name to job_id via Leap API

3. Workflow Orchestrator (workflow_automations.py):
   Calls run_autonomous_primer_workflow("John Doe", "6142599")

4. Step 1: Fetch Context
   - Customer details from Leap
   - Job description and notes
   
5. Step 2: Generate Script
   - Calls generate_primer_script()
   - Personalizes with customer first name
   - Uses job context for relevance
   
6. Step 3: Generate Audio (Two-Stage)
   - OpenAI TTS → intermediate_openai_primer_<timestamp>.mp3
   - ElevenLabs S2S → final_primer_audio.mp3
   - Cleanup intermediate file
   
7. Step 4: Upload Audio to Cloudinary
   - Unique public_id: primer/audio/john-doe-6142599-1699123456
   
8. Step 5: Build Video
   - Cloudinary transformation URL
   - Base video + audio overlay
   - Music at 30%, narration starts at +8s
   
9. Step 6: Download MP4
   - Saves to /tmp/john-doe-video01.mp4
   
10. Step 7: Upload to Google Drive
    - Filename: "John Doe Video 01.mp4"
    - Returns shareable link
    
11. Step 8: Add Job Note in Leap
    - HTML link: <a href="{drive_link}">Click to view video</a>
    
12. Step 9: 15-Minute Delay
    - time.sleep(15 * 60)
    - Allows Google Drive to process video
    
13. Step 10: Send Client Email
    - Dynamic HTML template with personalized content
    - Video link embedded
    - Sent via Leap email API
    
14. Cleanup (finally block):
    - Remove local MP3 and MP4 files
    - Best-effort Cloudinary asset cleanup
    - Clear cached audio paths
    
Total Time: 18-22 minutes (mostly waiting for Drive)
Human Involvement: Zero
```

---

## 3. Deployment Model

### 3.1. Current Production (Streaming Architecture)

**Infrastructure:**
- **Compute**: AWS EC2 t3.small (2 vCPU, 2 GiB RAM, 20 GiB EBS)
- **OS**: Amazon Linux 2
- **Region**: us-east-2 (Ohio)
- **Storage**: S3 (hard-e-transcripts bucket, ~4 MB)
- **Networking**: Elastic IP (3.149.127.157), domain (streaming.harde.app)

**Application Stack:**
- **Framework**: Streamlit (1.44.0)
- **Python**: 3.9 in virtual environment (.venv)
- **SDK**: OpenAI Agents (>=0.0.17) for ResponseTextDeltaEvent support
- **Dependencies**: boto3, openai, elevenlabs, requests, pydub, toml

**Web Server:**
- **Nginx**: Reverse proxy on ports 80/443
- **SSL**: Let's Encrypt certificates (auto-renewal via certbot)
- **WebSocket**: Proxy headers for Streamlit functionality
- **Auth**: HTTP Basic Authentication via .htpasswd

**Process Management:**
- **Main App**: systemd service (harde-streaming.service)
  - Port: 8504
  - User: ec2-user
  - Auto-start on boot, restart on failure
  - Logs: `sudo journalctl -u harde-streaming -f`

- **Email Listener**: Daemonized background process
  - Command: `nohup python3 -u email_listener.py &`
  - Logs: nohup.out in project directory
  - Monitoring: `ps aux | grep email_listener`

**Secrets Management:**
- **File**: /home/ec2-user/hard-e-streaming/secrets.toml (chmod 600)
- **Contents**: API keys for OpenAI, xAI, Leap, Perplexity, ElevenLabs, Cloudinary
- **Service Accounts**: Google Drive (hard-e-drive-sa.json, chmod 600)
- **Tokens**: RA Token (ra_token.txt, manually refreshed)

**Monitoring:**
- **Query Logs**: S3 (query_log.txt with timestamps)
- **Performance Timers**: Real-time UI display (first text, full text, audio ready)
- **Error Tracking**: journalctl logs, nohup.out for email listener
- **Resource Usage**: Manual monitoring via `top`, `df -h`, `free -h`

### 3.2. Dual Environment Architecture (Development)

**Legacy Production:**
- Directory: /home/ec2-user/hard-e-production/
- Port: 8501
- URL: agent.harde.app
- SDK Version: openai-agents==0.0.4
- Status: Maintained but not actively used

**Current Production:**
- Directory: /home/ec2-user/hard-e-streaming/
- Port: 8504
- URL: streaming.harde.app
- SDK Version: openai-agents>=0.0.17
- Status: Primary production environment

**Coexistence Strategy:**
- Complete environment isolation (separate venvs, secrets, services)
- Independent systemd services
- Different package versions
- No shared files or state

### 3.3. Future Target (v3.0 - Next-Generation)

**New Technology Stack:**

**Frontend:**
- **Framework**: React 18 with TypeScript
- **Build Tool**: Vite (fast development, optimized builds)
- **Styling**: Tailwind CSS for utility-first styling
- **State**: React hooks (useState, useReducer, useContext)
- **Streaming**: SSE consumption via custom hooks
- **Voice**: WebAudio API for recording, WebSocket for vendor integration
- **URL**: https://next.harde.app/app/

**Backend:**
- **Framework**: FastAPI (async Python)
- **Server**: Uvicorn with multi-worker support (2-4 workers on t3.small)
- **API Design**: RESTful endpoints at /api/*
- **Streaming**: Server-Sent Events (SSE) for text, WebSocket for voice
- **State**: Redis 6 (localhost:6379) for distributed session management
- **Authentication**: Bearer tokens for API calls, HTTP Basic Auth for human users

**Infrastructure Changes:**
- **Same EC2**: Keep t3.small initially (upgradeable to t3.medium if needed)
- **New Services**: 
  - Redis: Port 6379 (internal only)
  - FastAPI: Port 8000
  - React dev: Port 3000 (Nginx proxy)
- **Nginx Routes**:
  - `/` → Streamlit (existing production, untouched)
  - `/app/` → React frontend (v3.0)
  - `/api/` → FastAPI backend (v3.0)

**Voice Integration:**
- **Vendor**: Retell or Vapi (selection in progress)
- **Target Latency**: Sub-250ms (ASR + processing + TTS)
- **Webhook**: POST /api/voice-hook with bearer token auth
- **Benefits**: True conversational AI, instant responses, natural dialogue

**Migration Strategy:**
```
Phase 1: Parallel Development (No Production Disruption)
- Build v3.0 in /home/ec2-user/hard-e-next/
- Streamlit production stays on streaming.harde.app
- Test at next.harde.app

Phase 2: Beta Testing
- Select users try v3.0 while majority stay on Streamlit
- Gather feedback, iterate rapidly
- Monitor performance under real load

Phase 3: Gradual Migration
- Feature parity validated
- User training completed
- Rollback plan ready (Streamlit still available)

Phase 4: Full Transition
- Deprecate Streamlit when v3.0 proven stable
- Keep old environment as emergency fallback
```

**Resource Estimates:**
```
Current Usage (t3.small, 2 GiB RAM):
- Streamlit app: ~300-400 MB
- Email listener: ~50 MB
- OS + services: ~400 MB
- Available: ~1.2 GiB

Projected v3.0 Usage:
- FastAPI (2 workers): ~240 MB
- Redis: ~60 MB
- React (Nginx serving static): ~40 MB
- Existing Streamlit (during transition): ~350 MB
- Total: ~690 MB (comfortable headroom on t3.small)

Scaling Trigger: Upgrade to t3.medium (4 GiB) if:
- Concurrent users > 5 sustained
- Memory usage > 75% for > 1 hour
- Response latency degrades
```

---

## 4. Security & Compliance

### 4.1. Current Security Posture

**Authentication:**
- **Human Users**: HTTP Basic Auth via Nginx (.htpasswd)
- **API Calls**: Bearer tokens for machine-to-machine (v3.0)
- **CRM**: Leap API keys and RA Token (manual rotation)

**Data Protection:**
- **Secrets**: File-based with restrictive permissions (chmod 600)
- **Transmission**: HTTPS enforced, Let's Encrypt certificates
- **Storage**: S3 bucket private, IAM role-based access
- **Logs**: Query logs contain customer names/emails (PII present)

**API Security:**
- **Rate Limiting**: None currently (relying on OpenAI/Leap limits)
- **Input Validation**: Basic sanitation, LLM-based extraction
- **Error Handling**: Try/except blocks prevent crashes, log full context

### 4.2. Known Security Gaps

⚠️ **Secrets Management**: File-based storage, not AWS Secrets Manager  
⚠️ **PII Retention**: No data retention policy or automatic deletion  
⚠️ **Access Control**: Single user account, no RBAC  
⚠️ **RA Token**: Manual refresh, no expiry monitoring  
⚠️ **Query Logs**: Indefinite retention of customer data in plain text

### 4.3. Planned Improvements

**Phase 1 (v3.0):**
- Migrate to AWS Secrets Manager for API keys
- Implement JWT/OAuth2 for multi-user authentication
- Add API rate limiting and request throttling
- Enhanced input validation and sanitation

**Phase 2 (Post-Launch):**
- Data retention policy (auto-delete logs after 1 year)
- PII redaction in logs and analytics
- RBAC (Admin, Sales Rep, Manager, Read-Only roles)
- Audit logging for sensitive operations
- Encryption at rest for S3 data

**Phase 3 (Enterprise):**
- GDPR/CCPA compliance framework
- Right to deletion (customer data purge mechanism)
- Data processing agreements with third-party services
- Security audits and penetration testing
- SOC 2 compliance preparation

### 4.4. Transcript Data Sensitivity

**Types of PII Captured:**
- Customer names, addresses, phone numbers, email addresses
- Project details (property information)
- Pricing discussions (budget ranges)
- Sales rep names and strategies

**Compliance Considerations:**
- **GDPR** (if EU customers): Right to access, rectification, deletion
- **CCPA** (California): Consumer rights to know, delete, opt-out
- **Industry Standards**: CRM data typically covered under business contracts

**Current Practice:**
- Transcripts stored in S3 (private bucket)
- Query logs capture all user input
- No automatic anonymization or redaction
- Retention indefinite

---

## 5. Why This Architecture Matters

### 5.1. Not Just a Chatbot

Hard-E represents a fundamental architectural shift in how AI assists sales teams:

**Traditional Chatbots:**
- Stateless question-answering
- Single-turn interactions
- Generic responses
- No integration with business systems
- Require learning new interfaces

**Hard-E Multi-Agent System:**
- Stateful, multi-turn conversations with context preservation
- Specialized agents mirror real organizational structure (Sales Director + team)
- Deep CRM integration (read and write operations)
- Autonomous workflow execution (completes tasks independently)
- Natural language interface (voice and text, no learning curve)

### 5.2. Voice-Native Design Philosophy

**Why Voice Matters:**

Sales representatives are mobile, in the field, meeting customers. They don't have time to:
- Open multiple software applications
- Navigate complex UI menus
- Type detailed notes on mobile devices
- Remember specific system commands

**Hard-E's Approach:**
- Speak naturally: *"Add new customer John Doe, phone 617-555-1234"*
- Handles speech artifacts: "jane at demo dot com" → "jane@demo.com"
- Context awareness: Remembers conversation history across turns
- Immediate feedback: Streaming text and audio responses
- Mobile-first: Works on any device with browser and microphone

### 5.3. Data as Institutional Knowledge

**The Flywheel Effect:**
```
More Conversations → More Transcripts → Better Pattern Recognition
    ↓                                           ↑
Better Scripts ← Improved Agents ← Richer Training Data
```

Every interaction with Hard-E:
1. **Captures** actual sales conversations (query logs, transcripts)
2. **Structures** unstructured data (distilled transcripts, entity extraction)
3. **Analyzes** patterns (common objections, successful strategies)
4. **Improves** future responses (script generation, coaching)

**Example:**
- 90 distilled transcripts inform script generation
- Representative transcript selection provides structure inspiration
- Common objection patterns guide coaching responses
- Pricing queries reveal gaps in catalog data

### 5.4. Extensibility Through Modularity

**Adding New Capabilities:**
```python
# Define new agent
new_agent = Agent(
    name="Hover Integration Agent",
    instructions="Retrieve 3D model data and measurements from Hover API",
    tools=[tool_fetch_hover_measurements, tool_get_3d_model],
    model="gpt-4.1"
)

# Register with TriageAgent
triage_agent.handoffs.append(new_agent)

# Add routing keywords
HOVER_KEYWORDS = ["hover", "3d model", "measurements", "roof dimensions"]
```

**No Core Architecture Changes Required:**
- Agents are self-contained
- Tools are pluggable
- State management scales (just add new dataclass)
- Router extends naturally with keywords

### 5.5. Auditability and Compliance

**Every Action Leaves a Trail:**

- **Query Logs**: All user inputs timestamped in S3
- **Conversation History**: Full context stored in session state
- **Tool Calls**: Which tools fired, with what parameters (in logs)
- **CRM Operations**: All Leap API calls logged
- **Autonomous Workflows**: Complete execution trace from trigger to completion

**Benefits:**
- Debugging: Reproduce any interaction from logs
- Training: Real conversations for onboarding new reps
- Analytics: Understand feature usage and gaps
- Compliance: Demonstrate data handling for audits
- Improvement: Identify where system fails or excels

### 5.6. Practical Cloud-Native Design

**Built on Proven Technologies:**

- **AWS EC2**: Industry-standard compute
- **S3**: Scalable, durable storage with versioning
- **Nginx**: Battle-tested web server and reverse proxy
- **OpenAI APIs**: Reliable, well-documented LLM services
- **Systemd**: Standard Linux service management

**No Exotic Dependencies:**
- Can migrate to different cloud providers
- Can swap out individual components (different LLM, different CRM)
- Can scale horizontally (add more EC2 instances behind load balancer)
- Can containerize (Docker/Kubernetes ready with minimal changes)

**Why This Matters:**
- Lower operational risk
- Easier to hire developers familiar with stack
- Vendor lock-in minimized
- Clear upgrade paths

---

## 6. Architectural Decisions & Trade-offs

### 6.1. Hybrid Processing (Streaming + SDK)

**Decision**: Maintain two processing paths instead of forcing all queries through one system

**Rationale**:
- Simple queries (CRM lookups, knowledge Q&A) benefit from sub-500ms streaming
- Complex workflows (script generation, customer creation) require robust state management
- Forcing everything through SDK would sacrifice responsiveness
- Forcing everything through manual streaming would sacrifice multi-turn capabilities

**Trade-off**: Added complexity in routing logic, but 300-500% better perceived responsiveness

### 6.2. Streamlit vs React/FastAPI

**Decision**: Build v3.0 with React/FastAPI rather than continuing to extend Streamlit

**Rationale**:
- Streamlit lacks real-time voice support (no sub-250ms latency possible)
- UI customization extremely limited (can't achieve desired UX)
- Single-user session model doesn't scale to team usage
- Frequent reruns cause performance issues

**Trade-off**: Significant development effort for v3.0, but unlocks critical capabilities

### 6.3. S3 vs Database for Knowledge

**Decision**: Store training materials, transcripts, and pricing catalog in S3, not a database

**Rationale**:
- Documents are read-heavy, rarely updated
- S3 provides versioning, durability, and scalability
- No schema migrations needed for document structure changes
- Cost-effective for infrequently accessed data

**Trade-off**: Query performance slower than database, but acceptable for use case

### 6.4. Email Trigger vs Webhooks for Autonomy

**Decision**: Use email listener polling instead of Leap CRM webhooks

**Rationale**:
- Email works regardless of how stage was changed (UI or API)
- No need to expose public webhook endpoint
- Simpler security model
- Decoupled from Leap internals

**Trade-off**: 60-second polling delay, but acceptable for non-urgent workflows

### 6.5. Two-Stage TTS for Primers

**Decision**: OpenAI TTS → ElevenLabs S2S instead of direct ElevenLabs TTS

**Rationale**:
- Direct ElevenLabs TTS from text had quality/consistency issues
- OpenAI TTS reliable for generating intermediate audio
- ElevenLabs S2S transforms existing audio to better voice
- Automatic fallback if ElevenLabs unavailable

**Trade-off**: Adds 30 seconds to processing time, but doubles audio quality

### 6.6. Module-Based Pricing vs SDK Agent

**Decision**: Implement pricing as module with direct function calls, not SDK Agent

**Rationale**:
- Pricing queries don't require stateful conversation
- Direct function calls are faster than agent orchestration
- Easier to test and validate calculations
- Can evolve independently from agent architecture

**Trade-off**: Inconsistent with agent pattern, but pragmatic for this use case

---

## 7. Future Architecture Evolution

### 7.1. Short-Term (v3.0 Launch - Q1 2026)

**Focus**: React/FastAPI migration with real-time voice

**Key Milestones**:
- ✅ Validate new server infrastructure (smoke test)
- 🔄 Port agent logic to FastAPI routers
- 🔄 Build React components (ChatWindow, Recorder, MessageBubble)
- 🔄 Implement Redis state management
- 🔄 Integrate voice vendor (Retell or Vapi)
- 🔄 Achieve parity with Streamlit features
- 🔄 Beta launch at next.harde.app

### 7.2. Mid-Term (Post-v3.0 - Q2-Q3 2026)

**Focus**: Enhanced capabilities and multi-user scaling

**Planned Features**:
- **Image Analysis Agent**: Job site photo analysis via GPT-4 Vision
- **Hover Integration**: 3D measurements and property modeling
- **Sales Director Clone Agent**: Insights from meeting summaries
- **Enhanced Pricing**: "Pieces per square" parsing, multi-intent queries
- **Multi-User Support**: RBAC, individual session isolation
- **Advanced Analytics**: Usage dashboards, success rate tracking

### 7.3. Long-Term (SaaS Platform - 2027+)

**Focus**: Template for industry-specific AI employees

**Vision**: Companies can deploy their own AI sales directors by providing:
1. Training materials (upload PDFs, docs)
2. CRM integration credentials (Salesforce, HubSpot, JobNimbus)
3. Pricing catalogs (upload vendor price sheets)
4. Custom workflow definitions (trigger conditions, actions)

**Architecture Changes Required**:
- Multi-tenant data isolation
- White-label UI customization
- Pluggable CRM adapters
- Industry-specific agent templates
- Self-service onboarding
- Usage-based pricing and quotas

**Target Markets**:
- Residential construction (proven)
- Commercial contracting
- Real estate sales
- Insurance sales
- HVAC services
- Roofing companies
- Any industry with complex sales processes and inconsistent expertise

---

## 8. Appendix: Technical Glossary

**Agent**: Specialized AI system with defined role, tools, and model. Can hand off to other agents.

**State Management**: Tracking conversation context across multiple turns using dataclasses (ScriptDraftState, NewClientState).

**Hybrid Processing**: Architectural pattern using both manual streaming (fast, simple queries) and SDK orchestration (complex, stateful workflows).

**Tool**: Python function that agents can call to interact with external systems (CRM, S3, APIs).

**Handoff**: Transfer of conversation control from one agent to another via SDK orchestration.

**RA Token**: "RA Token" authentication for Leap's undocumented public API, enables @mentions and stage updates.

**Basis Extraction**: Parsing pricing descriptions (e.g., "PER 3.12 SQ FT") to calculate effective per-unit pricing.

**Distilled Transcript**: Structured JSON summary of raw transcript with section-by-section content.

**Autonomous Workflow**: Complete multi-step process executed by Hard-E without human intervention (e.g., primer video pipeline).

**Streaming**: Real-time, word-by-word text display using Server-Sent Events (SSE) or direct API streaming.

**Idempotency**: Preventing duplicate execution (e.g., script regeneration) by checking state flags.

**Session State**: In-memory storage of conversation history and state objects, scoped to user session.

---

**Document Version**: 2.0  
**Last Updated**: November 17, 2025  
**Status**: Production Architecture (Streamlit), v3.0 In Development (React/FastAPI)  
**Next Review**: After v3.0 beta launch

---

*This architecture overview is intended for technical stakeholders, developers, and system architects. For strategic vision and business context, see the Hard-E Blueprint and Core Roles document. For comprehensive implementation details, see the 128-page Hard-E Progress and Technical Log.*
