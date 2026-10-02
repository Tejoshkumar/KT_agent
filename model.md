1. Final KT Agent architecture
                         ┌─────────────────────┐
                         │   COMPANY SLACK     │
                         │                     │
                         │ New Employee joins  │
                         │ Messages / DMs      │
                         └──────────┬──────────┘
                                    │
                              Slack Events
                                    │
                                    ▼
                    ┌───────────────────────────┐
                    │      NODE.JS SERVER       │
                    │                           │
                    │ Slack Event Handler       │
                    │ REST APIs                 │
                    │ Authentication            │
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
                     │                     │
                     │ Agent State         │
                     │ Workflow Nodes      │
                     │ Conditional Edges   │
                     │ Tool Calls          │
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
                     │ Modules             │
                     │ APIs                │
                     │ Endpoints           │
                     │ Dependencies        │
                     │ Setup               │
                     │ Current code        │
                     └──────────┬──────────┘
                                │
                                ▼
                         ┌─────────────┐
                         │    SLACK    │
                         │             │
                         │ KT          │
                         │ Questions   │
                         │ Answers     │
                         └─────────────┘

LangGraph is a good fit here because its StateGraph lets nodes read/update shared state and connect those nodes into explicit workflows.

2. Most important design decision

Don't make one giant "AI agent".

Instead, create specialized workflows.

KT Agent
│
├── Onboarding Agent
│
├── Approval Workflow
│
├── Repository Analysis Agent
│
├── Knowledge Builder
│
├── KT Generator
│
└── Question Answering Agent

This makes the system much easier for three people to develop and debug.

3. Complete employee flow
Stage 1 — Employee joins Slack

Slack sends an event to:

POST /api/slack/events

Your backend identifies:

Slack User ID
Name
Email

Then:

employee.status = "ONBOARDING"
4. Welcome message

Bot sends:

Welcome to CompanyName 👋

I'm KT Agent, your AI onboarding assistant.

I'll help you understand your project,
team and development environment.

Let's get started.

Then collect:

Name
Role
Department
Team
Team Lead
Project

Don't make the employee type everything manually if Slack/company data already provides some of it.

5. Employee information

Database:

employees

Example:

id
slack_user_id
name
email
role
department
team
team_lead_id
project_id
onboarding_status
created_at
updated_at

Status:

ONBOARDING
WAITING_FOR_APPROVAL
APPROVED
ANALYZING
KT_READY
COMPLETED
6. Team Lead approval

This is a very important workflow.

Employee
   │
   ▼
Details collected
   │
   ▼
Find Team Lead
   │
   ▼
Slack DM to Team Lead

Message:

🔔 New Employee KT Request

Employee: Siraaj
Role: Software Engineer
Department: Engineering
Team: Payments
Project: Payment Gateway

The KT Agent wants to analyze the
project repository and prepare KT.

Approve?

[Approve] [Reject]

When Team Lead clicks:

Approve

Slack sends the interaction to:

POST /api/slack/interactions

Then:

approval.status = APPROVED

and LangGraph starts the repository-analysis workflow.

7. Your company database

You said the company database maps projects to repositories.

So I recommend:

departments
     │
     ▼
teams
     │
     ▼
projects
     │
     ▼
repositories

Example:

Department
Engineering

Team
Payments

Project
Payment Gateway

Repository
github.com/company/payment-gateway

The agent doesn't have to search GitHub to guess the repository.

That's much safer.

8. GitHub OAuth architecture

For your selected approach:

Admin/Team Lead
       │
       ▼
Connect GitHub
       │
       ▼
GitHub OAuth
       │
       ▼
Authorization
       │
       ▼
Callback
       │
       ▼
Backend
       │
       ▼
Store encrypted token

Endpoints:

GET /api/github/auth
GET /api/github/callback
POST /api/github/disconnect
GET /api/github/status

The OAuth callback should validate the state parameter before exchanging the authorization code. GitHub documents this flow and recommends state; authorization codes expire quickly, so exchange them promptly.

Do not put GitHub access tokens in Slack messages, frontend local storage, or normal logs.

9. GitHub analysis

You clarified that these 8 are excluded:

❌ Dockerfile
❌ .gitignore
❌ Git history
❌ Issues
❌ Pull Requests
❌ GitHub Wiki
❌ Commit history
❌ Source-code history

So your analyzer focuses on the current project/repository content and metadata such as:

