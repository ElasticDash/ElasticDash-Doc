# ElasticDash Documentation

Welcome to the ElasticDash documentation! This guide covers the ElasticDash Logger (modified Langfuse) for capturing and storing LLM application traces.

---

## 📚 Documentation Index

### Getting Started

1. **[Architecture Overview](./docs/architecture.md)**
   - System design and component responsibilities
   - Data flow between Frontend, Backend, and Logger
   - Direct ClickHouse database access patterns
   - Multi-tenancy and security considerations
   - Deployment architecture

2. **[SDK Overview](./docs/sdk-overview.md)** ⭐ Start here for instrumentation
    - Python SDK (`elasticdash` package)
    - JavaScript/TypeScript SDK (`@elasticdash/*` packages)
    - Quick start guides
    - Installation and configuration
    - Framework integrations (OpenAI, LangChain)
    - **Note:** The SDK is only for tracing LLM behaviours. It is not intended for fetching traces or performing evaluation directly.

### Core Concepts

3. **[Data Model Reference](./docs/data-model.md)**
   - Traces, Sessions, and Observations explained
   - Attribute tables and examples
   - Nested observation hierarchies
   - Best practices for metadata and tags

### Advanced Usage

4. **[Fetching Data from Logger](./docs/fetching-data.md)**
    - REST API authentication and endpoints
    - Common use cases and pagination
    - Performance tips
    - **Note:** Once traces are fetched, they are evaluated and shown on your dashboard. Use the dashboard to check your existing traces, test cases, and test runs.

---

## 🚀 Quick Links

### For Application Developers

Start here to instrument your LLM application:

→ **[SDK Quick Start](./docs/sdk-overview.md#quick-start)**

### For Backend Developers

Learn how ElasticDash Backend accesses trace data:

→ **[Architecture: Direct Database Access](./docs/architecture.md#evaluation-flow)**

### For Data Engineers

Query traces programmatically for analysis:

→ **[Fetching Data Guide](./docs/fetching-data.md)**

---

## 📖 What is ElasticDash?

ElasticDash is a three-component system for LLM application testing and evaluation:

```
User Application
    │
    └─► ElasticDash Logger (captures traces)
            │
            └─► ClickHouse Database ◄─── ElasticDash Backend (evaluates)
                                              │
                                              └─► ElasticDash Frontend (displays results)
```

### Components

- **ElasticDash Logger**: Modified Langfuse that captures and stores traces from instrumented applications
- **ElasticDash Backend**: Node.js service that runs test cases and evaluations by querying ClickHouse directly
- **ElasticDash Frontend**: User interface for test management and result visualization

### Key Features

✅ **Trace Recording** - Capture detailed LLM execution traces
✅ **Session Grouping** - Organize multi-turn conversations
✅ **Direct Database Access** - Backend queries ClickHouse for high performance
✅ **Python & JS/TS SDKs** - Easy instrumentation with type safety
✅ **OpenTelemetry Foundation** - Standards-based observability

---

## 🛠️ Installation

### Python SDK

```bash
pip install elasticdash
```

### JavaScript/TypeScript SDK

```bash
npm install @elasticdash/tracing @elasticdash/otel @opentelemetry/sdk-node
```

See the **[SDK Overview](./docs/sdk-overview.md)** for complete setup instructions.

---

## 📝 Example: Instrumenting Your App

### Python

```python
from elasticdash import get_client

elasticdash = get_client()

# Create a trace
with elasticdash.start_as_current_observation(
    as_type="generation",
    name="gpt-4-call",
    model="gpt-4"
) as generation:
    generation.update(
        input={"prompt": "Hello"},
        output={"response": "Hi there!"}
    )

elasticdash.flush()  # For short-lived apps
```

### TypeScript

```typescript
import { startActiveObservation } from "@elasticdash/tracing";

await startActiveObservation("gpt-4-call", async (span) => {
  span.update({
    model: "gpt-4",
    input: { prompt: "Hello" },
    output: { response: "Hi there!" }
  });
});
```

See **[SDK Quick Start](./docs/sdk-overview.md#quick-start)** for complete examples.

---

## 🔐 Configuration

All SDKs support environment variable configuration:

```bash
ELASTICDASH_PUBLIC_KEY="pk-ed-..."
ELASTICDASH_SECRET_KEY="sk-ed-..."
ELASTICDASH_BASE_URL="https://logger.your-elasticdash.com"
```

---

## 🏗️ Architecture Highlights

### Data Flow

1. **User Application** instruments code with ElasticDash SDK
2. **SDK** sends traces to **ElasticDash Logger** via HTTP API
3. **Logger** stores traces in **ClickHouse database**
4. **ElasticDash Backend** queries ClickHouse directly (not via API) for evaluation
5. **Frontend** displays evaluation results to users

### Why Direct Database Access?

ElasticDash Backend queries ClickHouse directly instead of using the Logger's HTTP API for:

- 🚀 **Better performance** - Direct SQL queries are faster
- 🔧 **More flexibility** - Custom queries and joins
- 📊 **Advanced analytics** - Complex aggregations and filtering

See **[Architecture Overview](./docs/architecture.md)** for full details.

---

## 🎯 Common Use Cases

### 1. Trace a Conversation

```python
# Python example
with elasticdash.start_as_current_observation(
    as_type="span",
    name="chat-turn",
    session_id="session-123",
    user_id="user-alice"
) as span:
    # Your chat logic
    span.update(output="Response sent")
```

### 2. Fetch Traces for Evaluation

```python
# Backend queries ClickHouse directly for traces
traces = backend.query_clickhouse(
    "SELECT * FROM traces WHERE user_id = 'alice' LIMIT 100"
)
```

### 3. Analyze LLM Usage

```python
# Query via SDK (for external tools)
observations = elasticdash.api.observations_v_2.get_many(
    type="GENERATION",
    from_timestamp="2026-02-01T00:00:00Z"
)
```

See **[SDK Overview](./docs/sdk-overview.md#basic-usage)** for more examples.

---

## 📚 Additional Resources

- **Langfuse Documentation** - ElasticDash Logger is based on Langfuse: https://langfuse.com/docs
- **OpenTelemetry** - Learn about the OTel standard: https://opentelemetry.io/
- **ClickHouse** - Database documentation: https://clickhouse.com/docs

---

## 🤝 Support

For issues and questions related to ElasticDash documentation, please refer to:

- Architecture questions → **[Architecture Overview](./docs/architecture.md)**
- SDK usage questions → **[SDK Overview](./docs/sdk-overview.md)**
- Data model questions → **[Data Model Reference](./docs/data-model.md)**
- API questions → **[Fetching Data Guide](./docs/fetching-data.md)**

---

## 📋 Documentation Status

This documentation covers:

✅ **ElasticDash Logger** (modified Langfuse)
✅ **Python SDK** (`elasticdash`)
✅ **JavaScript/TypeScript SDK** (`@elasticdash/*`)
✅ **Data model** (Traces, Sessions, Observations)
✅ **Architecture** (component communication, database access)
✅ **Querying data** (REST API and SDK methods)

⏳ **Coming soon:**
- ElasticDash Backend API reference
- ElasticDash Frontend user guide
- Deployment and operations guide
- Evaluation methods and test cases

---

**Last Updated:** February 2026
