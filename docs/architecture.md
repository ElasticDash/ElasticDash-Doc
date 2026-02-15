# ElasticDash Architecture

## Overview

ElasticDash is a three-component system for LLM application testing and evaluation, built on top of Langfuse's tracing infrastructure.

```text
┌─────────────────────┐
│  ElasticDash        │
│  Frontend           │
└──────────┬──────────┘
           │
           │ HTTP API
           ▼
┌─────────────────────┐
│  ElasticDash        │
│  Backend            │─────────────────────
└─────────────────────┘                    │
                                           |
                                           |
                                           ▼
┌─────────────────────┐         ┌─────────────────┐
│  ElasticDash        │         │   ClickHouse    │
│  Logger             │◄────────┤   Database      │
│  (Modified Langfuse)│ writes  └─────────────────┘
└─────────────────────┘                 ▲
           ▲                            │
           │                            │ Backend reads
           │ SDK/API                    │ traces directly
           │                            │
    ┌──────┴──────┐                     │
    │   User      │                     │
    │ Application │─────────────────────┘
    └─────────────┘
```

---

## Component Responsibilities

### ElasticDash Frontend

**Purpose**: User interface for managing test cases and viewing evaluation results.

**Key Functions**:
- Test case creation and management
- Evaluation result visualization
- User authentication interface

**Communication**:
- Connects to ElasticDash Backend via HTTP API

---

### ElasticDash Backend

**Purpose**: Core business logic for test execution and evaluation.

**Key Functions**:

- **Test case management**: Create, update, delete, and organize test cases
- **Evaluation engine**: Run automated and assisted evaluations
- **User authentication**: Manage user accounts and sessions
- **LLM key storage**: Securely store users' API keys for various LLM providers
- **Project management**: Organize tests and data by project
- **Trace retrieval**: Query Logger's ClickHouse database directly to fetch traces and sessions for evaluation

**Communication**:

- Exposes REST API to Frontend
- **Queries Logger's ClickHouse database directly** for trace data (not via HTTP API)

---

### ElasticDash Logger (Modified Langfuse)

**Purpose**: Capture and store traces and sessions from LLM applications. This is a modified version of Langfuse that focuses solely on observability.

**Key Functions**:
- **Trace recording**: Capture detailed execution traces from instrumented applications
- **Session grouping**: Group related traces into sessions (e.g., multi-turn conversations)
- **Data storage**: Persist traces, observations, and metadata
- **Query API**: Provide HTTP API and SDK access for retrieving stored data

**What Langfuse Features Are Used**:

- ✅ Trace and observation recording
- ✅ Session management
- ✅ Project organization
- ✅ ClickHouse data storage

**What Langfuse Features Are NOT Used**:

- ❌ Prompt management
- ❌ Evaluation (handled by ElasticDash Backend instead)
- ❌ Datasets
- ❌ LLM-as-a-Judge
- ❌ Custom dashboards
- ❌ Experiments

**Communication**:

- Receives traces via SDK or API from user applications
- Stores data in ClickHouse database
- ElasticDash Backend queries ClickHouse directly (bypassing Logger's API layer)

---

## Data Flow

### 1. Trace Recording Flow

```text
User Application
    │ (instrumented with Langfuse SDK)
    │
    ├─► Uses OpenAI/LangChain/etc.
    │   with Langfuse tracing enabled
    │
    └─► Sends traces asynchronously
        │
        ▼
ElasticDash Logger
    │
    ├─► Stores trace in ClickHouse
    └─► Returns trace ID
```

**Steps**:

1. Developer instruments their LLM application with Langfuse SDK
2. Application generates traces during execution (LLM calls, tool uses, etc.)
3. SDK sends traces asynchronously to ElasticDash Logger
4. Logger stores traces in ClickHouse with associated metadata (user ID, session ID, tags, etc.)

### 2. Evaluation Flow

```text
ElasticDash Backend
    │
    ├─► Direct SQL query to ClickHouse
    │   (filter by user_id, session_id, tags, time range)
    │
    ▼
ClickHouse Database
    │
    ├─► Returns matching traces with observations
    │
    ▼
ElasticDash Backend
    │
    ├─► Run evaluation logic on fetched traces
    ├─► Execute test cases
    ├─► Generate evaluation results
    │
    └─► Store results in Backend database
        │
        ▼
ElasticDash Frontend
    │
    └─► Display evaluation results to user
```

**Steps**:

1. Backend executes SQL queries directly against Logger's ClickHouse database
2. ClickHouse returns trace data including all nested observations
3. Backend evaluates traces against defined test cases
4. Results are stored in Backend database
5. Frontend displays results to end users

**Key Point**: ElasticDash Backend bypasses the Logger's HTTP API layer and queries ClickHouse directly for better performance and flexibility in data retrieval.

---

## Multi-Tenancy and Security

### Project Isolation

ElasticDash uses Langfuse's project system to isolate user data:

- Each user's traces are stored in separate projects within the Logger's ClickHouse database
- All projects are currently organized under a **single default user** in the Logger
- Multi-tenancy is enforced at the project level, not the user level
- ElasticDash Backend manages the mapping between users and projects
- Backend queries ClickHouse with project-specific filters to ensure data isolation

### Authentication

**For Users**:

- Users log in through ElasticDash Frontend
- Authentication is handled by ElasticDash Backend
- User credentials and LLM API keys are stored in Backend database

**For ElasticDash Backend → ClickHouse Communication**:

- Backend connects to ClickHouse using database credentials (username/password)
- Connection string includes host, port, database name, and authentication
- Backend has **read-only access** to Logger's ClickHouse database
- No HTTP API authentication needed between Backend and Logger

**For User Applications → Logger Communication**:

- Applications authenticate to Logger using Langfuse API keys
- Authentication uses HTTP Basic Auth (Public Key as username, Secret Key as password)
- API keys are project-scoped in the Logger

### Security Considerations for Self-Hosting

When deploying ElasticDash for cloud hosting:

1. **Network Isolation**: ClickHouse database should only be accessible from Backend and Logger containers
2. **Database Access Control**: Backend should have read-only ClickHouse credentials
3. **ClickHouse Authentication**: Use strong passwords and restrict access by IP/network
4. **LLM Key Storage**: Backend should encrypt user LLM keys at rest
5. **HTTPS**: All external communication must use HTTPS
6. **Database Security**: Both Backend database and ClickHouse should be isolated on private network

---

## Deployment Architecture

For self-hosting, all three components must be deployed and properly connected:

```text
┌──────────────────────────────────────────────────────────┐
│                    Docker Host                            │
│                                                           │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────┐         │
│  │  Frontend   │  │  Backend    │  │  Logger  │         │
│  │  Container  │  │  Container  │  │ Container│         │
│  └──────┬──────┘  └──────┬──────┘  └────┬─────┘         │
│         │                │              │                │
│         │ HTTP API       │ Writes       │                │
│         └────────────────┤              │                │
│                          │              │                │
│                          │              ▼                │
│                          │      ┌──────────────┐         │
│                          │      │  ClickHouse  │         │
│                          │      │   Database   │         │
│                          │      └──────────────┘         │
│                          │              ▲                │
│                          │              │                │
│                          └──────────────┘                │
│                            Reads (SQL queries)           │
│                                                           │
│  ┌─────────────┐                                         │
│  │  Backend DB │                                         │
│  │  (Postgres?)│                                         │
│  └─────────────┘                                         │
└──────────────────────────────────────────────────────────┘
```

**Key Points**:

- **Frontend** communicates with Backend via HTTP API
- **Logger** writes traces to ClickHouse database
- **Backend** reads traces directly from ClickHouse (bypasses Logger's API)
- **Backend** has its own database for test cases, evaluations, and user data

**Required Environment Variables** (examples):

**Frontend**:

- `BACKEND_URL`: URL of ElasticDash Backend

**Backend**:

- `CLICKHOUSE_HOST`: ClickHouse database host
- `CLICKHOUSE_PORT`: ClickHouse port (default: 8123 for HTTP, 9000 for native)
- `CLICKHOUSE_DATABASE`: ClickHouse database name (e.g., "langfuse")
- `CLICKHOUSE_USER`: ClickHouse username (read-only recommended)
- `CLICKHOUSE_PASSWORD`: ClickHouse password
- `DATABASE_URL`: Backend database connection string (for test cases, evaluations, etc.)

**Logger**:

- `CLICKHOUSE_URL`: ClickHouse connection URL (for Logger to write traces)
- `CLICKHOUSE_USER`: ClickHouse username (read-write)
- `CLICKHOUSE_PASSWORD`: ClickHouse password
- Additional Langfuse-specific configuration

---

## Summary

ElasticDash separates concerns clearly:

- **Logger**: Passive trace storage using Langfuse (writes to ClickHouse)
- **Backend**: Active test execution and evaluation logic (reads directly from ClickHouse)
- **Frontend**: User interface (communicates with Backend)

**Key Architectural Decisions**:

1. **Direct Database Access**: Backend queries ClickHouse directly rather than using Logger's HTTP API for better performance and flexibility
2. **Shared Data Store**: Logger writes traces to ClickHouse; Backend reads from the same ClickHouse database
3. **Read-Only Access**: Backend should use read-only ClickHouse credentials for security
4. **Separated Concerns**: Logger handles trace ingestion; Backend handles evaluation logic

This architecture allows ElasticDash to:

- Leverage Langfuse's robust tracing infrastructure for data capture
- Maintain full control over the test and evaluation workflow
- Achieve high-performance data retrieval via direct SQL queries
- Scale Backend evaluation independently from Logger trace ingestion

---

## See Also

- [Data Model Reference](./data-model.md) - Understanding traces, sessions, and observations
- [Fetching Data from Logger](./fetching-data.md) - API and SDK methods for external applications (not used by Backend)