README
Source code
Folder structure
package.json
requirements.txt
pom.xml
build.gradle
tsconfig.json
configuration files
API routes
controllers
services
models
database layer
frontend modules
backend modules
tests
documentation files
environment examples
dependency information
configuration
etc.
10. Don't send the entire repository to the LLM

This is a major architectural point.

Bad:

GitHub
   ↓
Entire repository
   ↓
LLM

Instead:

GitHub
   ↓
Repository Fetcher
   ↓
File Filter
   ↓
Code Parser
   ↓
Document Chunker
   ↓
Metadata Extraction
   ↓
Embeddings
   ↓
PostgreSQL + pgvector

pgvector lets PostgreSQL store vectors alongside your normal relational data and supports similarity search, including cosine distance and HNSW/IVFFlat indexes.

11. Repository ingestion pipeline
START
  │
  ▼
Get repository
  │
  ▼
Get repository tree
  │
  ▼
Filter files
  │
  ▼
Ignore excluded files
  │
  ▼
Download useful files
  │
  ▼
Detect language
  │
  ▼
Parse source
  │
  ▼
Extract:
 ├── modules
 ├── classes
 ├── functions
 ├── APIs
 ├── endpoints
 ├── dependencies
 └── configuration
  │
  ▼
Chunk content
  │
  ▼
Generate embeddings
  │
  ▼
Store PostgreSQL
  │
  ▼
Generate project summary
  │
  ▼
Generate KT
  │
  ▼
KT READY
12. LangGraph architecture

This is where LangGraph becomes very useful.

Create one graph for onboarding:

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
WAIT
  │
  ▼
approval_check
  │
 ┌┴───────────────┐
 │                │
REJECT          APPROVE
 │                │
 ▼                ▼
END         find_project
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
13. LangGraph state

Something like:

KTState

employeeId
slackUserId
employeeName
role
department
team
teamLeadId
projectId
repositoryId

approvalStatus

repositoryFiles
projectSummary
architecture
modules
apis
endpoints
dependencies

knowledgeChunks
ktDocument

question
retrievedContext
answer

error

The state is the memory of the current workflow.

14. Question-answering architecture

When the employee asks:

What does the payment service do?

Slack:

Employee
   ↓
Slack
   ↓
Backend
   ↓
Question Agent
   ↓
Retriever
   ↓
PostgreSQL / pgvector
   ↓
Relevant project chunks
   ↓
LLM
   ↓
Answer
   ↓
Slack

Example:

Employee:
What API is used to create a payment?

KT Agent:

The payment creation endpoint is:

POST /api/payments

It is handled by:
PaymentController

The request is passed to:
PaymentService

The service then communicates with:
PaymentRepository

The answer should be grounded in retrieved project information rather than allowing the model to invent an endpoint.

15. Separate "KT generation" from "Q&A"

This is important.

KT Generator

Produces:

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
14. Deployment/application flow
15. Important terminology
Q&A Agent

Answers:

"Where is authentication handled?"

"Which module handles payments?"

"What database does this project use?"

"Which API creates a payment?"

"How does the frontend communicate with backend?"

"What is the purpose of this service?"
16. Database architecture

I recommend these tables.

employees
teams
departments
projects
repositories

onboarding_sessions
approval_requests

repository_files
project_modules
project_apis
project_endpoints

knowledge_documents
knowledge_chunks

kt_documents
kt_progress

conversations
messages

oauth_tokens

agent_runs
17. Simplified database relationships
employees
    │
    │ belongs to
    ▼
teams
    │
    │ owns
    ▼
projects
    │
    │ mapped to
    ▼
repositories
    │
    ▼
repository_files
    │
    ▼
knowledge_chunks
    │
    ▼
embeddings

And:

employees
    │
    ▼
onboarding_sessions
    │
    ▼
approval_requests
    │
    ▼
kt_documents
    │
    ▼
conversations
18. Recommended project structure

Since you are using Node.js, I strongly recommend TypeScript rather than plain JavaScript.

