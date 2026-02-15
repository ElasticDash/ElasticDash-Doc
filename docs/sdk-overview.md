# ElasticDash SDKs

ElasticDash provides fully-typed SDKs for instrumenting your LLM applications and sending traces to the ElasticDash Logger.

- **Python** - `elasticdash` package on PyPI
- **JavaScript/TypeScript** - Modular packages under `@elasticdash/*` on npm

The ElasticDash SDKs are the recommended way to create traces and observations in your LLM applications for testing and evaluation.

---

## What Do the SDKs Do?

The ElasticDash SDKs enable you to:

1. **Instrument your application** - Capture traces of LLM calls, tool executions, and custom logic
2. **Record observations** - Track individual steps like model generations, retrievals, and events
3. **Add metadata** - Attach user IDs, session IDs, tags, and custom metadata to traces
4. **Async tracing** - Send traces in the background without adding latency to your application
5. **Automatic nesting** - Create hierarchical traces that show parent-child relationships

**Key Benefits:**

- Built on [OpenTelemetry](https://opentelemetry.io/) standard
- Fully async requests (no latency impact)
- Accurate latency tracking with synchronous timestamps
- Type-safe APIs with IntelliSense support
- Automatic error handling (SDK errors won't break your app)
- Compatible with popular frameworks (OpenAI, LangChain, etc.)

---

## Quick Start

### Python SDK

**1. Install package:**

```bash
pip install elasticdash
```

**2. Add credentials:**

```bash
# .env file
ELASTICDASH_SECRET_KEY="sk-ed-..."
ELASTICDASH_PUBLIC_KEY="pk-ed-..."
ELASTICDASH_BASE_URL="https://logger.your-elasticdash.com"
```

**3. Instrument your application:**

```python
from elasticdash import get_client

# Initialize client
elasticdash = get_client()

# Create a span using context manager
with elasticdash.start_as_current_observation(
    as_type="span",
    name="process-user-query"
) as span:
    # Your processing logic
    span.update(input="User question", output="Processing started")

    # Create nested generation for LLM call
    with elasticdash.start_as_current_observation(
        as_type="generation",
        name="gpt-4-completion",
        model="gpt-4"
    ) as generation:
        # Your LLM call here
        generation.update(
            input={"prompt": "What is the capital of France?"},
            output={"response": "The capital of France is Paris."},
            usage={"prompt_tokens": 10, "completion_tokens": 8}
        )

# Flush events (required for short-lived apps)
elasticdash.flush()
```

**4. See your trace in ElasticDash Logger**

Navigate to your ElasticDash Logger UI to view the captured trace.

---

### JavaScript/TypeScript SDK

**1. Install packages:**

```bash
npm install @elasticdash/tracing @elasticdash/otel @opentelemetry/sdk-node
```

**2. Set environment variables:**

```bash
# .env file
ELASTICDASH_SECRET_KEY="sk-ed-..."
ELASTICDASH_PUBLIC_KEY="pk-ed-..."
ELASTICDASH_BASE_URL="https://logger.your-elasticdash.com"
```

**3. Initialize OpenTelemetry:**

Create an `instrumentation.ts` file:

```typescript
import { NodeSDK } from "@opentelemetry/sdk-node";
import { ElasticDashSpanProcessor } from "@elasticdash/otel";

export const sdk = new NodeSDK({
  spanProcessors: [new ElasticDashSpanProcessor()],
});

sdk.start();
```

**4. Instrument your application:**

```typescript
import "./instrumentation"; // Must be first import
import { startActiveObservation } from "@elasticdash/tracing";

async function main() {
  await startActiveObservation("user-query-trace", async (span) => {
    span.update({
      input: { query: "What is the capital of France?" },
      output: { answer: "Paris" },
    });
  });
}

// Shutdown flushes events (required for short-lived apps)
main().finally(() => sdk.shutdown());
```

**5. Run and see your trace:**

```bash
npx tsx index.ts
```

---

## Installation

### Python SDK

Install the ElasticDash Python SDK from PyPI:

```bash
pip install elasticdash
```

**Requirements:**
- Python 3.8 or higher
- Compatible with async/await patterns

### JavaScript/TypeScript SDK

The ElasticDash JS/TS SDK is modular. Install the packages you need:

**Basic tracing setup:**

```bash
npm install @elasticdash/tracing @elasticdash/otel @opentelemetry/sdk-node
```

**All available packages:**

| Package | Description | Environment |
|---------|-------------|-------------|
| **`@elasticdash/core`** | Core utilities, types, and logger shared across packages | Universal JS |
| **`@elasticdash/client`** | Client for querying traces, sessions, and scores | Universal JS |
| **`@elasticdash/tracing`** | Core OpenTelemetry-based tracing functions | Universal JS |
| **`@elasticdash/otel`** | `ElasticDashSpanProcessor` to export traces to Logger | Node.js ≥ 20 |
| **`@elasticdash/openai`** | Automatic tracing for OpenAI SDK | Universal JS |
| **`@elasticdash/langchain`** | CallbackHandler for LangChain applications | Universal JS |

**Install specific integrations:**

```bash
# For OpenAI integration
npm install @elasticdash/openai

# For LangChain integration
npm install @elasticdash/langchain

# For querying data
npm install @elasticdash/client
```

---

## Configuration

### Environment Variables

Both SDKs support configuration via environment variables:

```bash
# Required
ELASTICDASH_PUBLIC_KEY="pk-ed-..."
ELASTICDASH_SECRET_KEY="sk-ed-..."

# Required for self-hosted Logger
ELASTICDASH_BASE_URL="https://logger.your-elasticdash.com"
```

### Python: Client Initialization

**Using environment variables (recommended):**

```python
from elasticdash import get_client

# Automatically uses environment variables
elasticdash = get_client()

# Verify connection
if elasticdash.auth_check():
    print("ElasticDash client authenticated!")
else:
    print("Authentication failed")
```

**Using constructor:**

```python
from elasticdash import ElasticDash

elasticdash = ElasticDash(
    public_key="pk-ed-...",
    secret_key="sk-ed-...",
    base_url="https://logger.your-elasticdash.com"
)
```

The ElasticDash client is a singleton. Access it anywhere with `get_client()`.

### JavaScript/TypeScript: Client Initialization

**Using environment variables:**

```typescript
import { ElasticDashClient } from "@elasticdash/client";

// Automatically uses environment variables
const elasticdash = new ElasticDashClient();
```

**Using constructor:**

```typescript
import { ElasticDashClient } from "@elasticdash/client";

const elasticdash = new ElasticDashClient({
  publicKey: "pk-ed-...",
  secretKey: "sk-ed-...",
  baseUrl: "https://logger.your-elasticdash.com"
});
```

---

## Basic Usage

### Creating Traces

**Python:**

```python
from elasticdash import get_client

elasticdash = get_client()

# Context manager (recommended)
with elasticdash.start_as_current_observation(
    as_type="span",
    name="my-operation"
) as span:
    span.update(
        input="Input data",
        output="Output data",
        metadata={"user": "alice"}
    )
```

**TypeScript:**

```typescript
import { startActiveObservation } from "@elasticdash/tracing";

await startActiveObservation("my-operation", async (span) => {
  span.update({
    input: "Input data",
    output: "Output data",
    metadata: { user: "alice" }
  });
});
```

### Recording LLM Generations

**Python:**

```python
with elasticdash.start_as_current_observation(
    as_type="generation",
    name="gpt-4-call",
    model="gpt-4"
) as generation:
    # Your OpenAI call
    response = openai_client.chat.completions.create(
        model="gpt-4",
        messages=[{"role": "user", "content": "Hello"}]
    )

    generation.update(
        input={"messages": [{"role": "user", "content": "Hello"}]},
        output={"content": response.choices[0].message.content},
        usage={
            "prompt_tokens": response.usage.prompt_tokens,
            "completion_tokens": response.usage.completion_tokens,
            "total_tokens": response.usage.total_tokens
        }
    )
```

**TypeScript:**

```typescript
import { startObservation } from "@elasticdash/tracing";

const generation = startObservation(
  "gpt-4-call",
  {
    model: "gpt-4",
    input: { messages: [{ role: "user", content: "Hello" }] }
  },
  { asType: "generation" }
);

// Your OpenAI call
const response = await openai.chat.completions.create({
  model: "gpt-4",
  messages: [{ role: "user", content: "Hello" }]
});

generation.update({
  output: { content: response.choices[0].message.content },
  usage: {
    promptTokens: response.usage.prompt_tokens,
    completionTokens: response.usage.completion_tokens
  }
}).end();
```

### Nested Observations

**Python:**

```python
with elasticdash.start_as_current_observation(
    as_type="span",
    name="parent-operation"
) as parent:

    # Child observation 1
    with elasticdash.start_as_current_observation(
        as_type="span",
        name="child-1"
    ) as child1:
        child1.update(output="Child 1 complete")

    # Child observation 2
    with elasticdash.start_as_current_observation(
        as_type="generation",
        name="child-2",
        model="gpt-4"
    ) as child2:
        child2.update(output="Child 2 complete")

    parent.update(output="Parent complete")
```

**TypeScript:**

```typescript
await startActiveObservation("parent-operation", async (parent) => {

  // Child observation 1
  const child1 = startObservation("child-1", {}, { asType: "span" });
  child1.update({ output: "Child 1 complete" }).end();

  // Child observation 2
  const child2 = startObservation(
    "child-2",
    { model: "gpt-4" },
    { asType: "generation" }
  );
  child2.update({ output: "Child 2 complete" }).end();

  parent.update({ output: "Parent complete" });
});
```

### Adding Metadata and Tags

**Python:**

```python
with elasticdash.start_as_current_observation(
    as_type="span",
    name="operation",
    user_id="user-123",
    session_id="session-456",
    tags=["production", "experiment-A"],
    metadata={
        "customer_tier": "premium",
        "region": "us-east-1"
    }
) as span:
    span.update(output="Complete")
```

**TypeScript:**

```typescript
await startActiveObservation("operation", async (span) => {
  span.update({
    userId: "user-123",
    sessionId: "session-456",
    tags: ["production", "experiment-A"],
    metadata: {
      customerTier: "premium",
      region: "us-east-1"
    },
    output: "Complete"
  });
});
```

---

## Flushing Events

For short-lived applications (scripts, serverless functions), you must flush events before the process exits to ensure all traces are sent.

**Python:**

```python
from elasticdash import get_client

elasticdash = get_client()

# Your tracing code here
with elasticdash.start_as_current_observation(
    as_type="span",
    name="script-task"
) as span:
    span.update(output="Task complete")

# Flush before exit (blocks until all events sent)
elasticdash.flush()
```

**TypeScript:**

```typescript
import { sdk } from "./instrumentation";
import { startActiveObservation } from "@elasticdash/tracing";

async function main() {
  await startActiveObservation("script-task", async (span) => {
    span.update({ output: "Task complete" });
  });
}

// Shutdown before exit (flushes all events)
main().finally(() => sdk.shutdown());
```

**When to flush:**
- ✅ Scripts and CLI tools
- ✅ Serverless functions (Lambda, Cloud Functions)
- ✅ Batch jobs and cron tasks
- ❌ Long-running servers (automatic background flushing)

---

## Framework Integrations

### OpenAI SDK Integration

**Python:**

```python
from elasticdash.openai import openai

# Use the wrapped OpenAI client (drop-in replacement)
completion = openai.chat.completions.create(
    name="chat-completion",
    model="gpt-4",
    messages=[{"role": "user", "content": "Hello"}],
    metadata={"user_id": "alice"}
)

# Automatically traced in ElasticDash!
```

**TypeScript:**

```typescript
import OpenAI from "openai";
import { observeOpenAI } from "@elasticdash/openai";

// Wrap your OpenAI client
const openai = observeOpenAI(new OpenAI());

// Use normally - automatically traced
const response = await openai.chat.completions.create({
  model: "gpt-4",
  messages: [{ role: "user", content: "Hello" }]
});
```

### LangChain Integration

**Python:**

```python
from elasticdash.langchain import CallbackHandler
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate

# Initialize callback handler
elasticdash_handler = CallbackHandler()

# Use with LangChain
llm = ChatOpenAI(model_name="gpt-4")
prompt = ChatPromptTemplate.from_template("Tell me about {topic}")
chain = prompt | llm

response = chain.invoke(
    {"topic": "machine learning"},
    config={"callbacks": [elasticdash_handler]}
)
```

**TypeScript:**

```typescript
import { CallbackHandler } from "@elasticdash/langchain";
import { ChatOpenAI } from "@langchain/openai";

// Initialize callback handler
const elasticdashHandler = new CallbackHandler();

// Use with LangChain
const llm = new ChatOpenAI({ model: "gpt-4" });

const response = await llm.invoke(
  [{ role: "user", content: "Tell me about machine learning" }],
  { callbacks: [elasticdashHandler] }
);
```

---

## OpenTelemetry Foundation

ElasticDash SDKs are built on [OpenTelemetry](https://opentelemetry.io/), providing:

- **Standardization** with the observability ecosystem
- **Context propagation** for nested spans across async workloads
- **Attribute propagation** for metadata, tags, user IDs, etc.
- **Interoperability** with third-party OTel instrumentations

**Mapping:**

| OpenTelemetry | ElasticDash |
|---------------|-------------|
| OTel Trace | ElasticDash Trace |
| OTel Span | ElasticDash Observation (Span/Generation/Event) |
| Span Attributes | Trace/Observation metadata, tags, user_id |

---

## Advanced Features

### Querying Data via SDK

**Python:**

```python
from elasticdash import get_client

elasticdash = get_client()

# Fetch traces
traces = elasticdash.api.trace.list(
    user_id="user-123",
    limit=100
)

# Fetch single trace
trace = elasticdash.api.trace.get("trace-id-abc")

# Fetch observations
observations = elasticdash.api.observations_v_2.get_many(
    trace_id="trace-id-abc",
    type="GENERATION"
)
```

**TypeScript:**

```typescript
import { ElasticDashClient } from "@elasticdash/client";

const elasticdash = new ElasticDashClient();

// Fetch traces
const traces = await elasticdash.api.trace.list({
  userId: "user-123",
  limit: 100
});

// Fetch single trace
const trace = await elasticdash.api.trace.get("trace-id-abc");

// Fetch observations
const observations = await elasticdash.api.observationsV2.getMany({
  traceId: "trace-id-abc",
  type: "GENERATION"
});
```

### Custom Observation Types

**Python:**

```python
# Record an event (point in time, no duration)
with elasticdash.start_as_current_observation(
    as_type="event",
    name="cache-hit"
) as event:
    event.update(metadata={"cache_key": "user-123-query"})
```

**TypeScript:**

```typescript
// Record an event
const event = startObservation(
  "cache-hit",
  { metadata: { cacheKey: "user-123-query" } },
  { asType: "event" }
);
event.end();
```

---

## Troubleshooting

### Authentication Errors

**Problem:** SDK cannot connect to ElasticDash Logger

**Solution:**

```python
# Python: Check authentication
from elasticdash import get_client

elasticdash = get_client()
if not elasticdash.auth_check():
    print("Check your credentials and base URL")
```

### Traces Not Appearing

**Problem:** Traces not visible in Logger UI

**Solutions:**

1. **Flush events** in short-lived apps:
   ```python
   elasticdash.flush()  # Python
   ```
   ```typescript
   sdk.shutdown()  // TypeScript
   ```

2. **Check base URL** - Ensure `ELASTICDASH_BASE_URL` points to your Logger

3. **Wait 15-30 seconds** - Data may take time to process

### Import Errors (TypeScript)

**Problem:** Cannot find module `@elasticdash/tracing`

**Solution:** Ensure OpenTelemetry SDK is installed:

```bash
npm install @opentelemetry/sdk-node
```

---

## See Also

- [Data Model Reference](./data-model.md) - Understanding traces, sessions, observations
- [Fetching Data](./fetching-data.md) - Query Logger for traces and sessions
- [Architecture Overview](./architecture.md) - How ElasticDash components work together
