# Hard-E Agent Roles

This document describes the specialized agents in Hard-E's multi-agent architecture and their specific responsibilities in the production system.

> **Note:** This is a public-facing overview. Actual implementation details, prompts, API keys, and proprietary logic are maintained in private repositories.

---

## System Architecture

Hard-E employs a **hybrid multi-agent system** with two distinct processing modes:

- **Manual Streaming Mode**: For simple queries requiring fast, real-time responses (CRM lookups, knowledge retrieval, web search)
- **SDK Processing Mode**: For complex, stateful workflows requiring multi-turn conversations (script generation, customer creation)

All agents are coordinated by the **TriageAgent**, which analyzes user intent and routes requests to appropriate specialists.

---

## 1. TriageAgent (The Router)

**Role:** Primary orchestrator and intelligent router

**Purpose:**  
Analyze user queries and conversation history to route tasks to the appropriate specialized agent. Does not execute tasks directly—only routes via handoffs.

**Routing Logic Priority:**
1. **Ongoing Processes**: If ScriptingAgent or NewClientsAgent recently asked for details and user provides them
2. **Keyword Detection**: Specific terms trigger specific agents ("Dartmouth" → KnowledgeAgent, "create script" → ScriptingAgent)
3. **Conversation History Analysis**: Examines last 3-5 turns to detect continuation of previous conversations
4. **Explicit Requests**: Direct mentions of agent capabilities
5. **Clarification**: If intent unclear after all checks, ask for clarification

**Model:** GPT-4.1

**Tools:** None (routing only via agent handoffs)

**Example Behaviors:**
- User: "What did we discuss in the Dartmouth meeting?" → Route to **KnowledgeAgent**
- User: "Create a proposal script" → Route to **ScriptingAgent**
- User: "Add new customer John Doe" → Route to **NewClientsAgent**
- User: "What's the job status for 566679?" → Route to **LeapInteractionAgent**

---

## 2. LeapInteractionAgent (CRM Operations)

**Role:** CRM integration specialist

**Purpose:**  
Execute all Leap CRM interactions for existing customer and job data. Handles retrieval, updates, and notifications.

**Capabilities:**
- Job stage retrieval and updates
- Customer address lookup
- Job notes fetching and creation
- Comprehensive customer information gathering
- **@Mention notifications** in job notes (via undocumented API with smart fallback)

**Model:** GPT-4.1

**Key Innovation:**  
Smart fallback pattern—attempts RA Token authentication for notifications, gracefully falls back to standard V3 API if unavailable.

**Example Behaviors:**
- User: "What's the status of job 566679?" → Retrieve current job stage
- User: "Get notes for the Johnson project" → Fetch all job notes
- User: "Move job to Contract stage" → Update stage with optional workflow trigger
- User: "Add note: @Scott Kelly please follow up" → Create note with notification

**Special Features:**
- **@Mention Format**: Transforms "@Scott Kelly" → "@[u:855997]" for notifications
- **Stage Updates**: Uses numeric stage IDs (11 stages from "Lead" to "Paid")
- **Comprehensive Lookup**: Single query returns customer + job + notes

---

## 3. KnowledgeAgent (Internal Knowledge Base)

**Role:** Company knowledge and training specialist

**Purpose:**  
Answer questions based on company training materials and project-specific knowledge bases stored in S3.

**Data Sources:**
- `s3://hard-e-transcripts/training/` - General sales training materials
- `s3://hard-e-transcripts/Yarmouth/` - Dartmouth Place Project documentation
- `s3://hard-e-transcripts/meeting_summaries/` - Sales meeting notes (planned)
- `s3://hard-e-transcripts/price_book/` - Pricing catalog data

**Process:**
1. Read all documents from configured S3 prefixes (up to 200K characters)
2. Pass content to GPT-4.1 with synthesis prompt
3. Return grounded answers based **only** on provided documents

**Model:** GPT-4.1

**Streaming Support:** Yes—composition happens in SDK model, enabling real-time text streaming

**Example Behaviors:**
- User: "What does the training say about LeadScout?" → Search training materials
- User: "What's our process for budget-conscious homeowners?" → Synthesize from training docs
- User: "Tell me about the Dartmouth Place Project" → Query project-specific knowledge base

