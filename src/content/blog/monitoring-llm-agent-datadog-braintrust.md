---
title: "How We Monitor Our Agentic Platform, Plus Datadog vs. Braintrust"
description: "A practical observability stack for LLM agents: what Datadog's classic monitoring reveals, what it misses, and where Braintrust adds value."
pubDate: 2026-08-07
heroImage: "../../assets/llm-agent-observability-hero.png"
---

I've set up service monitoring several times before, across different providers. This was the first time I built the monitoring stack from scratch—and the first time I had to monitor a fleet of agents on top of the underlying services and platform.

The agent is what made it different. A service fails loudly: it 500s, latency spikes, a queue backs up. An LLM agent "fails" quietly: it returns `200 OK` in 30 seconds while, inside the turn, some tool calls failed and some steps crawled—which can be hard to tell from the outside.

That quiet-failure gap is why the classic stack alone could not answer every question. I used Datadog because I know it, then looked at how Braintrust handles the part classic monitoring misses. Each component below shows both: what we built in Datadog, and how the corresponding capability looks in Braintrust.

One more reason for me to write it down: we can't tell Claude Code "set up monitoring for me" and expect a finished system. The engineer still has to decide what's worth watching, and the AI missed key nuances—including trace-to-log linking, as you'll see. What Codex and Claude Code did provide was leverage: a monitoring setup that had previously taken me more than three weeks took about three days this time.

## Table of Contents

