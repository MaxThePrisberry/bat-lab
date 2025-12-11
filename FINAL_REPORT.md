# Plaibook AI Outbound Agent - Final Report

**CS 452 Final Project**
**Author:** Maxwell Prisbrey

---

## Project Summary

Plaibook is an AI-powered platform that automates outbound customer engagement through SMS campaigns while giving sales managers real-time oversight through a "Cockpit" interface. The system also ingests call recordings from various softphones (Genesys, Five9, RingCentral, GoHighLevel, CallRail), runs them through AI analysis, and surfaces coaching insights to help teams close more deals. Think of it as giving every sales team an AI co-pilot that handles the grunt work while humans focus on the conversations that actually matter.

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CLIENT LAYER                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                    React 19 + TypeScript + Vite                      │    │
│  │  Cockpit │ Campaigns │ Analytics │ Coaching │ Integrations Settings │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                    │ tRPC (type-safe)          │ WebSocket                  │
└────────────────────┼───────────────────────────┼────────────────────────────┘
                     ▼                           ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              API LAYER (ECS Fargate)                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │  tRPC Router │  │  Socket.IO   │  │   BullMQ     │  │  Schedulers  │     │
│  │  (40+ procs) │  │  (Redis pub) │  │   Queues     │  │  (Cron jobs) │     │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘     │
└─────────────────────────────────────────────────────────────────────────────┘
         │                    │                   │
         ▼                    ▼                   ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────────────┐
│    MongoDB      │  │     Redis       │  │        AI Services              │
│  (18 collections│  │  (cache, queue, │  │  ┌───────────┐  ┌───────────┐  │
│   replica set)  │  │   socket.io)    │  │  │CSR Agent  │  │ Overseer  │  │
└─────────────────┘  └─────────────────┘  │  └───────────┘  └───────────┘  │
                                          │         OpenRouter API          │
                                          └─────────────────────────────────┘
                                                         │
┌─────────────────────────────────────────────────────────────────────────────┐
│                           AWS LAMBDA (Call Processing)                       │
│  S3 (recordings) → SQS → Lambda → FFmpeg → AI Analysis → MongoDB            │
│                          (0 → 100+ concurrent, auto-scaling)                 │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                              INTEGRATIONS                                    │
│  Genesys │ Five9 │ RingCentral │ GoHighLevel │ CallRail │ FieldRoutes │ Slack│
└─────────────────────────────────────────────────────────────────────────────┘
```

### Data Model Overview

The system uses MongoDB with 18+ collections organized around these core entities:

- **Organizations** - Multi-tenant isolation, integration credentials, settings
- **Users** - Authentication, roles, organization membership
- **Campaigns** - SMS campaign configuration, playbooks, funnel stages, business hours
- **Leads** - Customer records with campaign history, contact info
- **Conversations** - SMS message history, turn-by-turn AI analysis, billing tracking
- **Calls** - Call recordings, transcripts, AI-generated analysis (objections, outcomes, coaching)
- **Playbooks** - AI conversation scripts with versioning
- **Tasks** - Follow-up scheduling and calendar management

---

## What I Learned

> **[YOUR INSIGHTS - Write 2-3 sentences for each section below]**

### 1. Integration Architecture

_[Write about your experience working with multiple external APIs - what was challenging? What patterns helped?]_

```
YOUR NOTES HERE:




```

### 2. AI/LLM Data Structuring

_[Write about working with LLMs for conversation analysis, call processing, or chatbot features]_

```
YOUR NOTES HERE:




```

### 3. Database Design & NoSQL Patterns

_[Write about MongoDB schema design, atomic operations, or working with document databases]_

```
YOUR NOTES HERE:




