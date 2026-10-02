# KT Agent — System Architecture & Design Specification

An intelligent, Slack-native Knowledge Transfer (KT) Agent built with Node.js, TypeScript, LangGraph, and PostgreSQL (pgvector) to automate employee onboarding and project knowledge retrieval.

---

## 1. Executive Summary

### 1.1 Overview
KT Agent is an AI assistant that detects when a new employee joins a Slack team, identifies their assigned team and project repository, requests Team Lead approval, ingests project source code and structure, generates personalized Knowledge Transfer documentation, and provides grounded Q&A grounded in repository code.

### 1.2 Core Architectural Principle
Instead of building a single monolithic AI agent, the system uses **specialized workflows** orchestrated via **LangGraph StateGraphs**. Nodes read and update explicit shared state, enabling deterministic transitions, auditability, and team-based modular development.

```
                            KT Agent
                               │
   ┌───────────────┬───────────┼───────────┬───────────────┬──────────────┐
   ▼               ▼           ▼           ▼               ▼              ▼
Onboarding     Approval   Repository   Knowledge          KT          Question
  Agent        Workflow    Analysis     Builder        Generator     Answering
                            Agent                                      Agent
```

---

## 2. End-to-End System Architecture

### 2.1 High-Level Architecture Diagram

```
                         ┌─────────────────────┐
                         │    COMPANY SLACK    │
                         │                     │
                         │ New Employee joins  │
                         │ Messages / DMs      │
                         └──────────┬──────────┘
                                    │ Slack Events
                                    ▼
                    ┌───────────────────────────┐
                    │      NODE.JS SERVER       │
                    │                           │
                    │ Slack Event Handler       │
                    │ REST APIs / Auth          │
                    │ Middleware                │
                    └─────────────┬─────────────┘
                                  │
                 ┌────────────────┴────────────────┐
                 │                                 │
                 ▼                                 ▼
       ┌──────────────────┐             ┌──────────────────┐
       │ Onboarding       │             │ Question/Answer  │
       │ Workflow         │             │ Workflow         │
       └────────┬─────────┘             └─────────┬────────┘
                │                                 │
                └──────────────┬──────────────────┘
                               ▼
                     ┌─────────────────────┐
                     │      LANGGRAPH      │
                     │  (StateGraph Engine)│
                     └──────────┬──────────┘
                                │
              ┌─────────────────┼─────────────────┐
              │                 │                 │
              ▼                 ▼                 ▼
       ┌─────────────┐   ┌──────────────┐  ┌─────────────┐
       │ PostgreSQL  │   │ GitHub API   │  │ LLM         │
       │ + pgvector  │   │              │  │             │
       │             │   │ Repository   │  │ Analysis    │
       │ Employees   │   │ Files        │  │ KT          │
       │ Projects    │   │ Contents     │  │ Answers     │
       │ Knowledge   │   │ APIs         │  │             │
       │ Embeddings  │   │ Modules      │  │             │
       └─────────────┘   └──────────────┘  └─────────────┘
                                │
                                ▼
                     ┌─────────────────────┐
                     │ PROJECT KNOWLEDGE   │
                     │ BASE                │
                     │                     │
                     │ Project overview    │
                     │ Architecture        │
                     │ Modules / APIs      │
                     │ Setup / Code        │
                     └──────────┬──────────┘
                                │
                                ▼
                         ┌─────────────┐
                         │    SLACK    │
                         │ KT Delivery │
                         │ Q&A Chat    │
                         └─────────────┘
```

### 2.2 End-to-End Employee Lifecycle Flow

1. **Slack Joined**: HR assigns employee; employee joins Slack community.
2. **Event Detection**: Slack sends `team_join` event to `POST /api/slack/events`. Employee status set to `ONBOARDING`.
3. **Greeting & Detail Collection**: Bot sends DM, greets employee, collects/verifies details (Name, Role, Department, Team, Team Lead, Project).
4. **Team Lead Approval**: Bot sends Slack DM with interactive buttons `[Approve]` / `[Reject]` to Team Lead.
5. **Repo Mapping & Analysis**: Upon approval (`POST /api/slack/interactions`), repository mapping is retrieved from company DB. LangGraph initiates repository analysis.
6. **Ingestion & Knowledge Generation**: Code parser extracts modules, endpoints, dependencies, and chunk embeddings stored in PostgreSQL + pgvector.
7. **KT Delivery**: Customized KT document generated and posted to Slack DM.
8. **Interactive Q&A**: Employee asks questions; RAG pipeline queries pgvector and returns grounded answers with file references.

---

## 3. Workflows & LangGraph Agent Architecture

### 3.1 Onboarding & Analysis Graph

