# FORGE: Complete Architecture Document

**Version**: 1.0
**Date**: April 2026
**Status**: Planning -- Pre-Implementation

> This is the master reference document for the FORGE framework. It aggregates all architectural decisions from the planning phase and serves as the basis for detailed component specifications.

---

## Table of Contents

1. [Vision and Core Principles](#1-vision-and-core-principles)
2. [Repository Setup](#2-repository-setup)
3. [Three-Tier Agent Model](#3-three-tier-agent-model)
4. [Dual-Layer Ticket Contracts](#4-dual-layer-ticket-contracts)
5. [AIDEN Reuse Map](#5-aiden-reuse-map)
6. [Component Catalog](#6-component-catalog)
  - [A: Ticket Contracts + Guidance Templates](#component-a-ticket-contracts--guidance-templates)
  - [B: ROI Scoring Engine](#component-b-roi-scoring-engine)
  - [C: Kanban Board (API + MCP + UI)](#component-c-kanban-board-api--mcp--ui)
  - [D: Agent Runner + Pipeline Service](#component-d-agent-runner--pipeline-service)
  - [E: Support Widget](#component-e-support-widget)
  - [F: Bug Agent](#component-f-bug-agent)
  - [G: Feature Agent](#component-g-feature-agent)
  - [H: Coding Bridge](#component-h-coding-bridge)
  - [I: Communication Agent](#component-i-communication-agent)
  - [J: DevOps Dashboard](#component-j-devops-dashboard)
  - [K: MCP Server Extensions](#component-k-mcp-server-extensions)
7. [Project Structure](#7-project-structure)
8. [Build Phases](#8-build-phases)
9. [Technology Stack](#9-technology-stack)
10. [Critical Risks and Mitigations](#10-critical-risks-and-mitigations)
11. [Open Source Evaluation](#11-open-source-evaluation)
12. [Component Specification Index](#12-component-specification-index)

---

## 1. Vision and Core Principles

**FORGE** (Framework for Orchestrated, ROI-Guided Engineering) is a full-circle framework for developing and deploying AI-based software solutions. It targets tech-savvy founders whose primary skillset is economics, product, and sales -- enabling them to build, improve, run, and monitor a product based on an initial idea using a coordinated set of AI agents.

### Core Principles

**Tickets are prompts, not documents.** Every ticket's full payload is an agent-optimized context bundle with pre-assembled evidence, explicit instructions, success criteria, and constraints. Humans see only a thin summary on the board. The ticket IS the prompt for the next agent.

**Three agent tiers.** Gatherers (cheap, fast, high-volume) collect and structure data. Thinkers (powerful, expensive) analyze and decide. Communicators (empathetic, clear) translate outcomes into human language. Each tier uses models matched to its cognitive job.

**Module independence via contracts.** All modules communicate through well-defined ticket contracts on the Board API (REST or MCP). Any module can be swapped for a third-party tool (a different IDE, a different support widget, etc.) as long as it reads the input contract and writes the output contract. Modules never call each other directly.

**Built on AIDEN.** FORGE is built as a module layer on top of the existing AIDEN platform, inheriting LLM routing, RAG pipeline, queues, billing, auth, multi-tenancy, context engineering, DAG execution, tool execution, and the widget pattern. Only genuinely new functionality is built fresh.

**ROI drives everything.** Every feature, bug fix, and task is scored by ROI. The board is a prioritization tool, not just a task list. Agents default to minimal fixes and maximum pragmatism -- they are economists, not perfectionists.

---

## 2. Repository Setup

### GitHub Organization: `rootsandrivets`

Clean copy all 5 repos from `papa-bear-ventures` into the new `rootsandrivets` organization. Clean copies (not forks) are used to ensure fully independent access control -- fork visibility and collaborators are inherited from the parent org, which is not desired here.


| Source (papa-bear-ventures) | Copy (rootsandrivets) | Purpose                                                         |
| --------------------------- | --------------------- | --------------------------------------------------------------- |
| `aihub_api`                 | `forge_api`           | Backend API -- FORGE modules added here                         |
| `aihub_frontend`            | `forge_frontend`      | Frontend -- FORGE pages + support widget                        |
| `aihub_external_api`        | `forge_external_api`  | External API -- agent-facing board API + support widget backend |
| `aihub_gitops`              | `forge_gitops`        | Kubernetes / Rancher Fleet deployment configs                   |
| `aihub_mcp_servers`         | `forge_mcp_servers`   | MCP servers -- kanban, loki, code-analysis added                |


### Copy Strategy (with upstream remote for later merging)

Clean copies are independent repos initialized from a snapshot. An upstream remote is added manually for selective merging when needed:

```bash
# Initial copy (per repo, example for forge_api):
git clone git@github.com:papa-bear-ventures/aihub_api.git forge_api
cd forge_api
git remote remove origin
git remote add origin git@github.com:rootsandrivets/forge_api.git
git remote add upstream git@github.com:papa-bear-ventures/aihub_api.git
git push -u origin main

# Later: pull AIDEN improvements into FORGE:
git fetch upstream
git merge upstream/main       # or cherry-pick specific commits

# Later: push FORGE features back to AIDEN:
# Cherry-pick from forge_api into a local aihub_api clone, then PR to papa-bear-ventures
```

### Setup Tasks

1. Create the `rootsandrivets` GitHub organization
2. Create empty repos: `forge_api`, `forge_frontend`, `forge_external_api`, `forge_gitops`, `forge_mcp_servers`
3. Clone each AIDEN repo, re-point origin to rootsandrivets, add upstream remote, push
4. Update internal references: package names, Docker image tags, Fleet configs, CI workflows
5. Update environment variables and secrets for FORGE deployments
6. Set up GitHub Actions / CI for the new repos
7. Verify the copied codebase builds and deploys independently

### Code Namespacing Convention

All FORGE-specific code lives in `forge/` or `forge-*` namespaced directories within the existing AIDEN structure. This keeps FORGE code cleanly separated from inherited AIDEN code, making future merges conflict-free:

- `packages/types/src/forge/` -- FORGE TypeScript types
- `packages/schemas/src/forge/` -- FORGE Mongoose schemas
- `packages/validation/src/forge/` -- FORGE Zod contract schemas
- `packages/core/src/services/forge-*/` -- FORGE services
- `packages/core/src/modules/forge-*/` -- FORGE modules
- `src/tenant/controllers/forge-*.controller.ts` -- FORGE API routes
- Frontend: `src/pages/forge/`, `src/components/forge/`, `src/components/support-widget/`

---

## 3. Three-Tier Agent Model

Every agent in the system belongs to exactly one tier. Tiers differ in cognitive job, model requirements, cost profile, and how they interact with tickets.

### Tier Definitions


| Tier                  | Job                                                                  | Characteristics                                                                                        | Model Class                             | Cost/Call  |
| --------------------- | -------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ | --------------------------------------- | ---------- |
| **T1: Gatherers**     | Collect, extract, classify, structure raw data into clean context    | Fast, high-throughput; never make strategic decisions; error mode is missing data, not wrong decisions | Claude Haiku, GPT-4o-mini, Gemini Flash | $0.01-0.05 |
| **T2: Thinkers**      | Analyze, reason, plan, score, implement using pre-structured context | Powerful reasoning; receive pre-digested evidence; produce substantive decisions                       | Claude Sonnet/Opus, GPT-4o, Gemini Pro  | $0.10-1.00 |
| **T3: Communicators** | Translate technical outcomes into empathetic, clear human language   | Natural writing, tone matching, personalization; read the full resolution chain                        | Claude Sonnet, GPT-4o-mini              | $0.01-0.05 |


### Agent Inventory by Tier

**Tier 1 Agents (Gatherers)**:

- **Support Classifier**: Raw user conversation → classification (bug/feature/question) + structured IntakeContract
- **Bug Evidence Collector**: Classified bug ticket → evidence package (error logs, stack traces, similar tickets, affected user count)
- **Feature Context Collector**: Classified feature request → context package (similar features, request frequency, competitive references)
- **Code Context Mapper**: Execution-ready ticket → codebase context (affected files, recent commits, test coverage, dependencies)
- **Deploy Status Collector**: Merged PR → deployment status, health checks, rollback readiness

**Tier 2 Agents (Thinkers)**:

- **Support Responder**: User question + RAG context → helpful answer or escalation decision
- **Bug Analyst**: Structured evidence package → root cause, fix tiers (hotfix/proper/refactor), ROI score, coding guidance
- **Feature Analyst**: Structured feature context → user stories, acceptance criteria, RICE score, coding guidance
- **Coding Agent**: Analysis + fix plan + code context → implementation (branch, PR, tests)

**Tier 3 Agents (Communicators)**:

- **Resolution Notifier**: Full handoff chain → personalized user email/notification
- **Board Summarizer**: Ticket state changes → brief human-readable summary for the board
- **Release Notes Generator**: Batch of resolved tickets → user-facing changelog

### Why Splitting Matters

What traditionally would be "one Bug Agent" is actually two agents: a T1 gatherer that pulls logs, finds similar tickets, and assembles evidence, then a T2 thinker that reads the pre-digested evidence and produces the analysis. This means:

- The expensive Tier 2 model gets clean, dense input instead of wasting context on raw log parsing
- Tier 1 runs at high volume on cheap models
- Each tier is independently testable and replaceable
- You can improve gathering without touching analysis and vice versa

### Tier Interaction Pattern

Tiers interact only through tickets:

1. T1 agent enriches `agentLayer.context` (structured evidence)
2. T2 agent reads that context and writes `agentLayer.analysis` + rewrites `agentLayer.guidance` for the next agent
3. T3 agent reads the full handoff chain and produces human-facing text

### Model Routing

Model routing is configured per agent (stored in database, not hardcoded):

```typescript
const modelRouting = {
  // TIER 1: Fast and cheap
  'support-classifier':        { tier: 'gatherer',      model: 'claude-haiku',  maxTokens: 4000,   dailyBudgetCents: 500  },
  'bug-evidence-collector':    { tier: 'gatherer',      model: 'claude-haiku',  maxTokens: 8000,   dailyBudgetCents: 200  },
  'feature-context-collector': { tier: 'gatherer',      model: 'gemini-flash',  maxTokens: 8000,   dailyBudgetCents: 200  },
  'code-context-mapper':       { tier: 'gatherer',      model: 'claude-haiku',  maxTokens: 16000,  dailyBudgetCents: 300  },

  // TIER 2: Powerful reasoning
  'support-responder':         { tier: 'thinker',       model: 'claude-sonnet', maxTokens: 16000,  dailyBudgetCents: 2000 },
  'bug-analyst':               { tier: 'thinker',       model: 'claude-sonnet', maxTokens: 32000,  dailyBudgetCents: 1000 },
  'feature-analyst':           { tier: 'thinker',       model: 'claude-sonnet', maxTokens: 32000,  dailyBudgetCents: 1000 },
  'coding-agent':              { tier: 'thinker',       model: 'claude-opus',   maxTokens: 64000,  dailyBudgetCents: 5000 },

  // TIER 3: Good writing, empathetic
  'resolution-notifier':       { tier: 'communicator',  model: 'claude-sonnet', maxTokens: 4000,   dailyBudgetCents: 300  },
  'board-summarizer':          { tier: 'communicator',  model: 'claude-haiku',  maxTokens: 2000,   dailyBudgetCents: 200  },
  'release-notes-generator':   { tier: 'communicator',  model: 'claude-sonnet', maxTokens: 16000,  dailyBudgetCents: 500  },
};
```

This routing sits on top of AIDEN's existing `ModelRouterService` and `TriageService`.

---

## 4. Dual-Layer Ticket Contracts

Every ticket has two layers: a thin human summary and a deep agent-optimized payload.

### Human Layer (Board Card View)

This is what a founder sees scrolling the kanban board:

```typescript
interface HumanLayer {
  title: string;                          // max 80 chars, plain language
  summary: string;                        // 1-2 sentences
  type: 'feature' | 'bug' | 'support' | 'task';
  priority: 'P0' | 'P1' | 'P2' | 'P3';
  roiScore: number | null;               // 0-100, shown as color-coded badge
  status: string;                         // current list name
  pipelineStage: string;                  // human-readable stage
  waitingOn: 'agent' | 'human' | 'external' | null;
  humanActionRequired: string | null;     // e.g. "Approve fix plan"
  linkedUsers: { name: string; email: string }[];
  overriddenByHuman: boolean;
  createdAt: string;
  updatedAt: string;
}
```

### Agent Layer (Full Payload)

This is the context bundle agents consume via MCP:

```typescript
interface AgentLayer {
  contractVersion: string;               // semver for schema evolution

  // WHO CREATED THIS
  source: {
    module: string;                       // 'support-widget' | 'bug-agent' | etc.
    triggerEvent: string;
    createdAt: string;
  };

  // TIER 1 WRITES HERE: Pre-assembled context
  context: {
    originalUserInput: string;
    conversationHistory?: ConversationTurn[];
    environment?: {
      browser?: string;
      os?: string;
      appUrl?: string;
      appVersion?: string;
      userId?: string;
      tenantId?: string;
      timestamp?: string;
    };
    errorData?: {
      errorMessage?: string;
      stackTrace?: string;
      logSnippets?: LogEntry[];
      affectedEndpoints?: string[];
      errorFrequency?: string;
      firstSeen?: string;
      lastSeen?: string;
    };
    attachments: AgentAttachment[];
    relatedTickets: RelatedTicket[];
    codebaseContext?: {
      affectedFiles?: string[];
      recentCommits?: CommitSummary[];
      relevantCode?: CodeSnippet[];
    };
    extensions: Record<string, unknown>;  // module-specific data
  };

  // TIER 2 WRITES HERE: Decisions, scores, plans
  analysis?: {
    summary: string;
    rootCause?: string;
    bestPractices?: Reference[];
    fixPlan?: {
      tiers: {
        hotfix: { description: string; effort: string; risk: string };
        properFix: { description: string; effort: string; risk: string };
        refactor: { description: string; effort: string; risk: string };
      };
      recommended: 'hotfix' | 'properFix' | 'refactor';
      reasoning: string;
    };
    specification?: {
      userStories: string[];
      acceptanceCriteria: string[];
      technicalNotes: string;
      outOfScope: string[];
    };
    roiBreakdown?: ROIBreakdown;
  };

  execution?: {
    branch?: string;
    pullRequestUrl?: string;
    commits?: CommitSummary[];
    filesChanged?: string[];
    testResults?: { passed: number; failed: number; skipped: number };
    lintClean?: boolean;
  };

  // INSTRUCTIONS FOR THE NEXT AGENT (rewritten at each stage)
  guidance: {
    targetTier: 'gatherer' | 'thinker' | 'communicator';
    targetAgent: string;
    role: string;                         // "You are a bug analyst..."
    objective: string;
    steps: string[];
    successCriteria: string[];
    constraints: string[];                // append-only across pipeline
    contextPriority: {                    // for ContextAssembler token budgeting
      essential: string[];
      helpful: string[];
      optional: string[];
    };
    outputFormat: {
      zodSchemaRef: string;
      requiredFields: string[];
    };
  };

  // CHAIN OF REASONING (grows as ticket moves through pipeline)
  handoffNotes: HandoffNote[];

  // TIER 3 WRITES HERE: Delivery status
  delivery?: {
    deployment?: { environment: string; status: string; url?: string; deployedAt: string };
    notification?: { sent: boolean; channels: string[]; recipients: string[]; sentAt?: string };
  };
}

interface HandoffNote {
  fromModule: string;
  fromTier: 'gatherer' | 'thinker' | 'communicator';
  timestamp: string;
  summary: string;                        // 2-3 sentences: what I did, what I found
  confidence: number;                     // 0-1
  dataAdded: string[];                    // which agentLayer fields were added/updated
  openQuestions: string[];
  warnings: string[];
  nextStep: string;
  costs: { tokensUsed: number; estimatedCost: number; timeSpent: number };
}

interface AgentAttachment {
  id: string;
  filename: string;
  mimeType: string;
  url: string;
  description: string;                    // agent-readable: what this shows
  relevance: string;                      // why this matters for the next agent
}

interface RelatedTicket {
  ticketId: string;
  title: string;
  similarity: number;                     // 0-1
  relationship: 'duplicate' | 'related' | 'blocks' | 'blocked_by';
  summary: string;                        // what happened with that ticket
}
```

### Guidance Templates

Pre-built prompt templates stored in `packages/core/src/modules/forge-contracts/templates/`. Each template provides the skeleton for agent instructions at a specific pipeline transition:

```
templates/
  tier1/
    raw-to-classification.ts        # User message → Support Classifier
    classification-to-evidence.ts   # Classified ticket → Evidence Collector
    analysis-to-code-context.ts     # Analyzed ticket → Code Context Mapper
    pr-to-deploy-status.ts          # Merged PR → Deploy Status Collector
  tier2/
    evidence-to-bug-analysis.ts     # Evidence package → Bug Analyst
    context-to-feature-analysis.ts  # Feature context → Feature Analyst
    analysis-to-implementation.ts   # Full context → Coding Agent
    question-to-response.ts         # RAG context → Support Responder
  tier3/
    resolution-to-notification.ts   # Handoff chain → Resolution Notifier
    changes-to-board-summary.ts     # State changes → Board Summarizer
    batch-to-release-notes.ts       # Resolved tickets → Release Notes Generator
```

Templates contain: `role`, `steps`, `successCriteria`, `constraints` (append-only), and `contextPriority`. When an agent hands off, it pulls the relevant template, fills in context-specific details, and writes it into the ticket's `guidance` field.

### The Handoff Chain Example (Bug Ticket)

```
Step 1: Support Widget creates ticket
  T1 Support Classifier writes: context (conversation, environment, screenshots)
  Guidance filled: "classification-to-evidence" template for Bug Evidence Collector

Step 2: Evidence collection
  T1 Bug Evidence Collector writes: context.errorData (logs, stack traces, frequency)
  Guidance filled: "evidence-to-bug-analysis" template for Bug Analyst

Step 3: Bug analysis
  T2 Bug Analyst writes: analysis (root cause, fix tiers, ROI), handoffNotes (reasoning)
  Guidance filled: "analysis-to-code-context" template for Code Mapper

Step 4: Code context assembly
  T1 Code Context Mapper writes: context.codebaseContext (files, tests, deps)
  Guidance filled: "analysis-to-implementation" template for Coding Agent

Step 5: Implementation
  T2 Coding Agent writes: execution (branch, PR, tests), handoffNotes
  Ticket moves to "In Review" (human approval gate)

Step 6: After approval + deploy
  T3 Resolution Notifier reads full handoff chain, sends personalized email
  Ticket moves to "Done"
```

By Step 6, the Communication Agent has the full context: what the user reported, what evidence was found, what the root cause was, what was fixed, and how -- enabling a genuinely helpful notification like "Your Safari login issue has been fixed. The problem was a cookie setting that Safari 18 handles differently. The fix is live now."

---

## 5. AIDEN Reuse Map

### What We Get for Free (Zero New Code)


| FORGE Need                 | AIDEN Component                            | Location                                             |
| -------------------------- | ------------------------------------------ | ---------------------------------------------------- |
| LLM calls for all agents   | ModelRouterService + metered providers     | `packages/core/src/services/llm/`                    |
| RAG for Support Widget Q&A | RAGService + SmartRAGRouter                | `packages/core/src/services/rag/`                    |
| Async agent work           | BullMQ FlowQueues + workers                | `packages/core/src/modules/queue/`                   |
| Cost tracking per agent    | Billing service + limits + daily budgets   | `packages/core/src/services/billing/`                |
| Agent action audit trail   | Audit service + Redis buffer + persistence | `packages/core/src/services/audit/`                  |
| Auth + project isolation   | JWT middleware + tenant DB isolation       | Auth middleware + `tenant_{env}_{id}` pattern        |
| MongoDB models             | Mongoose schema patterns                   | `packages/schemas/`                                  |
| Zod validation             | Validation patterns                        | `packages/validation/`                               |
| MCP tool calling           | MCPClient + ToolExecutor                   | `packages/core/src/modules/tools/`                   |
| System health              | Health aggregation + New Relic             | `packages/core/src/services/health/` + `monitoring/` |


### What We Adapt (Minor Extensions)


| FORGE Need               | AIDEN Component                                      | Extension Needed                                           |
| ------------------------ | ---------------------------------------------------- | ---------------------------------------------------------- |
| Tier-based model routing | TriageService (cheap→powerful escalation)            | Add tier-aware config: gatherer=haiku, thinker=sonnet      |
| Agent prompt assembly    | ContextAssembler + TokenBudgetAllocator              | Add `TicketContextProvider` + `TicketGuidanceProvider`     |
| Agent execution loop     | OrchestratorService (tool-calling loop)              | Add ticket-aware start/stop + result capture               |
| Agent pipeline as DAG    | ExecutionEngine                                      | Model pipeline as Skill with agent nodes per ticket type   |
| Support Widget UI        | SalesChatWidget (SSE, sessions, rate limiting, i18n) | Add ticket creation flow, context collection, embed bundle |
| Support Widget backend   | External API public chat routes (domain allowlists)  | Add ticket creation endpoint                               |
| Agent tools              | ToolRegistry                                         | Register board/* tools for agent use                       |
| Email notifications      | Email service + notifications/ project (Brevo)       | Event-driven triggers from board card transitions          |


### Estimated Savings: ~35-50 Weeks of Engineering


| Capability                     | Weeks Saved |
| ------------------------------ | ----------- |
| Auth + JWT + 2FA + SSO         | 3-4         |
| Multi-tenancy + DB isolation   | 2-3         |
| LLM multi-provider integration | 4-6         |
| RAG pipeline                   | 6-8         |
| Context/prompt assembly        | 2-3         |
| Tool-calling orchestrator      | 3-4         |
| DAG execution engine           | 3-4         |
| Queue system + workers         | 2-3         |
| Billing + token tracking       | 2-3         |
| Audit logging                  | 1-2         |
| MCP server infrastructure      | 1-2         |
| Frontend foundation            | 2-3         |
| Chat widget pattern            | 1-2         |
| Email sending                  | 1-2         |
| Deployment pipeline            | 2-3         |
| Monitoring                     | 1-2         |
| **Total**                      | **~35-50**  |


### AIDEN Services Inventory (Available for FORGE)

Full inventory of `packages/core/src/services/` (31 service directories):


| Service                                                                                                                                                   | What It Does                                                                   | FORGE Relevance                           |
| --------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------ | ----------------------------------------- |
| `llm/`                                                                                                                                                    | Multi-provider registry, metered providers, ModelRouter, fallbacks             | **Core**: All agent LLM calls             |
| `rag/`                                                                                                                                                    | Full RAG stack: retrieval, vector, BM25, embeddings, SmartRAGRouter, reranking | **Core**: Support Widget Q&A              |
| `context/`                                                                                                                                                | ContextAssembler, token budget, layered providers                              | **Core**: Agent prompt assembly           |
| `billing/`                                                                                                                                                | Token/cost usage, limits, credits, daily budgets, invoices                     | **Core**: Per-agent cost tracking         |
| `audit/`                                                                                                                                                  | Redis-buffered compliance audit logging                                        | **Core**: Agent action audit trail        |
| `orchestrator/`                                                                                                                                           | Tool-calling loop, streaming, plan cache                                       | **Core**: Agent execution loop            |
| `triage/`                                                                                                                                                 | Cheap model first, escalate to stronger                                        | **Adapt**: Tier-based routing             |
| `feedback/`                                                                                                                                               | LLM analysis of user feedback                                                  | **Useful**: Support ticket enrichment     |
| `health/`                                                                                                                                                 | System health aggregation                                                      | **Direct**: FORGE health monitoring       |
| `monitoring/`                                                                                                                                             | Performance tracker, New Relic events                                          | **Direct**: Agent performance tracking    |
| `email/`                                                                                                                                                  | Outbound email via Resend                                                      | **Direct**: Resolution notifications      |
| `healing/`                                                                                                                                                | Self-healing coordinator                                                       | **Useful**: Agent failure recovery        |
| `decision-layer/`                                                                                                                                         | PDCA plan generation                                                           | **Reference**: Similar to agent fix plans |
| `chat/`                                                                                                                                                   | Redis event stream, pending answers                                            | **Reference**: SSE patterns               |
| `cache/`                                                                                                                                                  | Unified cache admin                                                            | **Direct**: Agent result caching          |
| `documents/`                                                                                                                                              | Document ingestion, parsing, summaries                                         | **Useful**: Attachment processing         |
| `content/`                                                                                                                                                | Content documents, AI text editing                                             | **Reference**: Rich text patterns         |
| `follow-up/`                                                                                                                                              | Suggested questions, follow-up extraction                                      | **Useful**: Support Widget suggestions    |
| `research/`                                                                                                                                               | Research doc generation                                                        | **Useful**: Feature Agent research        |
| `prompts/`                                                                                                                                                | Global prompt registry                                                         | **Direct**: Agent prompt management       |
| Others (agent-builder, cleanup, debug, experts, export, image, notifications, ocr, presentation-templates, skills, slide-designer, templates, tts, voice) | Various AIDEN features                                                         | Not directly used by FORGE initially      |


Full inventory of `packages/core/src/modules/` (17 modules):


| Module                                                                                   | What It Does                                                  | FORGE Relevance                     |
| ---------------------------------------------------------------------------------------- | ------------------------------------------------------------- | ----------------------------------- |
| `queue/`                                                                                 | BullMQ: flow, processing, skill, admin queues + workers       | **Core**: Async agent scheduling    |
| `engine/`                                                                                | DAG ExecutionEngine, ExecutionStore, node I/O                 | **Core**: Pipeline execution        |
| `tools/`                                                                                 | ToolRegistry, ToolExecutor, MCPClient, progressive disclosure | **Core**: Agent MCP tool access     |
| `skills/`                                                                                | Tenant skills (workflows), schemas, executions                | **Adapt**: Pipeline as Skill        |
| `nodes/`                                                                                 | Workflow node registry + executors (LLM, RAG, HTTP, etc.)     | **Adapt**: Agent step nodes         |
| `triggers/`                                                                              | Manual/webhook/cron triggers                                  | **Adapt**: Board event triggers     |
| `mcp/`                                                                                   | MCP server configs + tool discovery cache                     | **Direct**: MCP config management   |
| `audit/`                                                                                 | Global audit log with structured diffs                        | **Direct**: Agent audit             |
| `api-keys/`                                                                              | Provider API keys, model resolver, pricing                    | **Direct**: Model config            |
| `credentials/`                                                                           | Encrypted tenant/user credentials                             | **Useful**: Integration credentials |
| Others (agent-installations, agent-templates, installations, microapps, models, prompts) | AIDEN-specific                                                | Not directly used initially         |


---

## 6. Component Catalog

Each component described independently. Use as a reference when deep-diving into detailed specifications.

---

### Component A: Ticket Contracts + Guidance Templates

**What it is**: The shared schema package defining all ticket data structures and the prompt-engineered guidance templates that tell agents what to do at each pipeline stage.

**Why it matters**: This is the API specification of the entire framework. Every module validates against these schemas. If the contracts are right, modules are truly interchangeable. The guidance templates are prompt engineering at a system level.

**What's new to build**:

- Zod schemas: HumanLayer, AgentLayer, IntakeContract, AnalysisContract, ExecutionContract, DeliveryContract, HandoffNote, AgentGuidance, ROIBreakdown, AgentAttachment, RelatedTicket
- ~12 guidance templates (one per tier transition, see Section 4)
- TypeScript types inferred from Zod
- Contract versioning (`contractVersion` field, semver)
- Contract validation helpers

**AIDEN reuse**: Zod patterns from `packages/validation/`, type patterns from `packages/types/`

**Code location in forge_api**:

- `packages/validation/src/forge/` -- Zod schemas
- `packages/types/src/forge/` -- TypeScript types
- `packages/core/src/modules/forge-contracts/templates/` -- guidance templates

**Dependencies**: None (this is the foundation everything else depends on)

**Feature tiers**:

- *Basic*: Core schemas (base ticket, intake, analysis), 4 guidance templates (bug and feature pipelines)
- *Advanced*: All sub-contracts (execution, delivery), all 12 templates, `contextPriority` for ContextAssembler integration, `extensions` field for custom module data
- *Professional*: Contract diffing/migration tools, template A/B testing (track which templates produce best agent output), per-tenant template overrides

**Detailed specification**: [TODO: `docs/specs/component-a-contracts.md`]

---

### Component B: ROI Scoring Engine

**What it is**: Shared library for calculating ROI scores on features and bugs, producing a normalized 0-100 score with a transparent breakdown.

**Why it matters**: ROI scoring is what makes the board a prioritization tool, not just a task list. Every analysis agent uses this. The transparency of the breakdown builds trust -- humans can see exactly why a ticket got its score.

**What's new to build**:

- RICE scorer: `(Reach * Impact * Confidence) / Effort` with adjustments for strategic alignment and request frequency
- Bug ROI scorer: `(Severity * Frequency * UserImpact) / FixEffort` with fix-tier awareness
- Normalization to 0-100 scale
- Breakdown output showing every factor with its value and reasoning

**ROI Formulas**:

```
Feature ROI = (Reach * Impact * Confidence) / Effort
  Reach:      Users/month affected (1-10)
  Impact:     Needle movement per user (0.25x-3x)
  Confidence: Certainty about reach and impact (0-100%)
  Effort:     Person-weeks (0.5-10+)
  Adjustments: +strategic alignment (0-2), +request frequency, -tech debt penalty

Bug ROI = (Severity * Frequency * UserImpact) / FixEffort
  Severity:   P0=10, P1=7, P2=3, P3=1
  Frequency:  daily=10, weekly=5, monthly=2, rare=1
  UserImpact: % users affected * workflow disruption (1-10)
  FixEffort:  Minimal fix hours (NOT refactoring hours)

  Fix tier default: ALWAYS hotfix unless ROI of proper fix is 2x+ the hotfix ROI
```

**AIDEN reuse**: Billing service for cost-aware effort estimation

**Code location in forge_api**: `packages/core/src/services/forge-roi/`

**Dependencies**: Component A (uses ROIBreakdown type from contracts)

**Feature tiers**:

- *Basic*: RICE and Bug ROI formulas, manual factor input, 0-100 normalization with breakdown
- *Advanced*: Historical calibration (compare predicted vs. actual ROI of past tickets), effort estimation from codebase analysis, request frequency auto-count from similar tickets
- *Professional*: Portfolio-level optimization (given budget X, which ticket combination maximizes total ROI), trend analysis, confidence intervals

**Detailed specification**: [TODO: `docs/specs/component-b-roi.md`]

---

### Component C: Kanban Board (API + MCP + UI)

**What it is**: The central protocol layer. REST API for CRUD on boards/lists/cards, MCP server for agent access, and a React kanban UI for human oversight.

**Why it matters**: The nervous system of the entire framework. All modules communicate through it. The Board API + MCP server defines the contract boundary that makes modules interchangeable.

**What's new to build**:

- **Models**: Board, List, Card (storing ForgeTicket with humanLayer + agentLayer), CardComment, CardHistory
- **Service**: BoardService, CardService, CardTransitionService (validates moves, dispatches webhooks), WebhookService
- **API routes**: CRUD boards/lists/cards, card transitions, search, webhook dispatch, pipeline metrics
- **MCP server tools**: `board/list_all`, `board/get_card`, `board/create_card`, `board/update_card`, `board/move_card`, `board/add_comment`, `board/add_attachment`, `board/search`, `board/get_metrics`
- **React UI**: Drag-and-drop kanban columns, card rendering (humanLayer only: title, type badge, priority, ROI score, pipeline stage), expandable card detail (full agentLayer + handoff chain + agent activity log), "action required" flags, ROI distribution chart

**AIDEN reuse**:

- Mongoose connection + schema patterns from `packages/schemas/`
- Auth middleware (tenant-scoped board access)
- BullMQ for async webhook dispatch
- Express.js route patterns from existing controllers
- MCP server pattern from `forge_mcp_servers/src/index.ts`
- Frontend: React + Tailwind + Zustand from forge_frontend

**Code location**:

- Models: `forge_api/packages/schemas/src/forge/`
- Service: `forge_api/packages/core/src/services/forge-board/`
- Module: `forge_api/packages/core/src/modules/forge-board/`
- API: `forge_api/src/tenant/controllers/forge-board.controller.ts`
- External API: `forge_external_api/src/tenant/routes/forge-board.routes.ts`
- MCP: `forge_mcp_servers/src/servers/kanban.ts`
- Frontend: `forge_frontend/src/pages/forge/Board.tsx`, `CardDetail.tsx`, `Pipeline.tsx`

**Dependencies**: Component A (ticket contract schemas)

**Feature tiers**:

- *Basic*: Single board per project, fixed lists (Backlog → Todo → In Progress → Review → Done), card CRUD, drag-and-drop, MCP server with core tools, webhooks on transitions
- *Advanced*: Custom lists, WIP limits, card dependencies (blocks/blocked-by), SLA timers (time-in-status), bulk operations, search with filters (by type, priority, ROI range, pipeline stage, age), card templates per type, per-list age tracking
- *Professional*: Multiple boards per project, custom workflows per ticket type, swimlanes, throughput/velocity/cycle-time analytics, approval gates, role-based MCP permissions

**Detailed specification**: [TODO: `docs/specs/component-c-board.md`]

---

### Component D: Agent Runner + Pipeline Service

**What it is**: The coordination layer that runs agents and manages ticket flow through the pipeline. Integrates AIDEN's Orchestrator for execution, ContextAssembler for prompt building, and ExecutionEngine for pipeline DAGs.

**Why it matters**: This turns individual agents into a coherent pipeline. It decides which agent runs next, with which model, and handles errors, timeouts, and cost tracking.

**What's new to build**:

- **AgentRunner**: Wraps AIDEN's Orchestrator with ticket context injection (`TicketContextProvider`, `TicketGuidanceProvider` for ContextAssembler). Handles agent lifecycle: read ticket → build prompt → execute with tools → validate output → write back to ticket.
- **TierRouter**: Maps agent ID → model config, using Triage patterns for tier-based model selection. Handles fallbacks.
- **PipelineService**: Defines per-ticket-type DAGs (bug pipeline, feature pipeline, simple task pipeline). Triggers next agent when current step completes. Handles timeouts and error flagging.
- **Pipeline Skill definitions**: Model each pipeline as an AIDEN Skill where each node = an agent step.

**Integration with AIDEN ContextAssembler**:

```
Agent prompt = ContextAssembler.assemble({
  providers: [
    SystemInstructionsProvider(guidance.role),
    TicketContextProvider(ticket.agentLayer.context),
    TicketGuidanceProvider(ticket.agentLayer.guidance),
    ToolDefinitionsProvider(agentTools),
    RAGContextProvider(ragQuery),  // optional
  ],
  tokenBudget: modelConfig.maxTokenBudget,
  // TokenBudgetAllocator respects guidance.contextPriority
});
```

**Pipeline DAGs per ticket type**:

```
Bug:      Classify → Collect Evidence → Analyze Bug → Map Code → Implement → Notify
Feature:  Classify → Collect Context → Analyze Feature → Map Code → Implement → Notify
Simple:   Classify → (skip analysis) → Map Code → Implement → Notify
Question: Classify → RAG Answer → (done, no ticket)
```

**AIDEN reuse**: Orchestrator, ContextAssembler, TriageService, ExecutionEngine, BullMQ, ToolRegistry

**Code location in forge_api**: `packages/core/src/services/forge-agents/`

**Dependencies**: Components A (contracts), C (board API)

**Feature tiers**:

- *Basic*: Linear pipeline per ticket type, tier-based model routing, timeout per step, error flagging to board
- *Advanced*: Parallel agent steps (e.g., evidence collection + similar ticket search simultaneously), per-ticket cost tracking, confidence-based auto-escalation (confidence < 0.5 → flag for human), re-run capability
- *Professional*: Dynamic pipeline modification (skip steps for simple tickets), agent performance analytics (success rate, avg cost, avg time per agent), self-improving pipelines (track which guidance templates produce best results)

**Detailed specification**: [TODO: `docs/specs/component-d-runner.md`]

---

### Component E: Support Widget

**What it is**: Embeddable chat widget for end-user websites. Answers questions via RAG over product docs, classifies conversations, and creates tickets on the board with full context.

**Why it matters**: Primary intake channel. The quality of context it collects determines how well all downstream agents perform. First touchpoint with users.

**What's new to build**:

- Adapted SalesChatWidget with: ticket creation flow (structured form after classification), context collection (screenshot upload, browser/environment auto-detect), shadow DOM wrapper for style-isolated `<script>` embedding
- Backend: conversation-to-ticket mapping, IntakeContract assembly
- Tier 1 Support Classifier: auto-detect bug/feature/question from conversation
- Tier 2 Support Responder: RAG Q&A over product docs, escalation decision

**AIDEN reuse**:

- `SalesChatWidget.tsx`: SSE streaming, session management, rate limiting, i18n, markdown rendering
- External API public chat routes: domain allowlists, session creation
- `services/rag/` RAGService: Q&A over product knowledge base
- `services/llm/` for classifier and responder LLM calls
- `services/follow-up/` for suggested questions

**Code location**:

- Widget: `forge_frontend/src/components/support-widget/SupportChatWidget.tsx`
- Embed: `forge_frontend/src/components/support-widget/embed.ts`
- Backend: `forge_external_api/src/tenant/routes/forge-support.routes.ts`
- T1: `forge_api/packages/core/src/modules/forge-agents/tier1/support-classifier.ts`
- T2: `forge_api/packages/core/src/modules/forge-agents/tier2/support-responder.ts`

**Dependencies**: Components A (IntakeContract), C (board API for ticket creation), D (agent runner)

**Feature tiers**:

- *Basic*: Chat widget, RAG Q&A over product docs, manual ticket type selection, auto-collected environment info (browser, OS, URL), ticket created on board with full conversation context
- *Advanced*: Auto-classification (T1 Classifier), smart context assembly, screenshot/file upload, suggested articles before ticket creation (deflection), conversation history per user (session persistence)
- *Professional*: Proactive messaging (triggered by page events), satisfaction surveys, analytics dashboard (deflection rate, resolution time, common topics), multi-language, agent handoff protocol (escalate to human email)

**Detailed specification**: [TODO: `docs/specs/component-e-support.md`]

---

### Component F: Bug Agent

**What it is**: Two-tier agent that collects evidence about bugs (T1) and analyzes root cause, generates fix plans, and scores ROI (T2).

**Why it matters**: This is where "every bug fix becomes a refactoring" gets prevented. The economics-first approach ensures minimal fixes are the default. The tiered fix plan (hotfix/proper/refactor) with ROI comparison gives founders clear decision data.

**What's new to build**:

- **Tier 1 Bug Evidence Collector**: Pulls error logs (K8s MCP), error traces (NewRelic MCP), queries similar tickets (Board MCP), counts affected users (MongoDB MCP), structures everything into `context.errorData`
- **Tier 2 Bug Analyst**: Root cause analysis (or top hypotheses with evidence), three-tier fix plan (hotfix/proper fix/refactor with effort + risk for each), Bug ROI scoring, guidance rewrite for coding agent

**Key design constraint**: The Bug Analyst's system prompt explicitly encodes: "You are an economist, not a perfectionist. Default recommendation is ALWAYS the cheapest fix that resolves the user's problem. Only recommend a larger fix if the ROI math proves it."

**AIDEN reuse**: Agent Runner (Component D), K8s/NewRelic/MongoDB MCP tools, Board MCP, ROI Engine (Component B)

**Code location in forge_api**:

- `packages/core/src/modules/forge-agents/tier1/bug-evidence-collector.ts`
- `packages/core/src/modules/forge-agents/tier2/bug-analyst.ts`

**Dependencies**: Components A (AnalysisContract), B (Bug ROI), C (board read/write), D (agent runner)

**Feature tiers**:

- *Basic*: Log extraction, root cause hypothesis, 3 fix tiers with effort estimates, ROI score with breakdown, guidance for coding agent
- *Advanced*: Automated duplicate detection (vector similarity on descriptions), regression risk assessment, historical pattern recognition (recurring bug types), SLA-based urgency, related test coverage analysis
- *Professional*: Automated reproduction attempt (via coding agent), root cause chain (5-whys automation), technical debt accounting (track cumulative cost of workarounds), post-mortem template, trend analysis (bug rate by component over time)

**Detailed specification**: [TODO: `docs/specs/component-f-bug.md`]

---

### Component G: Feature Agent

**What it is**: Two-tier agent that collects context about feature requests (T1) and produces user stories, acceptance criteria, technical specification, and RICE score (T2).

**Why it matters**: Turns vague "we should add X" into scored, spec'd, implementation-ready tickets. Prevents building features that don't move the needle.

**What's new to build**:

- **Tier 1 Feature Context Collector**: Finds similar features (board search), counts request frequency (how many users asked for this), gathers competitive/market references (web search), identifies affected user personas
- **Tier 2 Feature Analyst**: User story generation, acceptance criteria, technical notes, out-of-scope definition, RICE scoring, guidance rewrite for coding agent

**AIDEN reuse**: Agent Runner (Component D), Board MCP, web search tool, ROI Engine (Component B), Research service patterns

**Code location in forge_api**:

- `packages/core/src/modules/forge-agents/tier1/feature-context-collector.ts`
- `packages/core/src/modules/forge-agents/tier2/feature-analyst.ts`

**Dependencies**: Components A (AnalysisContract), B (RICE), C (board), D (agent runner)

**Feature tiers**:

- *Basic*: Context collection, user stories, acceptance criteria, RICE score, guidance for coding agent
- *Advanced*: Competitive analysis, effort estimation from codebase analysis, dependency detection (what existing features this builds on), multi-stakeholder aggregation (combine similar requests)
- *Professional*: PRD (Product Requirements Document) generation, A/B test suggestions, market sizing integration, post-launch impact measurement, feature decomposition into sub-tasks

**Detailed specification**: [TODO: `docs/specs/component-g-feature.md`]

---

### Component H: Coding Bridge

**What it is**: Integration layer that makes execution-ready tickets consumable by any coding agent. Includes a T1 Code Context Mapper and an optional Claude Code wrapper. This is the most "swappable" module.

**Why it matters**: We don't build a coding agent -- we build a bridge. A founder using Cursor, Claude Code, Windsurf, or any IDE with MCP support can consume tickets directly. The bridge handles the translation.

**What's new to build**:

- **Tier 1 Code Context Mapper**: Identifies affected files, maps dependencies, checks test coverage, extracts relevant code snippets, enriches ticket with codebase context
- **Coding Bridge API**: "Next task" endpoint (returns highest-priority execution-ready ticket), result reporting endpoint (accepts PR link, test results, etc.)
- **Optional Claude Code CLI wrapper**: Reads ticket, executes Claude Code in a controlled environment, reports results back
- **Task Dashboard UI**: Active coding tasks, diff viewer, approval/reject buttons, cost per implementation

**AIDEN reuse**: Agent Runner (Component D), GitHub MCP (PR operations), Board MCP (ticket read/write)

**Code location**:

- Agent: `forge_api/packages/core/src/modules/forge-agents/tier1/code-context-mapper.ts`
- API: `forge_api/src/tenant/controllers/forge-coding.controller.ts`
- UI: `forge_frontend/src/pages/forge/CodingTasks.tsx`

**Dependencies**: Components A (ExecutionContract), C (board), D (agent runner)

**Feature tiers**:

- *Basic*: Code context mapping, "next task" API, result reporting API, task dashboard with basic status
- *Advanced*: Claude Code CLI wrapper, parallel tasks via git worktrees, auto-retry on CI failure, per-implementation cost tracking
- *Professional*: Multiple IDE integrations (Cursor MCP config, Windsurf adapter, Aider adapter), PR review agent, architecture-aware task routing (refuse tasks that need human architecture decisions), security scanning integration

**Detailed specification**: [TODO: `docs/specs/component-h-coding.md`]

---

### Component I: Communication Agent

**What it is**: Tier 3 agents that produce human-facing text: resolution notifications, board summaries, and release notes. Closes the loop with users.

**Why it matters**: When a user's bug is fixed or feature is live, they hear about it automatically. This transforms support from "black hole where tickets disappear" into a transparent, responsive process.

**What's new to build**:

- **Tier 3 Resolution Notifier**: Reads full handoff chain, generates personalized email explaining what was fixed/built and how
- **Tier 3 Board Summarizer**: Produces human-readable summaries for the board's humanLayer when agents update tickets
- **Tier 3 Release Notes Generator**: Batches resolved tickets into a user-facing changelog grouped by category
- **Event listeners**: Triggers on board card transitions (card → "Done" or "Deployed")
- **Templates**: Email templates per ticket type (bug resolved, feature live, question answered)

**AIDEN reuse**:

- `services/email/` Resend for transactional email
- Existing `notifications/` project: Brevo integration, release notes pipeline, role-based content, contact sync
- `services/llm/` for text generation
- `modules/queue/` BullMQ for event-driven job scheduling

**Code location in forge_api**:

- `packages/core/src/modules/forge-agents/tier3/resolution-notifier.ts`
- `packages/core/src/modules/forge-agents/tier3/board-summarizer.ts`
- `packages/core/src/modules/forge-agents/tier3/release-notes-generator.ts`

**Dependencies**: Components A (DeliveryContract), C (board events), D (agent runner)

**Feature tiers**:

- *Basic*: Email on ticket resolution, personalized with user name + original issue + resolution summary, unsubscribe link
- *Advanced*: In-app notifications (via support widget notification bell), notification preferences per user (email/in-app/none), batch digest mode (daily/weekly summary), auto-generated release notes from resolved tickets, status page integration
- *Professional*: Multi-channel (email + SMS + webhook), NPS/CSAT surveys after resolution, auto-generated changelog page, segmented communication (by role, plan, activity), communication analytics (open rates, click-through)

**Detailed specification**: [TODO: `docs/specs/component-i-comm.md`]

---

### Component J: DevOps Dashboard

**What it is**: Lightweight React dashboard for deploying, running, and monitoring the application from one UI. Proxies to existing infrastructure APIs. No DevOps jargon -- business language only.

**Why it matters**: Founders need "Deploy latest version" and "Is it healthy?" without touching kubectl or reading Grafana dashboards. All infrastructure concepts hidden behind simple actions.

**What's new to build**:

- **Deploy page**: Trigger GitOps deploy from UI, deployment history, rollback button, environment status
- **Logs page**: Loki log viewer with service/time/severity filters
- **Metrics page**: Prometheus metrics (CPU, memory, request count, error rate) as simple charts
- **Costs page**: Hetzner server costs + LLM costs (from billing) + MongoDB costs, all in one view
- **Environment switcher**: Staging vs. production toggle

**AIDEN reuse**:

- `services/health/` for health aggregation
- `services/monitoring/` for New Relic integration
- `services/billing/` for LLM cost data
- Frontend foundation (React + Tailwind + Zustand)
- Rancher Fleet API for deployment management
- Existing Prometheus/Loki infrastructure

**Code location**:

- UI: `forge_frontend/src/pages/forge/DevOps.tsx`, `Logs.tsx`, `Metrics.tsx`, `Costs.tsx`
- Proxy routes: `forge_api/src/tenant/controllers/forge-infra.controller.ts`

**Dependencies**: Component C (board for pipeline status), existing infrastructure

**Feature tiers**:

- *Basic*: Health status (green/yellow/red), deploy button, log tail viewer, basic CPU/memory metrics, environment switcher
- *Advanced*: Deployment history with rollback, alert management (Prometheus alertmanager), resource scaling controls, CI pipeline viewer (GitHub Actions status), secrets management UI, full cost dashboard
- *Professional*: Canary deployments and traffic splitting, performance profiling (New Relic APM), auto-scaling rules, multi-project support, SLA monitoring and uptime tracking

**Detailed specification**: [TODO: `docs/specs/component-j-devops.md`]

---

### Component K: MCP Server Extensions

**What it is**: New MCP server types added to `forge_mcp_servers` for board access, log queries, and code analysis.

**Why it matters**: MCP servers are how agents interact with infrastructure. The Kanban MCP is the most critical -- it's the interface between all agents and the board.

**What's new to build**:

- **Kanban MCP**: `board/`* tools (create, read, update, move, search, comment, add attachment, metrics)
- **Loki MCP**: `log/`* tools (query by service/time/severity, structured log search, log pattern detection)
- **Code Analysis MCP**: `code/`* tools (file tree, dependency graph, test coverage, lint results, recent changes)

**AIDEN reuse**:

- `forge_mcp_servers/src/index.ts` pattern: Express + @modelcontextprotocol/sdk, bearer auth, session management, idle cleanup
- Existing server types as code templates: `kubernetes.ts`, `github.ts`, `newrelic.ts`, `mongodb.ts`

**Code location in forge_mcp_servers**:

- `src/servers/kanban.ts`
- `src/servers/loki.ts`
- `src/servers/code-analysis.ts`

**Dependencies**: Component C (kanban MCP needs board service), Loki/Prometheus infrastructure

**Feature tiers**:

- *Basic*: Kanban MCP with core CRUD + move + search
- *Advanced*: Loki MCP (log queries), Code Analysis MCP (file/test analysis), enriched board search (by ROI range, pipeline stage, age, assignee)
- *Professional*: Composite queries across MCP servers (e.g., "find the bug ticket related to this error log"), MCP tool versioning with contract safety (mcpdiff), read-only vs. read-write MCP permissions per agent

**Detailed specification**: [TODO: `docs/specs/component-k-mcp.md`]

---

## 7. Project Structure

All FORGE code lives in `forge-`* namespaced directories within the forked AIDEN repos:

```
forge_api/
  packages/
    types/src/forge/                  # Component A: TypeScript types
    schemas/src/forge/                # Component C: Mongoose models
    validation/src/forge/             # Component A: Zod contract schemas
    core/src/
      services/
        forge-board/                  # Component C: Board service
        forge-roi/                    # Component B: ROI engine
        forge-agents/                 # Component D: Agent runner, tier router, pipeline
      modules/
        forge-board/                  # Component C: Board data module
        forge-contracts/              # Component A: Guidance templates
        forge-agents/                 # Components E-I: Agent logic
          tier1/
            support-classifier.ts     # Component E
            bug-evidence-collector.ts # Component F
            feature-context-collector.ts # Component G
            code-context-mapper.ts    # Component H
          tier2/
            support-responder.ts      # Component E
            bug-analyst.ts            # Component F
            feature-analyst.ts        # Component G
          tier3/
            resolution-notifier.ts    # Component I
            board-summarizer.ts       # Component I
            release-notes-generator.ts # Component I
  src/tenant/controllers/
    forge-board.controller.ts         # Component C
    forge-coding.controller.ts        # Component H
    forge-infra.controller.ts         # Component J

forge_external_api/
  src/tenant/routes/
    forge-board.routes.ts             # Component C: External board API
    forge-support.routes.ts           # Component E: Support widget backend

forge_frontend/
  src/
    pages/forge/
      Board.tsx                       # Component C
      CardDetail.tsx                  # Component C
      Pipeline.tsx                    # Component C
      CodingTasks.tsx                 # Component H
      DevOps.tsx                      # Component J
      Logs.tsx                        # Component J
      Metrics.tsx                     # Component J
      Costs.tsx                       # Component J
    components/
      forge/
        KanbanColumn.tsx              # Component C
        TicketCard.tsx                # Component C
        AgentActivityLog.tsx          # Component C
        ROIBadge.tsx                  # Component C
        GuidanceViewer.tsx            # Component C
      support-widget/
        SupportChatWidget.tsx         # Component E
        TicketCreationFlow.tsx        # Component E
        embed.ts                      # Component E

forge_mcp_servers/
  src/servers/
    kanban.ts                         # Component K
    loki.ts                           # Component K
    code-analysis.ts                  # Component K

forge_gitops/
  apps/
    forge-api/                        # Updated from aihub-api
    forge-frontend/                   # Updated from aihub-frontend
    forge-external-api/               # Updated from aihub-external-api
    forge-mcp-servers/                # New or updated deployment
```

---

## 8. Build Phases

### Phase 0: Repository Setup (~1 day)

- Create `rootsandrivets` GitHub organization
- Copy all 5 repos from `papa-bear-ventures`, rename to `forge_*`
- Update package names, Docker image tags, Fleet configs, CI workflows
- Update environment variables and secrets
- Verify independent build and deploy

### Phase 1: The Protocol + Board (3-4 weeks)

**Components**: A (contracts), B (ROI engine), C (board), K (kanban MCP only)

**Result**: Tickets can be created on the board, viewed by humans, accessed by agents via MCP. ROI scores can be calculated. The contract schemas define all module boundaries.

**Milestone**: Create a ticket via MCP, view it on the board, move it between lists, see ROI badge.

### Phase 2: Intake + Analysis (4-5 weeks)

**Components**: D (agent runner), E (support widget), F (bug agent), G (feature agent)

**Result**: Users can report issues via the embedded widget. Agents automatically collect evidence, analyze, score by ROI, and produce fix plans or feature specs.

**Milestone**: User reports a bug via widget → T1 collects evidence → T2 analyzes and scores → human sees a prioritized, analyzed ticket on the board with fix plan attached.

### Phase 3: Execution + Delivery (4-5 weeks)

**Components**: H (coding bridge), I (communication agent), J (DevOps dashboard), K (loki + code-analysis MCP)

**Result**: Full pipeline end-to-end. Coding agents can pick up tickets, implement, deploy, and users get notified.

**Milestone**: Bug ticket flows from report → analysis → implementation → deploy → user notification, with the board showing every step.

### Phase 4: Polish + Dogfood (2-3 weeks)

- Pipeline Skill definitions per ticket type
- End-to-end flow testing with synthetic tickets
- Integration guides ("Bring Your Own Coding Agent", "Bring Your Own Support Tool")
- Deploy FORGE for AIDEN as first real customer
- Iterate guidance templates based on real agent behavior

---

## 9. Technology Stack


| Concern       | Choice                               | Rationale                                         |
| ------------- | ------------------------------------ | ------------------------------------------------- |
| Language      | TypeScript (full stack)              | Matches AIDEN, shared contract types              |
| API Framework | Express.js                           | Matches AIDEN                                     |
| Frontend      | React 18 + TailwindCSS + Zustand     | Matches AIDEN frontend                            |
| Database      | MongoDB (Mongoose)                   | Document model fits nested tickets, matches AIDEN |
| Queue         | BullMQ + Redis                       | Async agent work, matches AIDEN                   |
| Vector Search | MongoDB Atlas Vector Search          | Similar ticket search, RAG, matches AIDEN         |
| LLM Access    | AIDEN's LLM service (multi-provider) | Single gateway with billing, already built        |
| Validation    | Zod                                  | Contracts as code, TypeScript inference           |
| MCP Protocol  | `@modelcontextprotocol/sdk`          | Proven pattern from aihub_mcp_servers             |
| Deployment    | Rancher Fleet / Kubernetes (RKE2)    | Existing GitOps pipeline                          |
| CI/CD         | GitHub Actions                       | Existing pipeline                                 |
| Hosting       | Hetzner                              | Existing infrastructure                           |
| Monitoring    | Prometheus + Loki + New Relic        | Existing stack                                    |
| Email         | Brevo (transactional) + Resend       | Existing integrations                             |


---

## 10. Critical Risks and Mitigations

### Architecture Risks

**1. Guidance template quality is everything.**
The ~12 guidance templates are prompt engineering at a system level. Poorly written guidance means bad agent output regardless of model quality. **Mitigation**: Treat templates as code -- version them, test them (feed sample tickets through and inspect output), review them regularly. Budget significant time for empirical iteration.

**2. Tier 1 garbage = Tier 2 garbage.**
If the Evidence Collector extracts wrong log entries, the Bug Analyst reasons about nonsense with high confidence. **Mitigation**: Strong validation on T1 output (check timestamp alignment, similarity thresholds > 0.7 for related tickets). T1 is cheap to rerun -- build a "re-gather" button for humans.

**3. Contract evolution.**
Ticket contracts WILL change as we learn what downstream modules actually need. **Mitigation**: Semver on the contracts package. Backward-compatible additions (new optional fields) as minor versions. Breaking changes (required fields) as major versions. `contractVersion` field on every ticket.

**4. Agent coordination races.**
Two agents updating the same ticket simultaneously. **Mitigation**: Optimistic concurrency (`updatedAt` check on writes, reject stale updates). Agents retry with fresh data.

**5. Pipeline stalls.**
If an agent step times out or errors, tickets pile up. **Mitigation**: Per-stage timeouts. If a step doesn't complete within N minutes, flag ticket with `humanActionRequired`. Per-list age tracking surfaces bottlenecks.

### Operational Risks

**6. Cost explosion.**
Each agent call = LLM API costs. A bug through the full pipeline: ~$0.50-$2.00. At 1000 tickets/month: $500-2000/month. **Mitigation**: Per-agent daily budgets via AIDEN billing. Visible per-ticket cost in handoff notes. Cost dashboard in DevOps page.

**7. Human-in-the-loop fatigue.**
If every ticket flags "action required", founders drown. **Mitigation**: Default to autonomous operation. Only flag for human when: confidence < 0.5, or action is high-risk (deploy to production, send user-facing email). Show "needs attention" count, not a wall of approvals.

**8. Over-tiering simple tasks.**
"Update footer copyright year" doesn't need evidence collection and analysis. **Mitigation**: The Support Classifier should route "simple task" tickets directly to "Ready for Implementation" with minimal guidance, skipping the analysis pipeline.

### Business Risks

**9. Scope management.**
This is 11 components. The modular architecture helps (ship independently), but you need the board + at least 2 agents before the value proposition is clear. **Mitigation**: Phase 1+2 delivers a complete intake-to-analysis loop. Ship and validate before Phase 3.

**10. Target user needs extreme simplicity.**
"Tech-savvy founders" means they understand APIs but not kubectl. Every infrastructure concept must be hidden behind business language. **Mitigation**: The board says "Deploy latest version" not "Rollout restart deployment." The dashboard says "Server healthy" not "Pod CrashLoopBackOff."

### Technical Risks

**11. Codebase divergence.**
FORGE and AIDEN codebases will drift over time. Since these are clean copies (not forks), syncing requires explicit effort. **Mitigation**: FORGE code lives in `forge-`* namespaced directories that don't conflict with AIDEN changes, making upstream merges clean. Add the `upstream` remote to each repo. Establish a regular sync cadence (e.g., monthly `git fetch upstream && git merge upstream/main`). For pushing features back to AIDEN, cherry-pick specific commits into a local AIDEN clone and PR to `papa-bear-ventures`.

**12. Human override always wins.**
If a human changes priority or ROI score, agents must not revert it. **Mitigation**: `overriddenByHuman` flag on the ticket. All agent writes check this flag before modifying human-set values. Constraints in all guidance templates: "If a human has overridden the priority, do NOT change it back."

**13. Graceful degradation.**
If the LLM API is down, the board, dashboard, and widget (minus AI answers) must still work. **Mitigation**: Agents are enhancements, not dependencies for the core workflow. The board is a standalone CRUD app. LLM features degrade gracefully with user-visible status.

**14. Rate limiting on public surfaces.**
The support widget is public-facing. **Mitigation**: Per-IP and per-session rate limits from day one. LLM calls via the widget get a separate, lower budget than internal agent calls.

**15. Dogfood from day one.**
Deploy FORGE for AIDEN as the first real customer. Route AIDEN bugs through FORGE's bug agent. This gives real data, real edge cases, and proves the system works.

---

## 11. Open Source Evaluation

For each component, we evaluated existing OSS solutions. The standard: adopt only if ~100% fit; otherwise build to make it truly fit our needs.

### Kanban Board -- BUILD


| Candidate  | Stars  | Stack      | Verdict                                                                                 |
| ---------- | ------ | ---------- | --------------------------------------------------------------------------------------- |
| Planka     | 11.7k  | JS/React   | Fair Use License (not truly OSS), no custom fields, no MCP, no agent-optimized payloads |
| Huly       | 25k    | TS/Svelte  | Full PM suite (overkill), heavy (16GB RAM), EPL-2.0                                     |
| Multiboard | small  | TS/Next.js | Too generic, no contract awareness                                                      |
| Flux       | small  | TS         | MCP-native (good inspiration), but CLI-only, no web UI, no custom schemas               |
| Kan.bn     | small  | unknown    | No API depth                                                                            |
| Leantime   | medium | PHP        | Wrong stack (PHP), AGPL                                                                 |


**Decision**: Build. The board must natively support dual-layer agent-optimized tickets, custom fields (ROI, guidance, handoff notes), and MCP. No OSS board comes close. Flux's MCP integration pattern is worth studying.

### Support Widget -- BUILD


| Candidate | Stars | Stack    | Verdict                                                                                  |
| --------- | ----- | -------- | ---------------------------------------------------------------------------------------- |
| Chatwoot  | 28k   | Ruby/Vue | Wrong stack, massive (full Intercom alternative), "Captain" AI not deeply RAG-integrated |
| SiteChat  | small | TS       | RAG + widget, but lacks ticket contract integration                                      |
| Ask0      | small | TS/PG    | RAG widget, no ticketing                                                                 |
| Persona   | small | TS       | Great zero-dep widget pattern (inspiration), but not a full support system               |


**Decision**: Build, inspired by Persona's zero-dep approach. We already have the hard parts (LLM + RAG via AIDEN). We just need a thin widget that connects to our board.

### Feature/Bug Agents -- BUILD

No OSS tool combines conversational analysis + ROI scoring + tiered fix plans + ticket contract output. The open-source "Prioritize" tool (RICE + Claude) is a useful reference for scoring UI.

### Coding Agent -- INTEGRATE, DON'T BUILD

Claude Code, Cursor, Devin invest hundreds of millions into coding agents. We build an integration bridge, not a coding agent.

### CI/CD Dashboard -- BUILD (lightweight)


| Candidate   | Stars  | Stack    | Verdict                                                      |
| ----------- | ------ | -------- | ------------------------------------------------------------ |
| Gantry      | new    | Go       | 50% fit -- wrong stack, wrong GitOps tool (ArgoCD not Fleet) |
| Kuberise.io | medium | Mixed    | Overkill (40+ tools), too opinionated                        |
| OpenChoreo  | new    | Mixed    | Full IDP, way too heavy                                      |
| Backstage   | 29k    | TS/React | Designed for 500-person orgs, enormous learning curve        |


**Decision**: Build lightweight. We already have all the infrastructure APIs. Just need a simple React dashboard proxying to them.

### Communication Agent -- EXTEND EXISTING

Extend the existing `notifications/` project with event-driven triggers from the board.

### MCP Servers -- EXTEND EXISTING

Extend the existing `aihub_mcp_servers` with new server types. Same pattern, same codebase.

---

## 12. Component Specification Index

Detailed specifications for each component will be developed during implementation. This section tracks the specification documents.


| Component                | Spec Document                         | Status |
| ------------------------ | ------------------------------------- | ------ |
| A: Ticket Contracts      | `docs/specs/component-a-contracts.md` | TODO   |
| B: ROI Scoring Engine    | `docs/specs/component-b-roi.md`       | TODO   |
| C: Kanban Board          | `docs/specs/component-c-board.md`     | TODO   |
| D: Agent Runner          | `docs/specs/component-d-runner.md`    | TODO   |
| E: Support Widget        | `docs/specs/component-e-support.md`   | TODO   |
| F: Bug Agent             | `docs/specs/component-f-bug.md`       | TODO   |
| G: Feature Agent         | `docs/specs/component-g-feature.md`   | TODO   |
| H: Coding Bridge         | `docs/specs/component-h-coding.md`    | TODO   |
| I: Communication Agent   | `docs/specs/component-i-comm.md`      | TODO   |
| J: DevOps Dashboard      | `docs/specs/component-j-devops.md`    | TODO   |
| K: MCP Server Extensions | `docs/specs/component-k-mcp.md`       | TODO   |


---

*This document is the single source of truth for FORGE architecture. Update it as decisions evolve. Each component spec doc (when created) will reference this document for context.*