kt-agent/
│
├── apps/
│   │
│   ├── backend/
│   │   ├── src/
│   │   │   │
│   │   │   ├── server.ts
│   │   │   │
│   │   │   ├── config/
│   │   │   │   ├── env.ts
│   │   │   │   └── database.ts
│   │   │   │
│   │   │   ├── routes/
│   │   │   │   ├── slack.routes.ts
│   │   │   │   ├── github.routes.ts
│   │   │   │   ├── employee.routes.ts
│   │   │   │   ├── project.routes.ts
│   │   │   │   └── kt.routes.ts
│   │   │   │
│   │   │   ├── controllers/
│   │   │   │   ├── slack.controller.ts
│   │   │   │   ├── github.controller.ts
│   │   │   │   └── kt.controller.ts
│   │   │   │
│   │   │   ├── services/
│   │   │   │   ├── slack/
│   │   │   │   ├── github/
│   │   │   │   ├── employee/
│   │   │   │   ├── project/
│   │   │   │   └── knowledge/
│   │   │   │
│   │   │   ├── agents/
│   │   │   │   ├── onboarding/
│   │   │   │   ├── repository/
│   │   │   │   ├── kt/
│   │   │   │   └── qa/
│   │   │   │
│   │   │   ├── graphs/
│   │   │   │   ├── onboarding.graph.ts
│   │   │   │   ├── analysis.graph.ts
│   │   │   │   └── qa.graph.ts
│   │   │   │
│   │   │   ├── tools/
│   │   │   │   ├── github.tools.ts
│   │   │   │   ├── slack.tools.ts
│   │   │   │   └── database.tools.ts
│   │   │   │
│   │   │   ├── ingestion/
│   │   │   │   ├── repository.loader.ts
│   │   │   │   ├── file.filter.ts
│   │   │   │   ├── code.parser.ts
│   │   │   │   ├── chunker.ts
│   │   │   │   └── embedder.ts
│   │   │   │
│   │   │   ├── rag/
│   │   │   │   ├── retriever.ts
│   │   │   │   ├── reranker.ts
│   │   │   │   └── prompt.ts
│   │   │   │
│   │   │   ├── db/
│   │   │   │   ├── schema/
│   │   │   │   ├── migrations/
│   │   │   │   └── repositories/
│   │   │   │
│   │   │   ├── middleware/
│   │   │   │   ├── auth.ts
│   │   │   │   ├── error.ts
│   │   │   │   └── logging.ts
│   │   │   │
│   │   │   └── utils/
│   │   │
│   │   ├── tests/
│   │   └── package.json
│   │
│   └── frontend/
│       ├── src/
│       │   ├── pages/
│       │   ├── components/
│       │   ├── services/
│       │   ├── hooks/
│       │   └── middleware/
│       └── package.json
│
├── database/
│   ├── migrations/
│   └── seed/
│
├── docs/
│   ├── architecture.md
│   ├── api.md
│   └── agent-flow.md
│
├── .env.example
├── docker-compose.yml
├── README.md
└── package.json
19. Now the most important part — 3-person division

Don't divide the work as:

Person 1 backend
Person 2 frontend
Person 3 AI

That creates dependencies and one person becomes a bottleneck.

Instead:

Person 1 — Platform + Slack + Backend Lead
Main responsibility

Employee lifecycle + Slack + APIs + authentication + backend infrastructure

They own:

Slack
Node.js
Express/Fastify
Slack Events
Slack interactions
Employee management
Team Lead approval
REST APIs
Authentication
GitHub OAuth integration
Render backend
Their files
routes/
controllers/
services/slack/
services/employee/
services/project/
middleware/
config/
server.ts
Their APIs
POST /api/slack/events

POST /api/slack/interactions

GET /api/github/auth

GET /api/github/callback

POST /api/github/disconnect

GET /api/employees

GET /api/employees/:id

GET /api/projects

GET /api/projects/:id

GET /api/kt/:employeeId
Person 1 delivers

Milestone 1

Slack Bot working

Milestone 2

Employee onboarding working

Milestone 3

Team Lead approval working

Milestone 4

GitHub OAuth working

Milestone 5

Backend deployed
Person 2 — AI + LangGraph Lead

This person owns the brain.

Main responsibility
LangChain
LangGraph
LLM
Prompt engineering
Repository analysis
KT generation
Question answering
Agent tools
Their files
agents/
graphs/
tools/
ingestion/
rag/
LangGraph workflows

They implement:

onboarding.graph.ts
analysis.graph.ts
qa.graph.ts
Analysis graph
START
 ↓
load_repository
 ↓
filter_files
 ↓
analyze_structure
 ↓
analyze_code
 ↓
extract_modules
 ↓