```
START
  │
  ▼
detect_employee
  │
  ▼
collect_details
  │
  ▼
identify_team_lead
  │
  ▼
request_approval
  │
  ▼
WAIT (For Team Lead Interaction)
  │
  ▼
approval_check
  │
 ┌┴──────────────────────┐
 │                       │
[REJECT]              [APPROVE]
 │                       │
 ▼                       ▼
END                 find_project
                         │
                         ▼
                     find_repo
                         │
                         ▼
                     analyze_repo
                         │
                         ▼
                     build_knowledge
                         │
                         ▼
                       create_KT
                         │
                         ▼
                      send_KT
                         │
                         ▼
                        END
```

### 3.2 Question-Answering Router Graph

```
                 ┌──────────────┐
                 │ Slack Event  │
                 └──────┬───────┘
                        ▼
                 ┌──────────────┐
                 │ Router Node  │
                 └──────┬───────┘
                        │
             ┌──────────┼──────────┐
             ▼          ▼          ▼
        Onboarding     KT         Question
        Workflow    Analysis     Retrieval
             │          │          │
             ▼          ▼          ▼
        onboarding  analysis    retrieval
        graph       graph       graph
             │          │          │
             └──────────┼──────────┘
                        ▼
                 ┌──────────────┐
                 │ Slack Reply  │
                 └──────────────┘
```

### 3.3 LangGraph State Definition (`KTState`)

Shared state passed across graph nodes:

```typescript
export interface KTState {
  // Employee Metadata
  employeeId: string;
  slackUserId: string;
  employeeName: string;
  role: string;
  department: string;
  team: string;
  teamLeadId: string;
  projectId: string;
  repositoryId: string;

  // Approval Flow
  approvalStatus: 'PENDING' | 'APPROVED' | 'REJECTED';

  // Repository Analysis Outputs
  repositoryFiles: string[];
  projectSummary: string;
  architecture: string;
  modules: Array<{ name: string; description: string; path: string }>;
  apis: Array<{ endpoint: string; method: string; description: string }>;
  endpoints: string[];
  dependencies: Record<string, string>;

  // RAG & Knowledge Base
  knowledgeChunks: Array<{ id: string; content: string; source: string }>;
  ktDocument: string;

  // Interaction State
  question?: string;
  retrievedContext?: string[];
  answer?: string;
  error?: string;
}
```

### 3.4 Agent Tool Definitions
Agents interact with external systems using defined tools:
* `get_employee()`
* `get_project()`
* `get_repository()`
* `get_repository_file()`
* `search_project_knowledge()`
* `get_module()`
* `get_api()`
* `get_endpoint()`
* `get_project_summary()`
* `get_kt()`
* `send_slack_message()`

---

## 4. Repository Ingestion & RAG Specification

### 4.1 Artifact Exclusion & Focus Policy
To minimize noise, cost, and hallucination, the repository parser enforces strict file filtering.

#### Excluded Artifacts (8 Explicitly Excluded):
1. ❌ `Dockerfile`
2. ❌ `.gitignore`
3. ❌ Git history
4. ❌ Issues
5. ❌ Pull Requests
6. ❌ GitHub Wiki
7. ❌ Commit history
8. ❌ Source-code history

#### Included Artifacts:
* `README.md`
* Source code (`src/`, `lib/`, `app/`)
* Package & Build Manifests (`package.json`, `requirements.txt`, `pom.xml`, `build.gradle`, `tsconfig.json`)
* Configuration files & Environment examples (`.env.example`)
* Controllers, API Routes, Services, Models, Database schemas
* Module files, documentation files, and unit/integration tests

### 4.2 Ingestion Pipeline Architecture

```
GitHub Repository
   │
   ▼
Repository Fetcher (Tree API)
   │
   ▼
File Filter (Excludes 8 banned types)
   │
   ▼
Code Parser (Extracts AST, Modules, APIs)
   │
   ▼
Document Chunker
   │
   ▼
Metadata Extraction & Embeddings
   │
   ▼
PostgreSQL + pgvector Storage
```

### 4.3 Anti-Hallucination & Source Grounding Rules
1. **Context Check**: Answers must strictly derive from retrieved repository vector chunks.
2. **Missing Information Policy**: If context is insufficient, response must state:
   > *"I couldn't find enough evidence in the analyzed repository to confirm how [feature/topic] is implemented."*
3. **Source Citation**: Answers must explicitly list file path references:
   ```text
   Authentication is handled in the authentication middleware.

   Relevant files:
   • src/middleware/auth.ts
   • src/services/auth.service.ts
   ```

---

## 5. Data Architecture & Schema Specification

### 5.1 Relational Structure

```
departments ──► teams ──► projects ──► repositories
                             │
                             ▼
                     repository_files
                             │
                             ▼
                      knowledge_chunks
                             │
                             ▼
                        embeddings
```

