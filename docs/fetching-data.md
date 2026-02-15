# Fetching Data from ElasticDash Logger

This guide explains how to retrieve traces, sessions, and observations from the ElasticDash Logger using the REST API and SDKs.

> **Note for ElasticDash Backend Developers**: ElasticDash Backend does **not** use these APIs. Instead, it queries the Logger's ClickHouse database directly via SQL for better performance. This guide is intended for:
> - External applications integrating with ElasticDash Logger
> - Custom data analysis tools
> - Third-party services that need trace data
>
> See [Architecture Overview](./architecture.md) for details on Backend's direct database access.

---

## Overview

ElasticDash Logger exposes the Langfuse Public API, which provides:

- **RESTful endpoints** for fetching traces, sessions, observations, and scores
- **Python SDK** with type-safe wrappers
- **JavaScript/TypeScript SDK** with type-safe wrappers

All data is typically available for querying within **15-30 seconds** of ingestion.

---

## Authentication

ElasticDash Logger uses **HTTP Basic Authentication** for API access.

### API Keys

API keys are project-scoped and can be obtained from the Logger's project settings.

Each project has:
- **Public Key** (username): `pk-lf-...`
- **Secret Key** (password): `sk-lf-...`

### Authentication Example

```bash
curl -u pk-lf-xxx:sk-lf-xxx https://logger.example.com/api/public/traces
```

---

## REST API

### Base URL

The base URL depends on your deployment:

```
https://your-logger-host.com/api/public
```

For example:
```
https://logger.elasticdash.example.com/api/public
```

### API Reference

Full API documentation is available at:
- Langfuse API Reference: https://api.reference.langfuse.com
- OpenAPI Spec: `https://your-logger-host.com/generated/api/openapi.yml`

### Common Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/traces` | GET | List traces with filters |
| `/traces/{traceId}` | GET | Get single trace by ID |
| `/sessions` | GET | List sessions |
| `/sessions/{sessionId}` | GET | Get single session by ID |
| `/observations` | GET | List observations (v1) |
| `/observations-v2` | GET | List observations with better performance (v2) |
| `/scores` | GET | List scores |

---

## Python SDK

The Python SDK provides a convenient wrapper around the REST API.

### Installation

```bash
pip install langfuse
```

### Initialization

```python
from langfuse import get_client

# Initialize with environment variables
langfuse = get_client()

# Or explicitly pass credentials
langfuse = get_client(
    public_key="pk-lf-xxx",
    secret_key="sk-lf-xxx",
    host="https://logger.elasticdash.example.com"
)
```

### Environment Variables

```bash
LANGFUSE_PUBLIC_KEY="pk-lf-xxx"
LANGFUSE_SECRET_KEY="sk-lf-xxx"
LANGFUSE_BASE_URL="https://logger.elasticdash.example.com"
```

### Fetching Traces

#### List All Traces

```python
traces = langfuse.api.trace.list(limit=100)

for trace in traces.data:
    print(f"Trace ID: {trace.id}")
    print(f"Name: {trace.name}")
    print(f"User: {trace.user_id}")
    print(f"Session: {trace.session_id}")
    print("---")
```

#### Get Single Trace

```python
trace = langfuse.api.trace.get("trace-abc123")

print(f"Trace: {trace.name}")
print(f"Input: {trace.input}")
print(f"Output: {trace.output}")
print(f"Metadata: {trace.metadata}")
```

#### Filter Traces

```python
# Filter by user
traces = langfuse.api.trace.list(user_id="user-456", limit=50)

# Filter by session
traces = langfuse.api.trace.list(session_id="session-789", limit=50)

# Filter by tags
traces = langfuse.api.trace.list(tags=["production", "gpt-4"], limit=50)

# Filter by name
traces = langfuse.api.trace.list(name="chat-completion", limit=50)

# Time range filter (timestamps as ISO 8601 strings)
traces = langfuse.api.trace.list(
    from_timestamp="2026-02-01T00:00:00Z",
    to_timestamp="2026-02-15T23:59:59Z",
    limit=100
)
```

#### Pagination

```python
# First page
response = langfuse.api.trace.list(limit=100)
traces = response.data

# Next page (using cursor from previous response)
if response.meta.next:
    next_response = langfuse.api.trace.list(limit=100, cursor=response.meta.next)
    traces.extend(next_response.data)
```

### Fetching Sessions

#### List All Sessions

```python
sessions = langfuse.api.sessions.list(limit=50)

for session in sessions.data:
    print(f"Session ID: {session.id}")
    print(f"Created At: {session.created_at}")
    print("---")
```

#### Get Single Session

```python
session = langfuse.api.sessions.get("session-789")

print(f"Session ID: {session.id}")
print(f"Number of traces: {len(session.traces)}")
```

### Fetching Observations

#### Using V2 API (Recommended)

The v2 API offers better performance with cursor-based pagination and selective field retrieval.

```python
observations = langfuse.api.observations_v_2.get_many(
    trace_id="trace-abc123",
    type="GENERATION",
    limit=100,
    fields="core,basic,usage"  # Selective fields for better performance
)

for obs in observations.data:
    print(f"Observation: {obs.name}")
    print(f"Type: {obs.type}")
    print(f"Model: {obs.model}")
    print(f"Tokens: {obs.usage}")
    print("---")
```

