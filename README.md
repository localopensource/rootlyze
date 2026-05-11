<div align="center">

# rootlyze

**Automated incident diagnosis for production systems.**

From raw logs to root cause — without the manual triage.

[![Build Status](https://img.shields.io/badge/build-passing-brightgreen)](https://github.com/localopensource/rootlyze)
[![Language](https://img.shields.io/badge/language-C%2B%2B17-blue)](https://isocpp.org/)
[![License](https://img.shields.io/badge/license-MIT-lightgrey)](LICENSE)
[![Status](https://img.shields.io/badge/status-early%20development-orange)](https://github.com/localopensource/rootlyze)

</div>

---

## Table of Contents

- [What is rootlyze?](#what-is-rootlyze)
- [Quick Example](#quick-example)
- [System Architecture and Design](#system-architecture-and-design)
  - [Design Philosophy](#design-philosophy)
  - [High-Level Architecture](#high-level-architecture)
  - [Pipeline Design](#pipeline-design)
  - [Component Responsibilities](#component-responsibilities)
  - [AI Research Engine](#ai-research-engine)
  - [Inter-Component Communication](#inter-component-communication)
  - [Concurrency Model](#concurrency-model)
  - [Memory Model](#memory-model)
  - [Error Handling Strategy](#error-handling-strategy)
  - [Extensibility Points](#extensibility-points)
- [Getting Started](#getting-started)
- [External Usage Guide](#external-usage-guide)
- [Internal Documentation](#internal-documentation)
- [Configuration](#configuration)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [Licence](#licence)
- [Author](#author)

---

## What is rootlyze?

rootlyze is a C++17 incident diagnosis system that ingests logs from production services, processes and correlates events, and produces human-readable root cause reports. It combines runtime log data with indexed system documentation and live AI-assisted research to suggest fixes grounded in both your own docs and current external knowledge.

It is built for engineers who are tired of spending the first hour of an incident manually grepping through logs across five services, cross-referencing runbooks, and searching Stack Overflow to find an answer that the system should already know.

**The problem rootlyze solves:**

- Logs are scattered across services with inconsistent formats
- Engineers must mentally correlate events across time and systems under pressure
- Documentation and runbooks are rarely consulted during an incident
- External knowledge (known bugs, upstream issues, community fixes) is never surfaced automatically
- Mean time to diagnosis is dominated by manual work, not the actual fix

**What rootlyze does instead:**

1. Receives logs from your services via HTTP or file-based agent
2. Normalises and correlates events across services into a unified timeline
3. Cross-references runtime events against your indexed system documentation
4. Searches external sources using an AI research engine to surface known issues, upstream bugs, and community-verified fixes
5. Combines internal and external knowledge to produce a structured root cause report with an evidence-backed fix suggestion

---

## Quick Example

**Input: logs received from two services**

```
[14:32:00] auth-service    ERROR  DB connection timeout after 30s
[14:32:01] auth-service    ERROR  DB connection timeout after 30s
[14:32:03] auth-service    ERROR  Retry limit exceeded — dropping request
[14:32:04] api-gateway     ERROR  Upstream auth-service unresponsive
[14:32:05] api-gateway     ERROR  Circuit breaker opened for auth-service
```

**Output: rootlyze diagnosis**

```
Root Cause:
  Connection pool exhaustion in auth-service caused cascading upstream
  failure in api-gateway.

Evidence:
  [14:32:00-14:32:03] auth-service: 3x DB connection timeout, retry limit hit
  [14:32:04-14:32:05] api-gateway: auth-service marked unresponsive,
                      circuit breaker opened

Internal Reference:
  docs/services/auth-service.md#database-configuration
  Relevant passage: "max_connections defaults to 10; increase under high
  concurrency workloads."

External Research:
  Known issue: PostgreSQL connection pool exhaustion under burst traffic
  Source: postgresql.org/docs — confirmed behaviour, not a bug
  Community fix: pgBouncer connection pooling layer recommended for
  services with >50 concurrent users

Suggested Fix:
  1. Increase max_connections in auth-service database configuration
     from the default of 10 to a value appropriate for your concurrency
  2. Consider introducing pgBouncer as a connection pooling layer
     between auth-service and the database
  3. Add connection pool monitoring to alert before exhaustion occurs
```

Notice that rootlyze pulls fix step 2 from external research, not just your internal docs. This is the core value of the AI research layer.

---

## System Architecture and Design

### Design Philosophy

rootlyze is built around five core principles that govern every architectural decision.

**1. Separation of concerns above all else**
Each component in the pipeline has exactly one job. Ingestion does not parse. Parsing does not diagnose. Diagnosis does not format output. This makes each component independently testable, replaceable, and debuggable without touching the rest of the system.

**2. Defensive processing over strict validation**
Production logs are messy. Services crash mid-write, formatters differ across frameworks, and fields go missing under load. rootlyze is designed to degrade gracefully at every stage rather than halt the pipeline on bad input. A malformed log should produce a warning, not a crash.

**3. Interface stability over implementation stability**
Component implementations will change significantly across phases. The interfaces between components are designed to be stable across those transitions. Swapping the diagnosis engine from rule-based to AI-driven, or the documentation index from keyword-based to vector-based, should not require changes to surrounding components.

**4. Performance as a first-class constraint**
The system is implemented in C++17 because log volumes at production scale are high and ingestion latency must be low. Memory allocations in the hot path are minimised. The concurrency model keeps the ingestion layer non-blocking regardless of downstream processing time.

**5. Research before assertion**
rootlyze does not guess. When it produces a fix suggestion, that suggestion is grounded in evidence: either your own documentation, a verified external source, or both. The AI research engine cites its sources. Every fix suggestion includes a provenance trail so engineers can verify the recommendation before acting on it.

---

### High-Level Architecture

At the highest level, rootlyze sits between your production services and your engineering team. External services emit logs. rootlyze processes them, researches them, and returns actionable diagnostic output.

```
+---------------------------+
|   Production Services     |
|                           |
|  auth-service             |
|  api-gateway      +-------+------+
|  payment-service          |      |
|  worker-service           |      v
+---------------------------+  +---+-----------------------------+
                               |        rootlyze                 |
+---------------------------+  |                                 |
|   Knowledge Sources       |  |  Ingest                         |
|                           +->+    -> Process                   |
|  Internal docs / runbooks |  |       -> Enrich                 |
|  External web sources     |  |          -> Research (AI)       |
|  Community knowledge      |  |             -> Diagnose         |
+---------------------------+  |                -> Emit          |
                               +--------+--------+---------------+
                                        |
                                        v
                          +-------------+-------------+
                          |      Engineering Team      |
                          |                            |
                          |  Root cause report         |
                          |  Evidence list             |
                          |  Internal doc reference    |
                          |  External source citation  |
                          |  Verified fix steps        |
                          +----------------------------+
```

---

### Pipeline Design

The core of rootlyze is a linear, multi-stage processing pipeline. Log data enters at one end and diagnostic output exits at the other. The AI research engine sits alongside the diagnosis stage, providing external context that the internal documentation index cannot supply on its own.

```
External Services
      |
      v
+---------------------+
|  Log Ingestion      |   Accepts raw log data via HTTP or file agent.
|  Layer              |   Validates required fields. Forwards downstream.
+----------+----------+
           |
           |  RawLogEntry
           v
+---------------------+
|  Event Processing   |   Parses message text. Normalises formats.
|  Engine             |   Assigns metadata. Produces LogEvent structs.
+----------+----------+
           |
           |  LogEvent
           v
+---------------------+
|  Context Builder    |   Queries internal documentation index.
|                     |   Correlates related events. Builds DiagnosisContext.
+----------+----------+
           |
           |  DiagnosisContext
           v
+---------------------+     +-------------------------+
|  Diagnosis Engine   +---->+   AI Research Engine    |
|                     |     |                         |
|  Applies rules.     |<----+   Searches internal     |
|  Correlates events. |     |   docs and external     |
|  Identifies cause.  |     |   sources. Returns      |
|  Generates result.  |     |   ResearchResult with   |
+----------+----------+     |   citations.            |
           |                +-------------------------+
           |  DiagnosisResult
           v
+---------------------+
|  Output Layer       |   Formats result as JSON or plain text.
|                     |   Returns via HTTP or writes to store.
+---------------------+
           |
           v
    Diagnostic Report
```

---

### Component Responsibilities

#### Log Ingestion Layer

The ingestion layer is the system's entry point. It is intentionally thin.

Responsibilities:
- Accept incoming log data via HTTP POST or file-based agent forwarding
- Validate that required fields are present and non-empty
- Reject malformed requests with descriptive error responses
- Forward valid entries to the processing engine as `RawLogEntry` objects
- Emit metrics on ingestion rate, rejection rate, and queue depth

What it does not do:
- Parse log message content
- Make any decisions about log severity or significance
- Block on downstream processing

The ingestion layer operates asynchronously. When a log entry is accepted, it is placed on the processing queue and the HTTP response is returned immediately. Downstream processing happens independently of the ingestion acknowledgement.

---

#### Event Processing Engine

The processing engine transforms unstructured `RawLogEntry` objects into structured `LogEvent` objects.

Responsibilities:
- Parse message text for known error patterns, identifiers, and keywords
- Normalise timestamp formats across different service conventions
- Normalise severity labels (`WARN`, `WARNING`, `warn` all map to `SeverityLevel::WARN`)
- Extract service-specific metadata where patterns are known
- Discard unparseable entries gracefully with a logged warning
- Forward `LogEvent` objects to the context builder

The processing engine maintains a registry of known log patterns loaded from the rules configuration directory at startup. New patterns can be added without recompilation.

---

#### Context Builder

The context builder enriches `LogEvent` objects with information from outside the log itself.

Responsibilities:
- Query the internal documentation index using keywords and service identifiers from the `LogEvent`
- Retrieve relevant documentation segments ranked by relevance
- Correlate events sharing a `trace_id` into a unified trace view
- Assemble a `DiagnosisContext` containing the event, related events, and internal documentation passages
- Forward the `DiagnosisContext` to the diagnosis engine

The documentation index is an abstracted interface. The Phase 2 implementation uses keyword-based lookup. The Phase 4 implementation replaces this with a vector-based semantic retrieval system. The context builder does not change between these implementations.

---

#### Diagnosis Engine

The diagnosis engine reasons over structured context and coordinates with the AI research engine to produce a complete diagnosis.

Responsibilities:
- Load diagnosis rules from the configuration directory at startup
- Apply rules against `DiagnosisContext` objects to identify matching failure patterns
- Score multiple matching rules and select the highest-confidence hypothesis
- Issue a `ResearchRequest` to the AI research engine with the current hypothesis and context
- Receive a `ResearchResult` containing internal documentation matches and external source findings
- Merge internal and external evidence into a single `DiagnosisResult`
- Produce fix suggestions grounded in both sources, with citations for each step
- Forward results to the output layer

The diagnosis engine is designed so that the rule-based matching layer (Phase 3) can be replaced with an LLM-based reasoning layer (Phase 4) without changes to the research engine integration or the output layer.

---

### AI Research Engine

The AI research engine is a distinct subsystem that runs alongside the diagnosis engine. Its job is to answer the question: given this failure pattern, what do we know about it and how has it been resolved?

It operates in two modes simultaneously: internal research and external research.

#### Internal Research

The internal research component searches your indexed system documentation for passages relevant to the current failure.

```
DiagnosisContext
      |
      v
+---------------------+
|  Query Builder      |   Extracts search terms from the DiagnosisContext:
|                     |   service name, error keywords, component names,
|                     |   known error codes.
+----------+----------+
           |
           v
+---------------------+
|  Documentation      |   Searches indexed documentation segments.
|  Index              |   Phase 2: keyword-based lookup.
|                     |   Phase 4: vector-based semantic search.
+----------+----------+
           |
           v
+---------------------+
|  Passage Ranker     |   Scores retrieved passages by relevance.
|                     |   Returns top-N passages with source references.
+---------------------+
           |
           v
   InternalResearchResult
   {
     passages: [...],
     sources:  ["docs/services/auth-service.md#database-configuration"],
     confidence: 0.87
   }
```

#### External Research

The external research component uses a language model with web search access to find current, relevant information about the failure pattern from outside your codebase.

```
DiagnosisContext + InternalResearchResult
      |
      v
+---------------------+
|  Research Query     |   Constructs a targeted search prompt from the
|  Composer           |   failure pattern, service context, and any gaps
|                     |   identified in the internal research result.
+----------+----------+
           |
           v
+---------------------+
|  LLM Research       |   Sends the research prompt to a language model
|  Agent              |   with web search capability. The model searches
|                     |   for: known bugs, upstream issues, changelog
|                     |   entries, community-verified fixes, official
|                     |   documentation, and CVEs where relevant.
+----------+----------+
           |
           v
+---------------------+
|  Source Validator   |   Filters returned sources by domain authority
|                     |   and relevance score. Discards low-confidence
|                     |   or unverifiable results.
+----------+----------+
           |
           v
+---------------------+
|  Citation Builder   |   Structures findings into cited fix suggestions.
|                     |   Each suggestion includes: the recommendation,
|                     |   the source URL, and a confidence indicator.
+---------------------+
           |
           v
   ExternalResearchResult
   {
     findings: [
       {
         summary:    "pgBouncer recommended for connection pool management",
         source:     "postgresql.org/docs/current/pgbouncer.html",
         confidence: "high",
         type:       "official_documentation"
       },
       {
         summary:    "max_connections default of 10 insufficient above 50 RPS",
         source:     "github.com/postgres/postgres/issues/...",
         confidence: "medium",
         type:       "upstream_issue"
       }
     ]
   }
```

#### Research Result Merging

The diagnosis engine merges internal and external research results into a single coherent fix suggestion.

Merging rules:
- Internal documentation takes precedence over external sources for service-specific configuration values
- External sources take precedence for known upstream bugs, version-specific issues, and third-party library behaviour
- Where internal and external sources agree, the fix is marked as high-confidence
- Where they conflict, both recommendations are surfaced with their respective sources and a note flagging the discrepancy
- Every fix step in the final output includes a provenance label: `[internal]`, `[external]`, or `[internal + external]`

Example merged output:

```
Suggested Fix:

  1. [internal + external] Increase max_connections in auth-service
     database configuration.
     Internal ref: docs/services/auth-service.md#database-configuration
     External ref: postgresql.org/docs — confirmed default of 10 is
     insufficient for concurrent workloads

  2. [external] Introduce pgBouncer as a connection pooling layer.
     Source: postgresql.org/docs/current/pgbouncer.html (high confidence)

  3. [internal] Add connection pool monitoring.
     Ref: docs/observability/database-metrics.md#pool-saturation
```

#### Research Engine Configuration

The research engine exposes the following configuration options:

| Key                          | Default    | Description                                      |
|------------------------------|------------|--------------------------------------------------|
| `research.internal.enabled`  | `true`     | Enable internal documentation search             |
| `research.external.enabled`  | `true`     | Enable external web research via LLM             |
| `research.external.model`    | (required) | LLM model identifier for external research       |
| `research.external.api_key`  | (required) | API key for the LLM provider                     |
| `research.source.min_confidence` | `medium` | Minimum confidence threshold for external sources |
| `research.source.allowed_domains` | `[]`  | Optional allowlist of trusted external domains   |
| `research.source.blocked_domains` | `[]`  | Optional blocklist of excluded domains           |
| `research.timeout_ms`        | `5000`     | Maximum time to wait for external research       |

External research is non-blocking. If the research engine does not return within `research.timeout_ms`, the diagnosis proceeds with internal research results only and the output notes that external research was unavailable.

---

### Inter-Component Communication

Components communicate exclusively via structs passed through a queue-based pipeline. There is no shared mutable state between components.

```cpp
template<typename T>
class PipelineQueue {
public:
    void push(T item);
    std::optional<T> pop();
    bool empty() const;
private:
    std::queue<T>            queue_;
    std::mutex               mutex_;
    std::condition_variable  cv_;
};

class EventProcessingEngine {
public:
    void run(PipelineQueue<RawLogEntry>& input,
             PipelineQueue<LogEvent>&    output);
};
```

This design means:
- Stages can run on separate threads without data races
- A slow diagnosis stage does not block the ingestion stage
- Stages can be tested in isolation by feeding mock queues
- The pipeline can be paused or drained at any stage without data loss

---

### Concurrency Model

rootlyze uses a thread-per-stage model. Each pipeline stage runs on a dedicated worker thread and communicates with adjacent stages via thread-safe queues. The AI research engine runs on a separate thread pool to prevent network I/O from stalling the main pipeline.

```
Thread 1:    HTTP Server             (ingestion layer)
Thread 2:    Event Processor         (processing engine)
Thread 3:    Context Builder         (context builder)
Thread 4:    Diagnosis Engine        (diagnosis engine)
Thread 5-N:  Research Workers        (AI research engine thread pool)
Thread N+1:  Output Emitter          (output layer)
```

The research worker pool is sized independently from the main pipeline. A long-running external research request occupies one research worker thread but does not block the diagnosis engine from processing other events. Results are returned to the diagnosis engine via a future-based callback.

**Back-pressure:** If the diagnosis engine is slow, the context builder queue fills up. If the context builder queue fills up, the processing engine slows down. This creates natural back-pressure through the pipeline without explicit flow control.

**Research timeout isolation:** If all research workers are occupied, new research requests are queued. If the queue exceeds a configurable depth, external research is skipped for that diagnosis and the output notes the degradation.

---

### Memory Model

rootlyze is designed to minimise heap allocation in the hot path.

Key design decisions:
- `RawLogEntry` and `LogEvent` structs use move semantics throughout the pipeline. Entries are moved, not copied, as they pass between stages.
- The documentation index is loaded into memory at startup and held read-only for the lifetime of the process. No locks are required for documentation lookups.
- The diagnosis rules are held as a sorted vector with early exit on high-confidence match.
- `DiagnosisContext` objects hold references to documentation segments rather than copies, reducing allocation per context build.
- `ResearchResult` objects are heap-allocated and passed by unique pointer from the research engine to the diagnosis engine to avoid copying large result payloads.

---

### Error Handling Strategy

rootlyze uses a layered error handling approach. Errors are classified by severity and handled accordingly.

**Recoverable errors (per-entry):** A malformed log entry, a documentation lookup failure, or a rule match with low confidence. These are logged as warnings, the entry is either discarded or passed through with reduced enrichment, and the pipeline continues.

**Research degradation:** If the external research engine is unavailable, times out, or returns no usable results, the diagnosis proceeds with internal research only. The output explicitly notes that external research was unavailable so engineers are not misled by an incomplete result.

**Stage errors (transient):** A queue timeout, a temporary file read failure, or an intermittent LLM API error. These are retried with exponential backoff up to a configured limit.

**Fatal errors (startup):** A missing configuration file, an unreadable rules directory, a missing LLM API key when external research is enabled, or a port binding failure. These terminate the process immediately with a clear error message. rootlyze does not start in a degraded state.

---

### Extensibility Points

rootlyze is designed with five explicit extensibility points.

**1. Ingestion adapters**
The ingestion layer accepts an `IngestorInterface` that can be implemented for any source: HTTP, file tailing, message queue consumer, or OS log integration.

**2. Pattern registry**
The processing engine loads patterns from a configuration directory at startup. New patterns can be added by writing a pattern definition file. No recompilation required.

**3. Documentation index backend**
The context builder uses a `DocumentIndexInterface`. Phase 2 is keyword-based. Phase 4 is vector-based. Swapping backends requires implementing the interface.

**4. Diagnosis engine backend**
The diagnosis engine uses a `DiagnosisEngineInterface`. Phase 3 uses rule-based matching. Phase 4 uses an LLM-based reasoning backend.

**5. Research engine backend**
The AI research engine uses a `ResearchEngineInterface`. The default implementation uses a hosted LLM with web search. This can be replaced with a self-hosted model, a different LLM provider, or a purely local research implementation for air-gapped environments.

---

## Getting Started

### Prerequisites

- C++17 compatible compiler (GCC 9+, Clang 10+, or MSVC 2019+)
- CMake 3.15 or later
- Git
- An LLM API key (required for external research — see [Configuration](#configuration))

### Build

```bash
git clone https://github.com/localopensource/rootlyze.git
cd rootlyze
mkdir build && cd build
cmake ..
make
```

### Run

```bash
./rootlyze --config rootlyze.conf
```

> HTTP ingestion and the file-based agent are in active development (Phase 1). External AI research is a Phase 2 and Phase 4 feature. See [Roadmap](#roadmap) for timelines.

---

## External Usage Guide

This section is for engineers integrating their services with rootlyze.

### What you need to do

Nothing complex. Your services only need to emit logs. rootlyze handles all parsing, normalisation, and research.

### Sending logs via HTTP

*(Available: Phase 1)*

```http
POST /log
Content-Type: application/json

{
  "service":   "auth-service",
  "level":     "error",
  "message":   "DB connection timeout after 30s",
  "timestamp": "2025-06-01T14:32:00Z"
}
```

**Required fields:**

| Field       | Type   | Description                           |
|-------------|--------|---------------------------------------|
| `service`   | string | Name of the emitting service          |
| `level`     | string | `debug` `info` `warn` `error` `fatal` |
| `message`   | string | Log message text                      |
| `timestamp` | string | ISO 8601 timestamp                    |

**Optional fields:**

| Field      | Type   | Description                              |
|------------|--------|------------------------------------------|
| `trace_id` | string | Distributed trace ID for correlation     |
| `host`     | string | Hostname or container ID                 |
| `metadata` | object | Arbitrary key-value pairs for enrichment |

**Response: event queued**

```json
{ "status": "accepted", "event_id": "evt_abc123" }
```

**Response: diagnosis available**

```json
{
  "status":    "diagnosed",
  "event_id":  "evt_abc123",
  "diagnosis": {
    "root_cause": "Connection pool exhaustion in auth-service",
    "evidence":   ["..."],
    "fix_steps": [
      {
        "step":       "Increase max_connections in database configuration",
        "provenance": "internal + external",
        "sources":    ["docs/services/auth-service.md", "postgresql.org/docs"]
      },
      {
        "step":       "Introduce pgBouncer as a connection pooling layer",
        "provenance": "external",
        "sources":    ["postgresql.org/docs/current/pgbouncer.html"]
      }
    ]
  }
}
```

### File-based agent

*(Available: Phase 1)*

```bash
rootlyze-agent --file /var/log/auth-service.log \
               --service auth-service \
               --target http://rootlyze:8080
```

---

## Internal Documentation

### Repository structure

```
rootlyze/
├── src/
│   ├── ingestion/       Log Ingestion Layer
│   ├── processing/      Event Processing Engine
│   ├── context/         Context Builder
│   ├── diagnosis/       Diagnosis Engine
│   ├── research/        AI Research Engine
│   └── output/          Result formatter and emitter
├── include/             Public headers
├── tests/               Unit and integration tests
├── docs/                Extended documentation
├── examples/            Sample log input and expected output
└── CMakeLists.txt
```

### Key data structures

```cpp
struct RawLogEntry {
    std::string service;
    std::string level;
    std::string message;
    std::string timestamp;
    std::optional<std::string> trace_id;
    std::optional<std::string> host;
};

struct LogEvent {
    std::string          service;
    SeverityLevel        level;
    std::string          message;
    std::chrono::time_point<std::chrono::system_clock> timestamp;
    std::vector<std::string> parsed_tokens;
    std::optional<std::string> trace_id;
};

struct DiagnosisContext {
    LogEvent                    primary_event;
    std::vector<LogEvent>       related_events;
    std::vector<std::string>    documentation_segments;
    std::optional<std::string>  trace_id;
};

struct ResearchFinding {
    std::string summary;
    std::string source_url;
    std::string confidence;   // "high", "medium", "low"
    std::string type;         // "official_documentation", "upstream_issue",
                              // "community_fix", "changelog", "cve"
};

struct ResearchResult {
    std::vector<std::string>     internal_passages;
    std::vector<std::string>     internal_sources;
    std::vector<ResearchFinding> external_findings;
    bool                         external_research_available;
};

struct DiagnosisResult {
    std::string              root_cause;
    std::vector<std::string> evidence;
    std::vector<FixStep>     fix_steps;
};

struct FixStep {
    std::string              description;
    std::string              provenance;   // "internal", "external",
                                           // "internal + external"
    std::vector<std::string> sources;
};
```

### Data flow (step by step)

```
1.  External service emits a log entry
2.  Ingestion layer validates and accepts the entry as RawLogEntry
3.  Processing engine normalises it into a LogEvent
4.  Context builder enriches the LogEvent into a DiagnosisContext
5.  Diagnosis engine forms an initial hypothesis from the DiagnosisContext
6.  Diagnosis engine issues a ResearchRequest to the AI research engine
7.  Research engine queries the internal documentation index
8.  Research engine sends a research prompt to the LLM with web search
9.  Research engine validates and structures findings into a ResearchResult
10. Diagnosis engine merges DiagnosisContext and ResearchResult
11. Diagnosis engine produces a DiagnosisResult with cited fix steps
12. Output layer formats and emits the DiagnosisResult
```

---

## Configuration

| Key                               | Default    | Description                                      |
|-----------------------------------|------------|--------------------------------------------------|
| `server.port`                     | `8080`     | Ingestion HTTP server port                       |
| `server.bind`                     | `0.0.0.0`  | Bind address                                     |
| `storage.path`                    | `./data`   | Local log and event storage path                 |
| `docs.index_path`                 | `./docs`   | Path to documentation index                      |
| `diagnosis.rules`                 | `./rules`  | Path to diagnosis rule definitions               |
| `research.internal.enabled`       | `true`     | Enable internal documentation search             |
| `research.external.enabled`       | `true`     | Enable external web research via LLM             |
| `research.external.model`         | (required) | LLM model identifier                             |
| `research.external.api_key`       | (required) | API key for the LLM provider                     |
| `research.source.min_confidence`  | `medium`   | Minimum confidence threshold for external sources|
| `research.source.allowed_domains` | `[]`       | Optional allowlist of trusted external domains   |
| `research.source.blocked_domains` | `[]`       | Optional blocklist of excluded domains           |
| `research.timeout_ms`             | `5000`     | Maximum wait time for external research          |

---

## Roadmap

### Phase 1: Core pipeline *(in progress)*
- [ ] HTTP log ingestion server
- [ ] `RawLogEntry` and `LogEvent` data structures
- [ ] Event parsing and normalisation engine
- [ ] Local log and event storage

### Phase 2: Context enrichment and internal research
- [ ] Documentation ingestion and indexing
- [ ] Keyword-based internal documentation search
- [ ] `DiagnosisContext` assembly
- [ ] Internal research result integration

### Phase 3: Diagnosis
- [ ] Rule-based diagnosis engine
- [ ] Cross-service log correlation
- [ ] `DiagnosisResult` output with evidence and fix suggestions
- [ ] Fix step provenance tracking

### Phase 4: AI intelligence layer
- [ ] LLM-based diagnosis reasoning engine
- [ ] External web research via LLM with web search
- [ ] Vector-based semantic documentation retrieval
- [ ] Source validation and citation pipeline
- [ ] Real-time streaming pipeline
- [ ] Distributed ingestion support

---

## Contributing

Contributions are welcome. Before submitting a pull request:

1. Open an issue describing the change and its motivation
2. Ensure the proposal aligns with the project architecture
3. All code must build cleanly against the CMake configuration
4. Tests are required for new components

---

## Licence

MIT. See [LICENSE](LICENSE) for details.

---

## Author

**localopensource** — [github.com/localopensource](https://github.com/localopensource)

---

<div align="center">
<sub>rootlyze — from log noise to root cause.</sub>
</div>