**Grounding Principle:** Never hallucinates. If answer not in documents, explicitly states "This information is not in the available training materials."

---

## 4. ScriptingAgent (Video Script Generator)

**Role:** Stateful script generation specialist

**Purpose:**  
Manage multi-turn, stateful generation of both standard proposal scripts and short primer video scripts.

**State Management:**  
Uses `ScriptDraftState` dataclass to track:
- `job_id`, `client_name`, `project_type`, `focus_material`
- `context_fetched`, `job_description`, `job_notes`
- `draft`, `completed`, `audio_served` flags
- `draft_kind` ("final" or "primer")
- `last_generated_job_id` (for idempotency)

**Workflow:**
1. **Detection**: Identifies script request from initial query
2. **Data Collection**: Conversationally gathers required fields
3. **Context Fetching**: Retrieves job details from Leap CRM
4. **Script Type Decision**: Determines "primer" vs "standard" from keywords
5. **Generation**: Calls appropriate tool with all context
6. **Idempotency**: Reuses existing script if job_id unchanged

**Model:** GPT-4.1

**Key Feature:** Multi-turn conversation maintains context across 5-10 exchanges until all information collected

**Example Behaviors:**
- User: "Create a proposal script" → ScriptingAgent: "I'll need the Job ID"
- User: "Job 677789" → ScriptingAgent: [fetches context] "Now I need client name, project type, and focus material"
- User: "Client is Jane Smith, siding replacement, James Hardie" → [Generates full script]

**Primer Scripts:**
- Triggered by keywords: "primer", "intro", "introduction", "short"
- Length target: 1-2 minutes
- Personal, conversational tone
- Focus on "what to expect" messaging

**Standard Scripts:**
- Comprehensive proposal video scripts
- Includes job context from Leap CRM
- Uses patterns from 90+ historical transcripts
- Personalized to project specifics

---

## 5. NewClientsAgent (Customer Creation Specialist)

**Role:** Natural language customer intake specialist

**Purpose:**  
Create new customers and optional jobs in Leap CRM via natural language input (voice or text).

**State Management:**  
Uses `NewClientState` dataclass to track:
- `first_name`, `last_name`, `email`, `phone`
- `address`, `city`, `state_id`, `zip_code`
- `job_description`
- `customer_id`, `completed`, `creation_triggered` flags
- `skipped` fields set (for "skip email" commands)

**LLM-Powered Extraction:**
- Uses **GPT-4.1-mini** with function calling for field extraction
- Parses natural language for customer details
- Handles speech artifacts: "emilyatdemo" → "emily@demo.com"
- Supports synonym mapping: "street" → "address"
- Flexible data entry: allows skipping optional fields

**Workflow:**
1. **Phase 1 - Customer Basics**: Collect minimum required data (name + phone)
2. **Phase 2 - Optional Job**: Ask if user wants to create first job
3. **Phase 3 - Create Objects**: Execute Leap API calls
4. **Confirmation**: Provide summary of created records

**Model:** GPT-4.1 (orchestration), GPT-4.1-mini (field extraction)

**Success Rate:** 95%+ for single-pass voice/text customer creation

**Example Behaviors:**
- User: "Add new customer John Doe, phone 617-555-1234, skip email" → Creates customer with only name + phone
- User: "Create customer Jane Smith, jane at demo dot com, 123 Main Street, Boston MA" → Normalizes email, maps state, creates full profile
- User: "What do you have so far?" → Shows current collected information
- User: "Start over" → Resets client state

**Special Features:**
- **Speech Artifact Handling**: Understands "at" = "@", "dot" = "."
- **State Code Mapping**: "MA" → Massachusetts (ID 21), not Missouri
- **Skip Support**: Recognizes "skip email", "unknown address", etc.
- **Minimum Requirements**: Only name + phone truly required

---

## 6. PricingAgent (Cost Calculation Specialist)

**Role:** Pricing catalog search and calculation specialist

**Status:** Developing capability (data quality improvements needed)