```
employees ──► onboarding_sessions ──► approval_requests ──► kt_documents ──► conversations
```

### 5.2 Core Database Tables (PostgreSQL + pgvector)

1. `employees`: User profile, Slack ID, role, status enum (`ONBOARDING`, `WAITING_FOR_APPROVAL`, `APPROVED`, `ANALYZING`, `KT_READY`, `COMPLETED`).
2. `departments`: Company departments.
3. `teams`: Company teams and lead mappings.
4. `projects`: Project definitions.
5. `repositories`: Repository URLs and mapping to projects.
6. `onboarding_sessions`: Onboarding state per employee.
7. `approval_requests`: Status (`PENDING`, `APPROVED`, `REJECTED`) and timestamps for lead approvals.
8. `repository_files`: Index of ingested files and metadata.
9. `project_modules`: Extracted architectural modules.
10. `project_apis`: Parsed API definitions.
11. `project_endpoints`: Extracted HTTP endpoints.
12. `knowledge_documents`: Full generated knowledge bases.
13. `knowledge_chunks`: Vector chunks (`id`, `project_id`, `file_path`, `chunk_index`, `content`, `embedding vector`, `metadata`).
14. `kt_documents`: Generated KT sections delivered to employee.
15. `kt_progress`: Tracking section reading progress.
16. `conversations` & `messages`: Chat logs for Q&A history.
17. `oauth_tokens`: Encrypted GitHub OAuth tokens.
18. `agent_runs`: Audit log for LangGraph executions.

---

## 6. Monorepo Structure & Interface Contracts

### 6.1 Directory Structure

```text
kt-agent/
├── apps/
│   ├── backend/
│   │   ├── src/
│   │   │   ├── server.ts
│   │   │   ├── config/          # env.ts, database.ts
│   │   │   ├── routes/          # slack, github, employee, project, kt routes
│   │   │   ├── controllers/     # slack, github, kt controllers
│   │   │   ├── services/        # slack, github, employee, project, knowledge
│   │   │   ├── agents/          # onboarding, repository, kt, qa agents
│   │   │   ├── graphs/          # onboarding.graph.ts, analysis.graph.ts, qa.graph.ts
│   │   │   ├── tools/           # github, slack, database tools
│   │   │   ├── ingestion/       # repository loader, file filter, parser, chunker, embedder
│   │   │   ├── rag/             # retriever, reranker, prompt
│   │   │   ├── db/              # schema, migrations, repositories
│   │   │   ├── middleware/      # auth, error, logging
│   │   │   └── utils/
│   │   ├── tests/
│   │   └── package.json
│   └── frontend/
│       ├── src/                 # Admin/Middleware view for KT status & repo mapping
│       └── package.json
├── database/                    # Migrations & seed scripts
├── docs/                        # Architecture & API specs (contracts.md)
├── .env.example
├── docker-compose.yml
├── README.md
└── package.json
```

### 6.2 Primary API Endpoint Contracts

* `POST /api/slack/events` — Handles Slack `team_join` and user messages.
* `POST /api/slack/interactions` — Handles interactive button callbacks (`[Approve]`/`[Reject]`).
* `GET /api/github/auth` — Triggers GitHub OAuth flow.
* `GET /api/github/callback` — Validates OAuth state parameter and exchanges code for token.
* `POST /api/github/disconnect` — Revokes OAuth access.
* `GET /api/employees` / `GET /api/employees/:id` — Employee management.
* `GET /api/projects` / `GET /api/projects/:id` — Project details.
* `GET /api/kt/:employeeId` — Retrieves generated KT document.

---

## 7. Security, Authentication & Deployment Topology

### 7.1 Security Guidelines
* **OAuth Security**: Validate `state` parameter on `GET /api/github/callback` to prevent CSRF. Exchange authorization codes immediately.
* **Token Storage**: OAuth access tokens must be encrypted in PostgreSQL. Never log tokens or send them to Slack or client-side storage.
* **Scope Isolation**: Use designated team service accounts for repository OAuth authorizations during prototype/demo.

### 7.2 Environment Configuration (`.env`)
```bash
SLACK_BOT_TOKEN=xoxb-...
SLACK_SIGNING_SECRET=...
GITHUB_CLIENT_ID=...
GITHUB_CLIENT_SECRET=...
GITHUB_REDIRECT_URI=http://localhost:3000/api/github/callback
DATABASE_URL=postgresql://user:password@localhost:5432/kt_agent_db
LLM_API_KEY=...
EMBEDDING_API_KEY=...
NODE_ENV=development
PORT=3000
```

### 7.3 Render Deployment Topology