**Available fields for selective retrieval**:
- `core`: Essential fields (id, type, name, timestamps)
- `basic`: Basic details (input, output, metadata)
- `usage`: Token usage and costs
- `scores`: Associated scores

#### Using V1 API

```python
# Get observations for a trace
observations = langfuse.api.observations.get_many(
    trace_id="trace-abc123",
    type="GENERATION",
    limit=100
)

# Get single observation
observation = langfuse.api.observations.get("obs-xyz789")
```

#### Filter by Type

```python
# Get only GENERATION observations (LLM calls)
generations = langfuse.api.observations_v_2.get_many(
    trace_id="trace-abc123",
    type="GENERATION"
)

# Get only SPAN observations
spans = langfuse.api.observations_v_2.get_many(
    trace_id="trace-abc123",
    type="SPAN"
)
```

### Async Support

All endpoints have async equivalents under `async_api`:

```python
import asyncio

async def fetch_traces():
    trace = await langfuse.async_api.trace.get("trace-abc123")
    print(trace.name)

    traces = await langfuse.async_api.trace.list(limit=100)
    print(f"Found {len(traces.data)} traces")

asyncio.run(fetch_traces())
```

---

## JavaScript/TypeScript SDK

### Installation

```bash
npm install @langfuse/client
```

### Initialization

```typescript
import { LangfuseClient } from '@langfuse/client';

// Initialize with environment variables
const langfuse = new LangfuseClient();

// Or explicitly pass credentials
const langfuse = new LangfuseClient({
  publicKey: "pk-lf-xxx",
  secretKey: "sk-lf-xxx",
  baseUrl: "https://logger.elasticdash.example.com"
});
```

### Environment Variables

```bash
LANGFUSE_PUBLIC_KEY="pk-lf-xxx"
LANGFUSE_SECRET_KEY="sk-lf-xxx"
LANGFUSE_BASE_URL="https://logger.elasticdash.example.com"
```

### Fetching Traces

#### List All Traces

```typescript
const traces = await langfuse.api.trace.list({ limit: 100 });

for (const trace of traces.data) {
  console.log(`Trace ID: ${trace.id}`);
  console.log(`Name: ${trace.name}`);
  console.log(`User: ${trace.userId}`);
}
```

#### Get Single Trace

```typescript
const trace = await langfuse.api.trace.get("trace-abc123");

console.log(`Trace: ${trace.name}`);
console.log(`Input:`, trace.input);
console.log(`Output:`, trace.output);
```

#### Filter Traces

```typescript
// Filter by user
const userTraces = await langfuse.api.trace.list({
  userId: "user-456",
  limit: 50
});

// Filter by session
const sessionTraces = await langfuse.api.trace.list({
  sessionId: "session-789",
  limit: 50
});

// Filter by tags
const taggedTraces = await langfuse.api.trace.list({
  tags: ["production", "gpt-4"],
  limit: 50
});

// Time range filter
const recentTraces = await langfuse.api.trace.list({
  fromTimestamp: "2026-02-01T00:00:00Z",
  toTimestamp: "2026-02-15T23:59:59Z",
  limit: 100
});
```

### Fetching Sessions

```typescript
// List sessions
const sessions = await langfuse.api.sessions.list({ limit: 50 });

// Get single session
const session = await langfuse.api.sessions.get("session-789");
console.log(`Session has ${session.traces.length} traces`);
```

### Fetching Observations

#### Using V2 API (Recommended)

```typescript
const observations = await langfuse.api.observationsV2.getMany({
  traceId: "trace-abc123",
  type: "GENERATION",
  limit: 100,
  fields: "core,basic,usage"
});

for (const obs of observations.data) {
  console.log(`${obs.name}: ${obs.model}`);
  console.log(`Tokens: ${obs.usage?.totalTokens}`);
}
```

#### Using V1 API

```typescript
// Get observations for a trace
const observations = await langfuse.api.observations.getMany({
  traceId: "trace-abc123",
  type: "GENERATION"
});

// Get single observation
const observation = await langfuse.api.observations.get("obs-xyz789");
```

---

## Common Use Cases

### 1. Fetch All Traces for a User

**Python**:
```python
user_traces = langfuse.api.trace.list(user_id="user-456", limit=1000)

for trace in user_traces.data:
    print(f"{trace.name}: {trace.output}")
```

**TypeScript**:
```typescript
const userTraces = await langfuse.api.trace.list({
  userId: "user-456",
  limit: 1000
});
```

### 2. Fetch Entire Conversation (Session)

**Python**:
```python
# Get session with all traces
session = langfuse.api.sessions.get("session-789")

# Get detailed traces
for trace_summary in session.traces:
    trace = langfuse.api.trace.get(trace_summary.id)
    print(f"User: {trace.input}")
    print(f"Bot: {trace.output}")
    print("---")
```