**Type:** Module-based (not an SDK Agent)—functions called directly via tools

**Purpose:**  
Search pricing catalog and calculate material/labor costs with area-based pricing.

**Data Source:**  
`s3://hard-e-transcripts/price_book/compiled/catalog.jsonl`

**Capabilities:**
- Material/labor lookup by name or SKU
- Token-based search with fuzzy fallback
- Area calculations (supports squares, sq ft, linear ft)
- Basis extraction from descriptions ("PER 3.12 SQ FT")
- Automatic unit conversions (1 square = 100 sq ft)
- Coverage inference for wraps/rolls

**Pricing Catalog Structure:**
- **Raw Data**: Labor rates, ABC Supply materials, general materials
- **Compiled Catalog**: Normalized JSONL with effective pricing calculations
- **Schema**: type, name, price, unit, SKU, vendor, description, basis fields, effective pricing

**Search Algorithm:**
- Token presence scoring (+10 per match)
- All terms present bonus (+40)
- Exact match bonus (+60)
- SKU direct hit priority (+100)
- Fuzzy fallback (difflib) if no scored results

**Example Behaviors:**
- User: "Price for HardieWrap" → Search catalog, return item details
- User: "Calculate cost for 12×21 ft decking" → Area calculation with basis extraction
- User: "How many rolls to cover 1000 sq ft?" → Coverage inference + quantity calculation

**Current Limitations:**
- Source data requires cleanup (unit mislabeling, inconsistent descriptions)
- "Pieces per square" parsing not yet implemented
- Multi-intent pricing (labor + material in one query) requires two turns

**Safety & Compliance:**  
Never hallucinates prices. If item not in catalog, explicitly states "Not found in pricing catalog."

---

## 7. AnalysisAgent (Transcript Pattern Specialist)

**Role:** Historical pattern analysis specialist

**Purpose:**  
Analyze patterns across 90+ distilled historical transcripts to identify common themes, objections, and best practices.

**Data Source:**  
`s3://hard-e-transcripts/distilled_transcripts/` (90 structured JSON files)

**Use Cases:**
- Identify common objections in sales calls
- Find typical video script structures
- Discover frequently mentioned competitor products
- Extract best practices from successful sales
- Understand typical sales conversation flow

**Model:** GPT-4.1

**Example Behaviors:**
- User: "What are common objections about fiber cement?" → Analyze transcripts for patterns
- User: "What's the typical script structure?" → Identify common sections and flow
- User: "Which competitors come up most?" → Extract competitive mentions

---

## 8. WebSearchAgent (External Information Specialist)

**Role:** External product information specialist

**Purpose:**  
Retrieve current product information, warranties, and specifications via web search.

**API:** Perplexity (sonar-pro model)

**Use Cases:**
- Latest product warranty terms
- Color options for siding products
- Manufacturer technical specifications
- Competitive product comparisons

**Model:** GPT-4.1 (orchestration), Perplexity sonar-pro (search)

**Why Perplexity:** Provides synthesized answers with citations rather than raw search results, better suited for conversational responses.

**Example Behaviors:**
- User: "What's the warranty on James Hardie siding?" → Search manufacturer info
- User: "What colors does CertainTeed offer?" → Retrieve current product line
- User: "Compare Hardie vs vinyl for wide exposure" → Competitive analysis

---

## 9. Autonomous Workflow System (Background Orchestrator)

**Role:** Autonomous task execution system

**Purpose:**  
Execute complete multi-step workflows without human intervention when triggered by external events.

**Current Implementation: Primer Video Pipeline**

**Status:** Fully operational (since August 2025)

**Trigger:** Email to monitored inbox with subject "Please produce Primer Video"

**Components:**
- **Email Listener Daemon** (`email_listener.py`): 60-second polling of Gmail inbox
- **Master Workflow Orchestrator** (`workflow_automations.run_autonomous_primer_workflow()`)