extract_apis
 ↓
extract_endpoints
 ↓
analyze_dependencies
 ↓
build_project_summary
 ↓
generate_KT
 ↓
store_knowledge
 ↓
END
Q&A graph
START
 ↓
understand_question
 ↓
retrieve_context
 ↓
check_context
 ↓
generate_answer
 ↓
validate_answer
 ↓
END
Person 2 delivers
Repository Analyzer
        +
Knowledge Builder
        +
KT Generator
        +
RAG
        +
Question Answering Agent
Person 3 — Database + GitHub Ingestion + Frontend

This person owns the knowledge infrastructure.

Main responsibility
PostgreSQL
pgvector
Database schema
GitHub repository ingestion
File processing
Embeddings
Frontend
Their files
db/
ingestion/
frontend/
GitHub ingestion
GitHub
  ↓
Repository Tree
  ↓
File Downloader
  ↓
File Filter
  ↓
Parser
  ↓
Chunker
  ↓
Embedding
  ↓
PostgreSQL
Database

They create:

employees
teams
projects
repositories
repository_files
project_modules
project_apis
project_endpoints
knowledge_chunks
kt_documents
conversations
messages
agent_runs
Frontend

Since you said the frontend is for another/middleware purpose, Person 3 should keep it thin initially.

It can provide:

Employee information
Project information
KT status
Analysis status
Repository status
KT progress

Don't let frontend development delay the core Slack agent.

20. Team dependency structure

This is how the three people should connect.

                 PERSON 1
             Backend / Slack
                   │
                   │
          ┌────────┴─────────┐
          │                  │
          ▼                  ▼
     Person 2            Person 3
     AI Agent            Database
     LangGraph           GitHub
     RAG                 Ingestion
     KT                  Frontend

But everyone works against interfaces, not each other's unfinished code.

21. Define interfaces first

Before coding, create:

docs/contracts.md

Example:

Slack Event
{
    slackUserId,
    eventType,
    message,
    timestamp
}

Employee:

{
    id,
    slackUserId,
    name,
    role,
    department,
    team,
    teamLeadId,
    projectId
}

Project:

{
    id,
    name,
    repositoryId,
    teamId
}

Repository:

{
    id,
    owner,
    name,
    url
}

Agent result:

{
    status,
    summary,
    modules,
    apis,
    endpoints,
    knowledgeDocumentId
}

This allows all three people to develop independently.

22. Git branch strategy

Use:

main
│
├── develop
│
├── feature/slack-onboarding
├── feature/teamlead-approval
├── feature/github-oauth
├── feature/langgraph-analysis
├── feature/rag
├── feature/github-ingestion
├── feature/database
└── feature/frontend

Never let three people directly edit the same files unnecessarily.

23. Development phases
Phase 0 — Day 1

All three together.

Build:

Architecture
Database schema
API contracts
GitHub repository structure
Environment variables
Development setup

Deliver:

README
architecture.md
database schema
API contract
folder structure
Phase 1 — Foundation
Person 1
Node.js
Express
Slack app
Slack events
Person 2
LangChain setup
LangGraph setup
LLM connection
Basic graph
Person 3
PostgreSQL
pgvector
Database schema
Migration system

At the end:

Slack → Backend → Database

works.

Phase 2 — Employee onboarding

Person 1:

Slack user detection
Welcome message
Employee details
Team Lead lookup

Person 3:

Employee tables
Team tables
Project tables
Repository mapping

Person 2:

Onboarding LangGraph
State management
Phase 3 — Approval

Person 1:

Slack interactive buttons
Approve
Reject

Person 2:

Approval state in LangGraph

Person 3:

approval_requests table

Flow:

New employee
      ↓
Details
      ↓
Team Lead
      ↓
[APPROVE]
      ↓
Analysis starts
Phase 4 — GitHub

Person 1:

GitHub OAuth
OAuth callback
Token handling

Person 3:

GitHub API client
Repository loader
File loader
File filtering

Person 2:

Repository analysis
Module extraction
API extraction
Endpoint extraction
Architecture analysis
Phase 5 — Knowledge Base

Person 3:

Chunking
Embedding
pgvector
Retrieval

Person 2:

Knowledge generation
Project summary
KT generation

Result:

GitHub
 ↓
Analysis
 ↓
Knowledge
 ↓
PostgreSQL
 ↓