**TypeScript**:
```typescript
const session = await langfuse.api.sessions.get("session-789");

for (const traceSummary of session.traces) {
  const trace = await langfuse.api.trace.get(traceSummary.id);
  console.log(`User: ${trace.input}`);
  console.log(`Bot: ${trace.output}`);
}
```

### 3. Analyze LLM Usage and Costs

**Python**:
```python
# Get all GENERATION observations for a trace
observations = langfuse.api.observations_v_2.get_many(
    trace_id="trace-abc123",
    type="GENERATION",
    fields="core,basic,usage"
)

total_tokens = 0
total_cost = 0.0

for obs in observations.data:
    if obs.usage:
        total_tokens += obs.usage.get('totalTokens', 0)
    if obs.calculated_total_cost:
        total_cost += obs.calculated_total_cost

print(f"Total tokens: {total_tokens}")
print(f"Total cost: ${total_cost:.4f}")
```

### 4. Filter by Time Range and Tags

**Python**:
```python
production_traces = langfuse.api.trace.list(
    tags=["production"],
    from_timestamp="2026-02-15T00:00:00Z",
    to_timestamp="2026-02-15T23:59:59Z",
    limit=500
)

print(f"Found {len(production_traces.data)} production traces today")
```

### 5. Fetch Traces with Specific Metadata

**Python**:
```python
# Note: Direct metadata filtering might not be supported
# Fetch traces and filter in application code
all_traces = langfuse.api.trace.list(limit=1000)

# Filter by metadata in code
filtered = [
    trace for trace in all_traces.data
    if trace.metadata.get("experiment_id") == "exp-42"
]

print(f"Found {len(filtered)} traces for experiment exp-42")
```

### 6. Export Traces for Analysis

**Python**:
```python
import json

# Fetch all traces for a session
traces = langfuse.api.trace.list(session_id="session-789", limit=100)

# Export to JSON
with open("session_traces.json", "w") as f:
    json.dump([t.dict() for t in traces.data], f, indent=2, default=str)

print("Exported to session_traces.json")
```

---

## Pagination Best Practices

When fetching large datasets, use pagination to avoid timeouts and memory issues.

### Python Example

```python
def fetch_all_traces(user_id):
    all_traces = []
    cursor = None

    while True:
        response = langfuse.api.trace.list(
            user_id=user_id,
            limit=100,
            cursor=cursor
        )

        all_traces.extend(response.data)

        # Check if there are more pages
        if not response.meta.next:
            break

        cursor = response.meta.next

    return all_traces

traces = fetch_all_traces("user-456")
print(f"Fetched {len(traces)} traces")
```

### TypeScript Example

```typescript
async function fetchAllTraces(userId: string) {
  const allTraces = [];
  let cursor = undefined;

  while (true) {
    const response = await langfuse.api.trace.list({
      userId,
      limit: 100,
      cursor
    });

    allTraces.push(...response.data);

    if (!response.meta.next) {
      break;
    }

    cursor = response.meta.next;
  }

  return allTraces;
}

const traces = await fetchAllTraces("user-456");
console.log(`Fetched ${traces.length} traces`);
```

---

## Performance Considerations

### Use V2 APIs Where Available

The v2 APIs (e.g., `observations_v_2`) offer:
- Cursor-based pagination (faster than offset-based)
- Selective field retrieval (reduced payload size)
- Better query performance

```python
# Faster: only fetch fields you need
observations = langfuse.api.observations_v_2.get_many(
    trace_id="trace-abc123",
    fields="core,usage"  # Only core fields and usage
)

# Slower: fetches all fields
observations = langfuse.api.observations.get_many(trace_id="trace-abc123")
```

### Limit Result Sizes

Always specify a `limit` to avoid fetching too much data at once:

```python
# Good
traces = langfuse.api.trace.list(limit=100)

# Bad: might fetch thousands of traces
traces = langfuse.api.trace.list()
```

### Use Specific Filters

Filter at the API level rather than fetching all data and filtering in code:

```python
# Good: filter in API
traces = langfuse.api.trace.list(user_id="user-456", tags=["production"])

# Bad: fetch everything and filter in code
all_traces = langfuse.api.trace.list(limit=10000)
filtered = [t for t in all_traces.data if t.user_id == "user-456"]
```

---

## Error Handling

### Python

```python
from langfuse import LangfuseException

try:
    trace = langfuse.api.trace.get("invalid-trace-id")
except LangfuseException as e:
    print(f"Error fetching trace: {e}")
except Exception as e:
    print(f"Unexpected error: {e}")
```

### TypeScript

```typescript
try {
  const trace = await langfuse.api.trace.get("invalid-trace-id");
} catch (error) {
  console.error("Error fetching trace:", error);
}
```

---

## Rate Limits

ElasticDash Logger may have rate limits in place to prevent abuse. If you encounter rate limit errors:

1. Reduce request frequency
2. Use pagination with smaller page sizes
3. Cache results where possible
4. Implement exponential backoff retry logic

---

## See Also

- [Data Model Reference](./data-model.md) - Understanding traces, sessions, and observations
- [Architecture Overview](./architecture.md) - How ElasticDash components interact
- Langfuse API Reference: https://api.reference.langfuse.com