```
                    GitHub
                       │
                       ▼
                    Render
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
   Backend API      Frontend      PostgreSQL
   Node.js Service  React App     + pgvector
        │
        ▼
     Slack
        │
        ▼
   KT Agent
```

---

## 8. Team Ownership Matrix & Execution Roadmap

### 8.1 3-Person Ownership Distribution

To prevent bottlenecks, work is distributed across vertical slices with explicit interface contracts (`docs/contracts.md`):

```
                 PERSON 1
             Backend / Slack
                   │
          ┌────────┴─────────┐
          ▼                  ▼
     Person 2            Person 3
     AI Agent            Database
     LangGraph           GitHub / Ingestion
     RAG                 Frontend
```

| Domain / Responsibility | Person 1 (Platform & Backend Lead) | Person 2 (AI & LangGraph Lead) | Person 3 (Database & Ingestion Lead) |
| :--- | :--- | :--- | :--- |
| **Slack App & Onboarding** | **Lead** | Support | DB Mapping |
| **Team Lead Approval Flow** | **Lead** | Graph Node | DB Tables |
| **Node.js REST Backend** | **Lead** | API Integration | Support |
| **GitHub OAuth** | **Lead** | — | Support |
| **GitHub API & Ingestion** | Support | Analysis Logic | **Lead** |
| **LangGraph Workflows** | — | **Lead** | — |
| **RAG & Vector Retrieval** | — | **Lead** | DB/Vector Setup |
| **KT Generation & Q&A** | — | **Lead** | Support |
| **PostgreSQL & pgvector** | Support | Schema Requirements | **Lead** |
| **Frontend Dashboard** | — | — | **Lead** |
| **Deployment (Render)** | **Lead** | Support | Support |

### 8.2 Primary Integration Points
1. **Integration 1 (Slack → Backend)**: Owned by Person 1.
2. **Integration 2 (Backend → LangGraph)**: Person 1 + Person 2.
3. **Integration 3 (LangGraph → PostgreSQL)**: Person 2 + Person 3.
4. **Integration 4 (GitHub → Knowledge Base)**: Person 2 + Person 3.

### 8.3 Git Branching Strategy
* `main`: Production / Render deployment branch.
* `develop`: Integration branch.
* Feature branches:
  * `feature/slack-onboarding`
  * `feature/teamlead-approval`
  * `feature/github-oauth`
  * `feature/langgraph-analysis`
  * `feature/rag`
  * `feature/github-ingestion`
  * `feature/database`
  * `feature/frontend`

### 8.4 Phased Implementation Plan

```text
Phase 0 ──► Phase 1 ──► Phase 2 ──► Phase 3 ──► Phase 4 ──► Phase 5 ──► Phase 6/7 ──► Phase 8-15
 Architecture  Foundation   Employee    Approval    GitHub      Knowledge    KT & Q&A     Deployment
 Setup         & DB Schema  Onboarding  Workflow    OAuth       Ingestion    Engine       & Demo
```

* **Phase 0**: Architecture & Contract Setup (`docs/contracts.md`, DB Schema, Monorepo layout).
* **Phase 1**: Foundation (Node.js/Express, Slack event listener, PostgreSQL + pgvector setup).
* **Phase 2**: Employee Onboarding (Slack user detection, welcome DM, detail gathering).
* **Phase 3**: Approval Workflow (Interactive Slack buttons, `approval_requests` updates).
* **Phase 4**: GitHub Integration (OAuth flow, encrypted token storage, GitHub API loader).
* **Phase 5**: Knowledge Pipeline (File filtering, AST code parsing, text chunking, pgvector embedding generation).
* **Phase 6**: KT Generation (Generating 15 key architectural sections).
* **Phase 7**: Q&A Agent (Grounded vector search, hallucination checks, file citations).
* **Phase 8-15**: Integration testing, Render deployment, and end-to-end demo execution.

---

## 9. Deliverable KT Structure & Presentation Demo

### 9.1 Generated KT Section Outline (15 Key Sections)
1. Project Overview
2. Business Purpose
3. Architecture
4. Technology Stack
5. Repository Structure
6. Modules
7. APIs
8. Endpoints
9. Database Structure
10. Important Services
11. Development Setup
12. Configuration
13. Testing
14. Application Flow
15. Important Terminology

### 9.2 Scope Boundaries (MVP vs Out-of-Scope)

#### In Scope for MVP:
* Slack-native bot interface with interactive buttons.
* Team Lead approval workflow.
* Grounded repository ingestion & pgvector search.
* Automated 15-point KT document generation.
* Grounded Q&A with file source citations.

#### Out of Scope for Initial Version:
* Multi-company SaaS multi-tenancy.
* Kubernetes / complex microservice orchestrations.
* GitHub Issue, Pull Request, or Wiki analysis.
* Continuous background GitHub polling.