- [The components, and what each one catches](#the-components-and-what-each-one-catches)
  - [Synthetics: is my service up?](#1-synthetics-is-my-service-up)
  - [Logs: what happened?](#2-logs-what-happened)
  - [APM: where did the time go?](#3-apm-where-did-the-time-go)
  - [Connecting logs and traces](#connecting-logs-and-traces)
  - [Infrastructure metrics: the layer underneath](#4-infrastructure-metrics-the-layer-underneath)
  - [Monitors: what wakes you?](#5-monitors-what-wakes-you)
  - [Dashboards: what do you look at?](#6-dashboards-what-do-you-look-at)
- [What you end up with](#what-you-end-up-with)
- [What the classic stack can't tell you](#what-the-classic-stack-cant-tell-you)
- [What Braintrust adds](#what-braintrust-adds)
- [Where each one fits](#where-each-one-fits)
- [Implementation note: Azure log-trace correlation](#implementation-note-azure-log-trace-correlation)

---

## The components, and what each one catches

Datadog is several products sharing a bill—and its catalog is far bigger than this table, as is Braintrust's. This post compares the classic Datadog stack we used—APM, Logs, Infrastructure Monitoring, and Synthetics—with the closest Braintrust capability, when there is one. Datadog also offers [**Agent Observability**](https://docs.datadoghq.com/llm_observability/), a separate LLM-native product that captures model inputs and outputs and includes evaluations, datasets, and experiments. I haven't used it, so it isn't the Datadog product under test here.

Both platforms are moving quickly, so the product details in this post are current as of **August 2026**.

| Component                  | What it is                                                                  | Answers                 | Catches                                    |
| -------------------------- | --------------------------------------------------------------------------- | ----------------------- | ------------------------------------------ |
| **Synthetics**             | A prober outside your network hitting an endpoint on a schedule             | Is it up, from outside? | Total outage, expired certs, broken DNS    |
| **Logs**                   | Your application's own output, parsed into structured fields and indexed    | What happened?          | Errors, job failures, agent behavior       |
| **APM / tracing**          | Spans stitched into a timeline per request, shipped by a collector          | Where did the time go?  | Slow model calls, slow queries, nesting    |
| **Infrastructure metrics** | CPU, memory, restarts, and database counters pulled from your cloud provider | Is the box healthy?     | OOM kills, restarts, connection exhaustion |
| **Monitors**               | Saved queries with a threshold and a notification target                    | What should wake me?    | Anything above, once it crosses a line     |
| **Dashboards**             | Saved queries with a chart instead of an alert                              | What do I look at?      | Trends nobody should be paged for          |

The architecture looks like this:

[![An LLM agent service sends logs and traces to Datadog, while an external probe checks availability and cloud integrations provide infrastructure metrics. Datadog monitors and dashboards use all four data sources.](/llm-agent-observability-stack.svg)](/llm-agent-observability-stack.svg)

_The classic stack observes the service from four directions. Braintrust adds a separate, LLM-native view of the agent's inputs, outputs, and quality. [Open the diagram at full size.](/llm-agent-observability-stack.svg)_

---

### 1. Synthetics: is my service up?

The smallest component, and the most important one: an external prober hits a health endpoint on a schedule from outside your network.

This is the only component in this stack that directly tests end-to-end availability from outside. Without it, an app that stops answering may produce no application errors or slow traces — just silence. Infrastructure or no-data monitors may still reveal the failure, but they don't prove that a user can reach the endpoint.

Terraform to set it up:

```hcl
resource "datadog_synthetics_test" "health" {
  type    = "api"
  subtype = "http"
  status  = "live"

  request_definition {
    method = "GET"
    url    = "https://api.example.com/health"
  }

  assertion {
    type     = "statusCode"
    operator = "is"
    target   = "200"
  }

  locations = ["aws:us-west-2", "aws:eu-central-1"]

  options_list {
    tick_every          = 300   # seconds
    min_location_failed = 2     # don't page on one bad region
  }
}
```

**Include the database in the end-to-end health check.** Have the public `/health` endpoint run a lightweight query against a table the application depends on. That tests the path from network → frontend → backend → database, rather than merely proving that the HTTP process can respond. Cache whether the query succeeded for a short time so frequent health checks do not create continuous database traffic. Keep this separate from the container's liveness check: a database outage should alert you, not restart every healthy application instance.

In FastAPI, the shape is roughly:

```python
from fastapi import HTTPException

@app.get("/health")
async def health():
    query = "SELECT 1 FROM conversations LIMIT 1"
    if not await cached_query_succeeds(query, ttl_seconds=30):
        raise HTTPException(status_code=503)
    return {"ok": True}
```


**Braintrust:** no documented equivalent for probing an application's availability.

---

### 2. Logs: what happened?

**A structured record per turn**, with fields you can aggregate:

```python
logger.info(
    "agent turn finished",
    extra={
        "conversation_id": conversation_id,
        "outcome": outcome,              # completed | failed | cancelled
        "duration_ms": duration_ms,
        "tool_calls": len(tool_calls),
        "tools_used": sorted({t.name for t in tool_calls}),
        "model_calls": model_call_count,
        "hit_step_limit": hit_step_limit,
    },
)
```

The difference matters at query time: a sentence like `"turn took 4200ms"` cannot be aggregated as a number until it is parsed, but a field like `duration_ms: 4200` can be averaged, used for percentiles, and alerted on directly.

Getting those fields queryable in Datadog is a two-step job, and both steps live in a *pipeline* — an ordered list of processors applied to every matching log.

**First, get the line parsed.** A bare JSON line is parsed into attributes automatically. But check what actually lands, because platforms like to wrap stdout: ours — Azure Container Apps — delivers each line inside an envelope, with the JSON as a *string* in `properties.Log` — and Datadog doesn't look inside strings, so every log arrives as one opaque blob. The fix is the pipeline's first processor: a grok parser whose single rule `%{data::json}` parses that string and lifts every key to a top-level attribute — which also means it never needs updating when the formatter gains a field.

**Then verify the fields Datadog treats as special** — `status`, `message`, `timestamp`, `service`, `trace_id`. Datadog automatically recognizes common JSON names such as `level`, `severity`, `status`, `service` and `dd.trace_id`, but a custom envelope or parser can keep them from landing in the expected place. An explicit remapper is useful when your source uses a non-standard name or when you want the mapping to be unambiguous. The timestamp is especially important: if ours is not remapped, Datadog uses the log's *arrival* time, and everything in a batch can share one timestamp minutes late — we measured seven. Ordering inside a request becomes meaningless, and a trace's Logs tab can find nothing because it searches the trace's own time window.

An example, trimmed to the essentials:

```hcl
resource "datadog_logs_custom_pipeline" "console_logs" {
  name       = "Container App console logs"
  is_enabled = true

  filter { query = "source:azure.app @category:ContainerAppConsoleLogs" }

  processor {
    grok_parser {
      name   = "parse the JSON payload out of properties.Log"
      source = "properties.Log"
      grok {
        support_rules = ""
        match_rules   = "app_json %%{data::json}"
      }
    }
  }

  processor {
    status_remapper { sources = ["level"] }        # level -> status
  }

  processor {
    date_remapper { sources = ["timestamp"] }      # our time, not arrival time
  }
}
```

**Conversation ID on every line.** This one is specific to agents. When a user reports a bad answer, what they hand you is a conversation — and the turn that actually went wrong may not be the one they're complaining about. Finding it means pulling every log line for that conversation — so every line has to carry the identifier.

The naive approach is to pass the conversation ID into every log call, which means touching every call site and still missing the ones inside libraries. Instead, put it in a context variable and let a logging filter copy it onto every record.

It takes three pieces. First, somewhere to hold the value, and a filter that reads it:

```python
from contextvars import ContextVar
import logging

_context: ContextVar[dict] = ContextVar("log_context", default={})

class ContextFilter(logging.Filter):
    def filter(self, record):
        for key, value in _context.get().items():
            if not hasattr(record, key):
                setattr(record, key, value)
        return True          # never drop the record, only decorate it
```

Second, attach the filter once, where you configure logging:

```python
handler = logging.StreamHandler()
handler.addFilter(ContextFilter())
handler.setFormatter(JsonFormatter())     # emits record attributes as JSON fields
logging.getLogger().addHandler(handler)
```

Third, set the value once at the start of a turn — in middleware, or wherever you first know it — and reset it when the turn ends:

```python
token = _context.set({"conversation_id": conversation_id, "user_id": user_id})
try:
    await run_turn()
finally:
    _context.reset(token)
```

During the turn, every record processed by this handler picks up those fields — including records from library code that propagates to it. A `ContextVar` is the right container because each async task gets its own context, so concurrent requests don't leak identifiers into each other's logs. Resetting it prevents the same task from carrying an old identifier into later work.

The one requirement is that your formatter actually writes record attributes into the output. If it only prints the message string, the fields are set and then thrown away.

**Braintrust: different model.** Its [Logs page](https://www.braintrust.dev/docs/observe/view-logs) is a trace store, not a general stdout stream. You instrument calls and Braintrust records their input, output, exceptions and timing; it does not collect a `logging.warning(...)` or library log line merely because it occurred inside a traced function. The instrumentation example belongs with tracing, next.

---

### 3. APM: where did the time go?

Logs tell you a turn took thirty seconds. Tracing gives you the clearest request-level answer to *which part*.

**The architecture matters here.** In our deployment, Datadog's tracer doesn't send spans to Datadog directly; it sends them to a collector Datadog calls the *Agent* — an unfortunate name collision in a post about LLM agents, so I'll call it the collector throughout. On VMs it is commonly a host daemon. In our container setup it runs as a **sidecar**: a second container alongside the app, sharing a network namespace, so the app reaches it on localhost:

```
┌─────────────────────────────┐
│  Container App              │
│                             │
│   ┌────────┐   ┌─────────┐  │
│   │  app   │──▶│collector│──┼──▶ Datadog
│   │        │   │ sidecar │  │
│   └────────┘   └─────────┘  │
│      localhost:8126         │
└─────────────────────────────┘
```

**In code, auto-instrumentation creates most of the trace.** `ddtrace` instruments supported web frameworks, database drivers, and model SDKs. The one span worth adding yourself wraps each tool call:

```python
from ddtrace import tracer

with tracer.trace("agent.tool", resource=tool_name) as span:
    span.set_tag("tool.name", tool_name)
    result = await execute_tool(tool_name, params)
```

**Braintrust: supported, with less setup.** No collector, no sidecar, no port — the SDK ships spans directly, and [Python auto-instrumentation](https://www.braintrust.dev/docs/instrument/trace-llm-calls) patches supported AI libraries:

```python
import braintrust
from braintrust import traced

braintrust.auto_instrument()              # patches supported AI libraries
braintrust.init_logger(project="agent")  # destination for spans

@traced
async def run_turn(question: str) -> str:
    return await agent.run(question)

@traced
async def call_tool(name: str, params: dict):
    return await execute_tool(name, params)
```

Despite the name, `init_logger` has nothing to do with Python's `logging` module — it creates Braintrust's destination for spans. `@traced` can wrap ordinary application logic as well as LLM code ([docs](https://www.braintrust.dev/docs/instrument/trace-application-logic)).

`auto_instrument()` is the closest analogue to `ddtrace`'s auto-instrumentation, with a different target: model SDKs and agent frameworks (OpenAI, Anthropic, LangChain, CrewAI and the like) rather than web frameworks and database drivers. If you prefer to be explicit, `wrap_openai(OpenAI())` does the same for one client.

Which you want depends on the question. Datadog gives us model time beside SQL time automatically; doing that in Braintrust requires manual or OpenTelemetry instrumentation around the non-AI work. Braintrust gives us the model input and output as first-class trace data.

---

### Connecting logs and traces

This isn't a seventh product — it's the wiring between the last two, and the step most worth getting right. Wiring traces to logs is extremely helpful: from a slow span in the waterfall, you can jump directly to the logs emitted during that operation instead of trying to match them by timestamp.

**Turn on injection.** Two ways to run under the tracer, which patches the logging library so every record picks up the active span:

```bash
ddtrace-run python app.py       # run under the tracer, or
```
```python
import ddtrace.auto             # first import in your entrypoint
```

On `ddtrace < 3.11` you additionally need `DD_LOGS_INJECTION=true` in the environment; on newer versions injection is on by default once the tracer is active. The flag on its own does nothing — it configures the tracer, it doesn't start one.

The full detail is in [Datadog's docs](https://docs.datadoghq.com/tracing/other_telemetry/connect_logs_and_traces/python/).

What lands on each record is five fields: `dd.env`, `dd.service`, `dd.version`, `dd.trace_id`, `dd.span_id`.

**If you log plain text**, that's the whole job — add them to your format string:

```python
FORMAT = ('%(asctime)s %(levelname)s [%(name)s] '
          '[dd.service=%(dd.service)s dd.env=%(dd.env)s dd.version=%(dd.version)s '
          'dd.trace_id=%(dd.trace_id)s dd.span_id=%(dd.span_id)s] - %(message)s')
```

That works because `dd.trace_id` is a single key whose *name* contains a dot — not a nested object.

**If you log JSON, keep the IDs as strings and check their final parsed shape.** Injection sets values as record attributes literally named `dd.trace_id`, `dd.service` and so on. A formatter that copies record attributes into a JSON object therefore produces:

```json
{ "dd.trace_id": "8763124...", "dd.span_id": "43891..." }
```

Datadog documents that top-level dotted shape as valid and automatically extracts it from ordinary JSON logs. For direct correlation, inspect a processed log and verify that Datadog shows the string value as its reserved `trace_id`. Our Azure log path needed an extra mapping step; the failure mode and formatter are in the implementation note at the end.

**Braintrust: no equivalent correlation step for its trace data.** Braintrust stores the function input, output and timing together on its [Logs page](https://www.braintrust.dev/docs/observe/view-logs). Your separate stdout logs remain separate, but the AI interaction itself does not require Datadog-style log-to-APM wiring.

---

### 4. Infrastructure metrics: the layer underneath

Everything above reasons about your application from its own output. This is the layer beneath it: CPU, memory, restarts, database connections, deadlocks.

It matters more than it sounds for agents. A worker being OOM-killed shows up in your logs as a task that stopped mid-flight, if it shows up at all. Metrics may show memory climbing toward a limit *before* the kill, which is when you can act.

**Braintrust: outside its scope for your application's infrastructure.** Braintrust does expose an [infrastructure dashboard for its self-hosted data plane](https://www.braintrust.dev/docs/admin/self-hosting#infra-dashboard); that monitors Braintrust's own deployment, not the containers, hosts or database running your agent.

---

### 5. Monitors: what wakes you?

The rule I'd apply: **every monitor should map to a concrete failure mode and a clear response**, ideally one validated by a real incident or failure test. Guessed thresholds get muted, and muted monitors are worse than none because they read as coverage.

```hcl
resource "datadog_monitor" "agent_turns_failing" {
  name    = "agent turns failing or exhausting their budget"
  type    = "log alert"
  query   = "logs(\"@outcome:failed OR @hit_step_limit:true\").index(\"*\").rollup(\"count\").by(\"@tenant\").last(\"8h\") > 10"
  message = "Agent turns are failing on {{@tenant.name}}. @team-handle"

  monitor_thresholds { critical = 10 }
}
```

**Braintrust: supported, over different data.** [Log alerts](https://www.braintrust.dev/docs/observe/alerts) use a SQL filter over your traces. Two examples — ordinary errors and quality scores:

```sql
-- quality, not availability
scores.factuality < 0.8 AND metadata.environment = 'production'

-- and the ordinary kind too
error IS NOT NULL AND metadata.environment = 'production'
```

An alert can notify a Slack channel or hit a webhook. Braintrust evaluates logs in batches, so alerts aren't instantaneous.

---

### 6. Dashboards: what do you look at?

Monitors are for things that need a response. Dashboards are for trends nobody should be paged about but someone should see: latency over time, tool durations, failure rates.

**Braintrust: supported, with scores as first-class data.** [Custom dashboards](https://www.braintrust.dev/docs/observe/dashboards) chart request counts, latency, token usage, cost and scores over time. The first few overlap with what you'd build in classic Datadog. You can send custom quality scores into Datadog yourself, and Datadog Agent Observability supports them natively; Braintrust's distinction is that scores are built into the same tracing and evaluation workflow.

---

## What you end up with

Assembled, the six components answer a specific set of questions: **is it up, what happened, where did the time go, is the machine healthy, what should wake me, what's the trend.**

For an agent, the most valuable of those turned out to be the third. Here's one turn, thirty-three seconds end to end:

| span              | time            |
| ----------------- | --------------- |
| HTTP request      | **33s**         |
| ↳ model calls ×5  | **27.8s — 84%** |
| ↳ tool calls ×4   | **62ms total**  |
| ↳ every SQL query | under 25ms      |

That inverted where I'd have spent effort. The instinct is to optimize tool code and queries—worth milliseconds here. **In this trace, the lever was how many times the agent called the model**, which made this a prompt and control-flow problem rather than a low-level code optimization problem. If your agent feels slow, measure the split before optimizing anything.

## What the classic stack can't tell you

All of that describes the *shape* of a request. None of it describes the *content*.

Our classic APM trace knows how many times the model was called and how long those calls took. It doesn't know what was asked, what came back, or whether the answer was right — and for an agent that last one is the question. Datadog Agent Observability closes that gap too; as noted at the start, it isn't the stack we implemented and tested here.

## What Braintrust adds

The Braintrust capabilities that stood out to me follow from one design choice: it stores prompts and completions alongside timing data. Four useful workflows follow from that.

**[Scorers, offline and online.](https://www.braintrust.dev/docs/evaluate/write-scorers)** A scorer can be custom code, an LLM-as-a-judge, or a built-in Autoeval. The same kind of quality check can run against a fixture suite and against live traffic:

```python
from braintrust import Eval, init_dataset
from autoevals import Factuality

Eval(
    "agent",
    experiment_name="prompt-v3",
    data=init_dataset(
        project="agent",
        name="regressions",
    ),
    task=lambda input: run_turn(input),
    scores=[Factuality],
    metadata={"model": "gpt-5-mini"},
)
```

For production, select a built-in scorer or push a custom one, then create an [online scoring automation](https://www.braintrust.dev/docs/evaluate/score-online) in the UI or API. It applies the scorer asynchronously as traces arrive, either to all matching traffic or to the percentage you choose based on cost and risk.

**[Experiments as immutable records.](https://www.braintrust.dev/docs/evaluate/run-evaluations)** Each run is a fixed snapshot you can compare against another, so "did prompt v3 beat v2" has an answer rather than an anecdote — and it can run in CI to catch regressions before release.

**[Production traces become test cases.](https://www.braintrust.dev/docs/annotate/datasets)** The loop the classic monitoring stack has no equivalent for: filter live logs for interesting ones, pull them into a dataset, and they become reusable fixtures for future experiments. A real failure turns into a regression test rather than a note in a ticket.

**[Prompts extractable to a playground.](https://www.braintrust.dev/docs/observe/view-logs#iterate-in-playgrounds)** Take the exact prompt and inputs from a bad production trace, iterate on them interactively, and compare variations with your datasets and scorers.

## Where each one fits

**The classic Datadog stack's strength is everything around the model**, where many operational failures happen: a worker killed for memory, a sync job that stops writing while the agent keeps answering from stale data, or a deploy that quietly stops shipping. In each case the agent is fine and its surroundings aren't. In the stack we used, the blind spot is the content: it measures the model call without storing what was in it.

**Braintrust's strength is the content**: was the answer right, and is that improving. Its blind spot is the machine: it only sees what your application sends, and it won't tell you the application is gone.

These are not exact substitutes, so the choice starts with the question you cannot answer today. If the gap is availability and infrastructure, start with the classic Datadog stack. If the gap is answer quality and prompt regressions, use Braintrust or an LLM-native product such as Datadog Agent Observability. If you own both problems, pair operational monitoring with an LLM evaluation workflow instead of forcing one tool to do both. We started with Datadog because availability failures take us offline, and we're building the content and evaluation half ourselves.

---

## Implementation note: Azure log-trace correlation

Azure places our JSON inside `properties.Log`, where a custom grok processor extracts it. That is where Claude Code's configuration failed us. The pipeline applied cleanly, but the ID did not end up as Datadog's reserved trace attribute. Nothing looked broken; the trace simply had no related logs.

The reliable debugging method is to inspect a processed log and verify three things: the trace ID exists, it is a string, and Datadog shows it as the reserved `trace_id`. If a custom pipeline still does not recognize the dotted field, rename it deliberately in the formatter instead of relying on another implicit transformation. `get_log_correlation_context()` returns the five dotted Datadog keys, so the rename has to be explicit:

```python
from datetime import datetime, timezone
import json
import logging

from ddtrace import tracer


_STANDARD_FIELDS = frozenset(
    logging.makeLogRecord({}).__dict__
)


class JsonFormatter(logging.Formatter):
    def format(self, record):
        correlation = tracer.get_log_correlation_context()

        # Keep application fields added through extra={...} or ContextFilter.
        payload = {
            key: value
            for key, value in record.__dict__.items()
            if (
                key not in _STANDARD_FIELDS
                and not key.startswith("dd.")
            )
        }
        payload.update(
            {
                "timestamp": datetime.fromtimestamp(
                    record.created, tz=timezone.utc
                ).isoformat(),
                "level": record.levelname,
                "message": record.getMessage(),
                "trace_id": str(
                    correlation["dd.trace_id"]
                ),
                "span_id": str(
                    correlation["dd.span_id"]
                ),
                "service": correlation["dd.service"],
                "env": correlation["dd.env"],
                "version": correlation["dd.version"],
            }
        )
        if record.exc_info:
            payload["exception"] = self.formatException(
                record.exc_info
            )

        return json.dumps(payload, default=str)
```

Because this formatter renames `dd.trace_id` to `trace_id`, our custom pipeline's trace remapper must point to that renamed key:

```hcl
processor {
  trace_id_remapper { sources = ["trace_id"] }
}
```

The broader lesson is to verify the processed log, not just the application output or Terraform plan. Correlation is working only when Datadog recognizes the final field as its reserved trace ID.