**Complete Workflow:**
1. Fetch customer and job context from Leap CRM
2. Generate personalized primer script (uses ScriptingAgent logic)
3. Generate two-stage TTS audio (OpenAI → ElevenLabs)
4. Upload audio to Cloudinary with unique asset ID
5. Build video transformation (music 30%, narration starts +8s)
6. Download rendered MP4
7. Upload to Google Drive with shareable link
8. Add job note in Leap with HTML link
9. 15-minute delay for Google Drive processing
10. Send personalized client email with video link
11. Comprehensive cleanup (files + cloud assets)

**Total Time:** 18-22 minutes (end-to-end)

**Error Handling:** try/except/finally with automatic cleanup regardless of outcome

**Key Features:**
- Works whether job stage changed via Hard-E or manually in Leap UI
- Decoupled architecture (Leap doesn't need to know about Hard-E)
- Comprehensive resource cleanup prevents leaks
- Audit trail for compliance

---

## Planned Agents (Future Development)

### ImageAnalysisAgent
**Status:** Planned  
**Purpose:** Analyze job site photos for property features, damage assessment, and measurements  
**Technology:** Multimodal LLM (GPT-4 Vision or equivalent)  
**Planned Capabilities:**
- Identify siding types, trim styles, architectural features
- Assess damage severity and flag safety hazards
- Extract measurements from images
- Answer interactive questions about photos

### SalesDirectorCloneAgent
**Status:** Planned  
**Purpose:** Provide insights from sales meeting summaries and strategic discussions  
**Data Source:** `s3://hard-e-transcripts/meeting_summaries/` (regularly updated)  
**Planned Capabilities:**
- Query recent strategic decisions from meetings
- Retrieve talking points for campaigns
- Access team best practices and lessons learned
- Provide context on company priorities and focus areas

### HoverIntegrationAgent
**Status:** Planned  
**Purpose:** Access 3D models and precise measurements from Hover platform  
**Planned Capabilities:**
- Retrieve exact square footage for walls, roof, trim
- Pull dimensions for script generation
- Use measurements for pricing calculations
- Potentially embed 3D models in proposals

---

## Cross-Cutting Patterns

All agents share common design principles:

### 1. Grounding & Safety
- **No Hallucination**: Agents defer to actual data sources (CRM, S3, APIs)
- **Explicit Uncertainty**: If information unavailable, agents say so clearly
- **Source Attribution**: Responses linked to specific data sources when relevant

### 2. Tool Calling
- All agents use function calling for structured operations
- Tools have explicit input/output schemas
- Error handling with graceful degradation

### 3. Refinement (Optional)
- All agent responses can be refined via xAI Grok (grok-3-mini-beta)
- Configurable via `ENABLE_GROK_REWRITE` flag
- Ensures consistent persona and professional polish

### 4. State Management
- **Stateless Agents**: TriageAgent, LeapInteractionAgent, KnowledgeAgent, AnalysisAgent, WebSearchAgent
- **Stateful Agents**: ScriptingAgent (ScriptDraftState), NewClientsAgent (NewClientState)
- State passed via SDK context parameter for multi-turn conversations

### 5. Streaming Support
- **KnowledgeAgent**: Supports text streaming via SDK model composition
- **Manual Streaming Flow**: Used for simple queries across all agents
- **SDK Processing**: Used for complex workflows requiring state

### 6. Traceability
- All queries logged to S3 (query_log.txt)
- Workflow execution logged comprehensively
- Agent decisions and tool calls tracked
- Performance metrics captured (first text time, full text time, audio time)

---

## System Evolution

This agent architecture is designed for extensibility:

**Easy to Add:**
- New specialized agents (define role, tools, handoff logic)
- New tools for existing agents (add to tools list)
- New data sources (expand S3 prefixes or API integrations)
- New autonomous workflows (follow primer pipeline pattern)

**Current Production State:**
- 8 operational agents (7 SDK agents + 1 module-based)
- 30+ tools across agents
- 1 autonomous workflow (primer video pipeline)
- Hybrid processing architecture (manual streaming + SDK)

**Next-Generation (v3.0):**
- Same agent logic, new infrastructure
- React frontend + FastAPI backend
- Real-time voice integration
- Multi-user concurrent support
- Redis state management

---

**This design document represents the production system as of November 2025. Actual implementation includes proprietary prompts, fine-tuned parameters, and integration details not shown here.**