Vector Search
Phase 6 — KT

Agent generates:

PROJECT KT
────────────

1. Project Overview

2. Business Purpose

3. Architecture

4. Technology Stack

5. Repository Structure

6. Modules

7. APIs

8. Endpoints

9. Database

10. Important Services

11. Application Flow

12. Development Setup

13. Configuration

14. Testing

15. Frequently Asked Questions

Then Slack sends:

🎓 Your KT is ready!

I've prepared your project KT.

1️⃣ Project Overview
2️⃣ Architecture
3️⃣ Modules
4️⃣ APIs
5️⃣ Application Flow
6️⃣ Development Setup

You can ask me questions anytime.
24. Phase 7 — Q&A

Employee:

@KT-Agent where is authentication handled?

Agent:

Question
 ↓
Intent
 ↓
Retriever
 ↓
PostgreSQL
 ↓
Relevant chunks
 ↓
LLM
 ↓
Grounding check
 ↓
Slack response
25. Prevent hallucinations

This is extremely important for your project.

The agent should have a rule:

If information exists in project knowledge:
    answer

If information is not found:
    say that the repository knowledge does not contain
    enough information to answer.

Don't allow:

LLM:
"I think this project uses JWT..."

Instead:

KT Agent:
"I couldn't find enough evidence in the analyzed
repository to confirm how authentication is implemented."

This will make the demo much more trustworthy.

26. Add source references to answers

For example:

Employee:
Where is payment authentication handled?

Agent:

Authentication is handled in the authentication
middleware.

Relevant files:
• src/middleware/auth.ts
• src/services/auth.service.ts

The middleware validates the authentication token
before requests reach protected routes.

This is much better than a generic chatbot.

27. Agent tools

Your LangGraph agent should have tools like:

get_employee()
get_project()
get_repository()
get_repository_file()
search_project_knowledge()
get_module()
get_api()
get_endpoint()
get_project_summary()
get_kt()
send_slack_message()

The AI should use tools to retrieve facts rather than directly guessing.

28. Example LangGraph architecture
                 ┌──────────────┐
                 │ Slack Event  │
                 └──────┬───────┘
                        ↓
                 ┌──────────────┐
                 │ Router Node  │
                 └──────┬───────┘
                        │
             ┌──────────┼──────────┐
             ↓          ↓          ↓
        onboarding     KT         question
             │          │          │
             ↓          ↓          ↓
        onboarding    analysis   retrieval
        workflow      workflow   workflow
             │          │          │
             └──────────┼──────────┘
                        ↓
                 ┌──────────────┐
                 │ Slack Reply  │
                 └──────────────┘
29. PostgreSQL + pgvector

I would use:

PostgreSQL
    │
    ├── Normal relational data
    │
    └── pgvector
           │
           └── Project embeddings

For example:

knowledge_chunks

id
project_id
file_path
chunk_index
content
embedding
metadata
created_at

Then similarity search:

Question
   ↓
Embedding
   ↓
Vector similarity
   ↓
Top relevant chunks
   ↓
LLM

pgvector supports vector columns and similarity operators, so it is a reasonable way to keep your relational data and vector knowledge in PostgreSQL.

30. Render deployment

For the prototype:

                    GitHub
                       │
                       ▼
                    Render
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
   Backend API      Frontend      PostgreSQL
   Node.js          React          + pgvector
        │
        ▼
     Slack
        │
        ▼
   KT Agent

Render can host Node.js web services and provides public HTTPS endpoints; your service needs to listen on 0.0.0.0 and the configured port.

For the first version, you don't need Kubernetes.

31. Environment variables

Never commit these.

SLACK_BOT_TOKEN=
SLACK_SIGNING_SECRET=

GITHUB_CLIENT_ID=
GITHUB_CLIENT_SECRET=
GITHUB_REDIRECT_URI=

DATABASE_URL=

LLM_API_KEY=

EMBEDDING_API_KEY=

NODE_ENV=
PORT=

Use:

.env

locally.

Render environment variables in deployment.

32. What the final demo should look like

This is the demo I would build for your presentation.

Step 1

New employee joins Slack.

👋 Welcome Siraaj!
I'm your KT Agent.
Let's get you onboarded.
Step 2

Agent asks:

What is your role?

Employee:

Software Engineer

Then:

Department?
Team?
Project?
Step 3

Agent finds:

Team Lead: Rahul
Project: Payment Gateway
Repository: company/payment-gateway
Step 4

Team Lead receives:

New KT request for Siraaj

Project:
Payment Gateway

[Approve]

Team Lead clicks:

APPROVE
Step 5

Agent starts:

🔍 Analyzing Payment Gateway...

✓ Repository structure
✓ Source code
✓ Modules
✓ APIs
✓ Endpoints
✓ Dependencies
✓ Configuration
✓ Application flow

Building project knowledge...
Step 6

Agent:

🎓 Your Project KT is Ready

Payment Gateway

Architecture
████████████

Modules
████████████

APIs
████████████

Application Flow
████████████

You can now ask me questions.
Step 7

Employee:

How does the payment request flow through the system?

Agent:

The request follows this flow:

API Request
    ↓
Payment Controller
    ↓
Payment Service
    ↓
Payment Repository
    ↓
Database

The relevant modules are:
...
33. MVP — don't build everything

For your college project, your MVP should be exactly this:

                    MVP
                     │
     ┌───────────────┼────────────────┐
     │               │                │
    Slack          GitHub          PostgreSQL
     │               │                │
     ▼               ▼                ▼
Employee          Repository        Knowledge
Onboarding        Analysis          Storage
     │               │                │
     ▼               ▼                ▼
Team Lead       Project KT          RAG
Approval             │                │
     │               └───────┬────────┘
     │                       │
     └───────────────────────▼
                          Q&A

Don't start with:

❌ Multi-company SaaS
❌ Complex autonomous agents
❌ Kubernetes
❌ Microservices
❌ Multiple vector databases
❌ Complex frontend
❌ Continuous GitHub monitoring
❌ Automatic project discovery
❌ GitHub issue/PR analysis

Your core innovation is:

A Slack-based AI KT Agent that understands a new employee's assigned project from its repository and provides personalized, project-grounded knowledge transfer.

34. Three-person ownership in one table
Area	Person 1	Person 2	Person 3
Slack App	Lead	Support	—
Employee onboarding	Lead	Support	DB
Team Lead approval	Lead	Graph	DB
Node.js backend	Lead	API integration	Support
GitHub OAuth	Lead	—	Support
GitHub API	Support	Analysis	Lead
Repository ingestion	—	Analysis logic	Lead
LangChain	—	Lead	Support
LangGraph	—	Lead	—
RAG	—	Lead	DB/Vector
KT generation	—	Lead	Support
Q&A	—	Lead	Support
PostgreSQL	Support	Schema requirements	Lead
pgvector	—	Retrieval	Lead
Frontend	—	—	Lead
Render	Lead	Support	Support
Testing	Integration	AI tests	DB/UI tests
Documentation	Architecture	AI architecture	DB/API
35. The three people should meet at these integration points

There should be only four major integration points.

Integration 1
Slack → Backend

Person 1 owns it.

Integration 2
Backend → LangGraph

Person 1 + Person 2.

Integration 3
LangGraph → PostgreSQL

Person 2 + Person 3.

Integration 4
GitHub → Knowledge Base

Person 2 + Person 3.

This prevents the team from constantly blocking each other.

36. Recommended development order

Follow this exact order:

WEEK / PHASE
     │
     ├── 1. Repository + architecture
     │
     ├── 2. PostgreSQL schema
     │
     ├── 3. Slack bot
     │
     ├── 4. Employee onboarding
     │
     ├── 5. Team Lead approval
     │
     ├── 6. GitHub OAuth
     │
     ├── 7. GitHub repository ingestion
     │
     ├── 8. Project analysis
     │
     ├── 9. Knowledge + embeddings
     │
     ├── 10. KT generation
     │
     ├── 11. RAG Q&A
     │
     ├── 12. Frontend/middleware
     │
     ├── 13. Integration testing
     │
     ├── 14. Render deployment
     │
     └── 15. Final demo

The key engineering principle is: build the deterministic workflow first, then add intelligence. Slack events, employee mapping, Team Lead approval, repository selection, and permissions should not depend on an LLM. The LLM/LangGraph layer should handle project understanding, KT generation, and grounded Q&A.

One final implementation note: because you chose GitHub OAuth, make the token owner a clearly defined authorized company/team account for the prototype. OAuth access is on behalf of the authorizing user, and GitHub notes that OAuth apps don't provide the same fine-grained repository permissions as GitHub Apps