```

---

## Key Learnings (Technical)

### 1. Integration Architecture is Where the Real Complexity Lives

Building the core product features was actually the straightforward part. The real challenge was pulling data from half a dozen different platforms (Genesys, Five9, RingCentral, GoHighLevel, CallRail, FieldRoutes) that all have different APIs, auth flows, rate limits, and data formats. The codebase uses a "touchpoint model" where the core system is completely agnostic to specific integrations - it just calls generic `IIntegrationProvider` interfaces and each provider implements the specifics. This meant adding new integrations (like GoHighLevel or CallRail) followed an established pattern.

### 2. LLM Data Synthesis Requires Structured Thinking

Getting useful insights out of LLMs isn't just about writing good prompts - it's about structuring your data so the model can actually reason about it. For call analysis, the schema captures conversation stages, sentiment shifts, objection handling (with hierarchical categories), and outcome classification in a way that's both machine-parseable and human-readable. The chatbot feature that converts natural language to MongoDB aggregation pipelines demonstrates this well: the LLM needs to understand both the user's intent AND the data model to generate valid queries.

### 3. Separation of Storage Concerns: S3 for Blobs, MongoDB for Everything Else

Call recordings don't fit in the database. Recordings land in S3, which triggers SQS, which invokes Lambda, which processes with FFmpeg, sends to AI for analysis, and writes results back to MongoDB. The recording metadata (duration, size, transcription, AI analysis) lives in Mongo while the actual audio stays in S3. This separation means you can query and filter calls without touching blob storage, and Lambda can scale to 100+ concurrent executions without database connection pool exhaustion.

### 4. Real-time Systems Need Redis

Socket.IO works great on a single server. The moment you scale to multiple ECS instances, everything breaks unless you have Redis coordinating between them. The Redis adapter for Socket.IO handles cross-instance pub/sub so that when a conversation updates on server A, clients connected to server B still get the message. BullMQ uses Redis for job queues, which means jobs persist across deploys and server restarts.

### 5. Type Safety Across the Stack Saves Hours of Debugging

Using tRPC with Zod schemas means the TypeScript compiler catches API contract violations before they become runtime bugs. When you change an API response shape, the frontend refuses to compile until you fix all the call sites. Combined with shared types in `@plaibook/shared`, this prevents an entire category of "why is this undefined?" debugging sessions.

---

## AI Integration

AI is deeply woven into the product at multiple levels:

### CSR Agent (SMS Conversations)
Handles customer conversations following configurable "playbooks." The agent receives:
- Conversation history
- Customer context from integrations (like service history from FieldRoutes)
- Available tools (appointment booking, etc.)
- Organization-specific configuration (model, temperature, persona)

Generates contextually appropriate responses with fallback to secondary models (Deepseek) on timeouts.

### Overseer Agent (Conversation Analysis)
A meta-level AI that reviews each conversation turn for:
- Conversation stage classification
- Customer sentiment tracking
- Escalation need detection
- Playbook adherence scoring
- Summary generation

The Overseer's analysis is stored as an append-only log, enabling conversation replay and historical analysis.

### Call Analysis Pipeline
Lambda functions process call recordings through:
1. FFmpeg (audio format conversion)
2. AI transcription and comprehensive analysis
3. Extraction of: conversation stages, objections raised (with hierarchical categorization), sales outcomes, coaching recommendations

Uses Gemini 2.5 Flash for cost-effective processing at scale.

### Natural Language Analytics Chatbot
Managers can ask questions like "show me calls from last week where the customer mentioned price concerns" and the system:
1. Converts natural language to MongoDB aggregation pipelines
2. Executes queries against the call database
3. Generates visualizations and summaries
4. Caches results to avoid re-running expensive LLM calls

Uses Claude Sonnet 4.5 for reliable query generation.

---

## How AI Assisted in Building This Project

Claude (via Claude Code) was instrumental throughout development:

- **Architecture decisions**: Validated the integration provider pattern and touchpoint model approach
- **Code generation**: Generated boilerplate for new tRPC procedures, Mongoose models, and React components
- **Debugging**: Traced issues through the Lambda → SQS → MongoDB pipeline when calls weren't processing
- **Refactoring**: Assisted with feature-based code organization and type safety improvements
- **Documentation**: Generated technical documentation including CLAUDE.md

> **[YOUR EXPERIENCE - Write 2-3 sentences about how YOU used AI assistance]**

```
YOUR NOTES HERE:




```

---

## Why This Project Interests Me

> **[WRITE YOUR PERSONAL MOTIVATION HERE - 1-2 paragraphs about why this project matters to you]**

```
YOUR NOTES HERE:




```

---

## Technical Deep-Dive

### Scaling Characteristics

| Component | Strategy | Notes |
|-----------|----------|-------|
| API (ECS Fargate) | Horizontal scaling (2-10 instances) | ALB distributes traffic, Redis coordinates state |
| Lambda (Call Processing) | Instant scaling (0 → 50+ concurrent) | Perfect for bursty call processing workloads |
| MongoDB | Atlas replica set with read replicas | Writes go to primary, analytics queries hit secondaries |
| Redis (ElastiCache) | Multi-AZ cluster | Handles Socket.IO pub/sub, BullMQ queues, session cache |
| Socket.IO | Redis adapter | Cross-instance real-time communication |

### Concurrency Handling

- **Atomic MongoDB operations**: All updates use `$set`, `$push`, `$inc` operators with `$elemMatch` for array queries - no read-modify-write cycles
- **Queue isolation**: Separate BullMQ queues for different job types prevent one slow operation from blocking others
- **Optimistic patterns**: Turn analysis uses append-only writes, conversation updates are idempotent
- **Rate limiting**: Express middleware with configurable limits per endpoint
- **Connection pooling**: Lambda uses shared MongoDB connections with proper maxPoolSize settings

### Failover Strategy

- **ECS multi-AZ**: Instances run across availability zones (2 AZs), ALB health checks route around failures
- **Redis automatic failover**: ElastiCache with `automatic_failover_enabled` and multi-AZ
- **MongoDB replica set**: Automatic primary election if a node fails
- **Dead letter queues**: Failed SQS messages go to DLQ for inspection (14-day retention, 3 retry attempts)
- **Graceful shutdown**: SIGTERM handlers close connections cleanly, drain in-flight requests

### Authentication & Security

- **JWT tokens**: 1-hour access tokens with refresh token rotation
- **Helmet.js**: Security headers (HSTS, CSP, X-Frame-Options)
- **CORS**: Origin validation against allowed frontend URLs
- **Secrets management**: AWS SSM Parameter Store (SecureString) for all production credentials
- **Input validation**: Zod schemas on every tRPC procedure
- **Brute force protection**: Failed login tracking and lockout
- **Webhook authentication**: HMAC signature verification for inbound webhooks

---

## Links

- **GitHub Repo:** [INSERT LINK]
- **Class Channel Post:** [INSERT TEAMS LINK]

---

## Sharing Permission

**No, don't share**

---

## Technology Stack Summary

| Layer | Technology |
|-------|------------|
| Frontend | React 19, Vite, TypeScript, Tailwind CSS, Zustand, TanStack Query/Table |
| API | Node.js, Express, tRPC, Socket.IO, BullMQ |
| Database | MongoDB (Mongoose ODM) |
| Cache/Queue | Redis (ElastiCache), BullMQ |
| AI | OpenRouter (Claude, GPT-4, Gemini, Deepseek) |
| Infrastructure | AWS ECS Fargate, Lambda, S3, SQS, ALB, ElastiCache |
| IaC | Terraform |
| Monitoring | Datadog APM, CloudWatch |
| Build | Turborepo monorepo |
| Testing | Jest (API), Vitest (Web) |

