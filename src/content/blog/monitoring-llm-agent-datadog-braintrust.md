---
title: "How We Monitor Our Agentic Platform, Plus Datadog vs. Braintrust"
description: "How we built monitoring for our agentic platform with Datadog, and what we learned comparing it with Braintrust along the way."
pubDate: 2026-08-07
heroImage: "../../assets/llm-agent-observability-hero.png"
---

I've set up service monitoring several times before, across different providers. This was the first time I built the monitoring stack from scratch—and the first time I had to monitor a fleet of agents on top of the underlying services and platform.

The agent is what made it different. A service fails loudly: it 500s, latency spikes, a queue backs up. An LLM agent "fails" quietly: it returns `200 OK` in 30 seconds while, inside the turn, some tool calls failed and some steps crawled—which can be hard to tell from the outside.

We used Datadog because I was already familiar with it, and it covers the traditional service side well. Since we also run a fleet of agents, I looked into AI-native monitoring tools like Braintrust to see if there was anything specific to agents that we might need but would not get from our Datadog setup.

One more reason for me to write it down: we can't tell Claude Code "set up monitoring for me" and expect a finished system. The engineer still has to decide what's worth watching. Even after I broke the task down into detailed requirements, the AI still missed key details—for example, trace-to-log linking, as you'll see. What Codex and Claude Code did was great heavy-lifting: a platofmr’s monitoring setup that can previously take people more than three weeks might just take about a week now.

## Table of Contents

- [What we set up in Datadog](#what-we-set-up-in-datadog)
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
- [Which one, when](#which-one-when)
- [Implementation note: Azure log-trace correlation](#implementation-note-azure-log-trace-correlation)

---

## What we set up in Datadog

We built the operational side with four Datadog products: Synthetics, Logs, APM, and Infrastructure Monitoring. Monitors and dashboards sit on top of those signals. For each one, I'll explain why we added it, what mattered in the implementation, and how Braintrust approaches the same area.

Datadog also offers [**Agent Observability**](https://docs.datadoghq.com/llm_observability/), an LLM-native product that captures model inputs and outputs and includes evaluations, datasets, and experiments. I haven't used it, so the Datadog discussion below is about the stack we actually built.

Both platforms are moving quickly, so the product details in this post are current as of **August 2026**.

The architecture looks like this:

[![An LLM agent service sends logs and traces to Datadog, while an external probe checks availability and cloud integrations provide infrastructure metrics. Datadog monitors and dashboards use all four data sources.](/llm-agent-observability-stack.svg)](/llm-agent-observability-stack.svg)

_This is the Datadog setup we built. [Open the diagram at full size.](/llm-agent-observability-stack.svg)_

---

### 1. Synthetics: is my service up?

Synthetics was the smallest part to set up, but I think it may be the most important. An external prober hits a health endpoint on a schedule, making this the only component that directly checks availability from outside our network. If the app stops responding, its logs and traces may show nothing. Infrastructure or no-data monitors can hint at a failure, but they do not prove that a user can reach the endpoint.

Here is an example of the Terraform we used:

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

Our `/health` endpoint also runs a lightweight query against a table the application depends on, so the check covers both the application and the database. We cache the result for 30 seconds to avoid adding unnecessary database traffic.

In FastAPI, it looks roughly like this:

```python
from fastapi import HTTPException

@app.get("/health")
async def health():
    query = "SELECT 1 FROM conversations LIMIT 1"
    if not await cached_query_succeeds(query, ttl_seconds=30):
        raise HTTPException(status_code=503)
    return {"ok": True}
```


**Braintrust:** Based on what I researched, I don't think Braintrust offers a documented equivalent for this kind of availability probe.

---

### 2. Logs: what happened?

Application logs already tell us about requests, errors, and jobs. To see how an agent behaved within a turn, we added one structured record at the end of every turn:

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

Keeping these values as fields matters at query time. A sentence like `"turn took 4200ms"` cannot be aggregated as a number until it is parsed; a field like `duration_ms: 4200` can be averaged, used for percentiles, and alerted on directly.

For us, getting those fields queryable took two steps in a Datadog *pipeline*, the ordered list of processors applied to matching logs.

**First, parse the line.** Datadog parses bare JSON automatically, but Azure Container Apps that we use wraps each stdout line in an envelope and stores the JSON as a string in `properties.Log`. Datadog sees that string as one opaque value. We added a grok parser with `%{data::json}` to parse it and lift its keys into top-level attributes. Because the rule parses arbitrary JSON, it does not need to change when we add fields.

**Then verify the fields Datadog treats as special** — `status`, `message`, `timestamp`, `service`, and `trace_id`. Datadog recognizes common names automatically, but a custom envelope or parser can leave them in the wrong place. The timestamp was especially important for us. Without a date remapper, Datadog used the logs' arrival time; logs arrived in batches up to seven minutes late and shared the same timestamp. That broke their ordering and could put them outside a trace's time window.

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

**Braintrust:** Logs work differently. Its [Logs page](https://www.braintrust.dev/docs/observe/view-logs) is a trace store, not a general stdout stream. You instrument calls and Braintrust records their input, output, exceptions and timing; it does not collect a `logging.warning(...)` or library log line merely because it occurred inside a traced function. I cover that instrumentation in the tracing section next.

---

### 3. APM: where did the time go?

Logs tell you that a turn took thirty seconds. Tracing shows where those thirty seconds went.

**The architecture matters here.** In our deployment, Datadog's tracer sends spans to a collector that Datadog calls the *Agent*, not directly to Datadog. To avoid confusing it with our LLM agents, I'll call it the collector. It runs as a **sidecar** beside the application container and is reachable on localhost:

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

**In code, auto-instrumentation creates most of the trace.** `ddtrace` instruments supported web frameworks, database drivers, and model SDKs. For an agent trace, the key span we add ourselves wraps every tool call. Without it, we can see the model and database calls, but not which tools the agent chose, how long each one took, or where one failed:

```python
from ddtrace import tracer

with tracer.trace("agent.tool", resource=tool_name) as span:
    span.set_tag("tool.name", tool_name)
    result = await execute_tool(tool_name, params)
```

**Braintrust:** Its SDK ships spans directly, so there is no collector, sidecar, or port to configure. [Python auto-instrumentation](https://www.braintrust.dev/docs/instrument/trace-llm-calls) patches supported AI libraries:

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

The two traces answer different questions. Datadog puts model time beside SQL time automatically. Braintrust requires manual or OpenTelemetry instrumentation for the non-AI work, but it stores model inputs and outputs as first-class trace data.

---

### Connecting logs and traces

This is the wiring between Logs and APM, not a separate product. Once they are connected, we can jump from a slow span in the trace directly to the logs from that operation instead of matching them by timestamp.

For us, the first step was turning on log injection. The tracer can be activated in either of these ways, and then it adds the active span to each log record:

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

**Braintrust:** It does not need the same correlation step for its own traces. It stores the function input, output and timing together on its [Logs page](https://www.braintrust.dev/docs/observe/view-logs). Your separate stdout logs remain separate, but the AI interaction itself does not require Datadog-style log-to-APM wiring.

---

### 4. Infrastructure metrics: the layer underneath

At first, most of my attention was on what the application reported through logs and traces. Infrastructure metrics cover the layer underneath: CPU, memory, restarts, database connections, and deadlocks.

These metrics matter for agents because a worker can be OOM-killed in the middle of a task, sometimes without leaving a useful application log. The metrics can show memory approaching the limit before the worker is killed.

**Braintrust:** I did not find the same coverage for our application infrastructure. It does expose an [infrastructure dashboard for its self-hosted data plane](https://www.braintrust.dev/docs/admin/self-hosting#infra-dashboard), but that monitors Braintrust's own deployment, not the containers, hosts or database running your agent.

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

**Braintrust:** It supports alerts too, but over its trace data. [Log alerts](https://www.braintrust.dev/docs/observe/alerts) use a SQL filter. Two examples are ordinary errors and quality scores:

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

**Braintrust:** Its dashboards cover some of the same ground. [Custom dashboards](https://www.braintrust.dev/docs/observe/dashboards) chart request counts, latency, token usage, cost and scores over time. The first few overlap with what you'd build in classic Datadog. You can send custom quality scores into Datadog yourself, and Datadog Agent Observability supports them natively. In Braintrust, scores are already part of the same tracing and evaluation workflow.

---

## What you end up with

Together, the six components answer a specific set of questions: **is it up, what happened, where did the time go, is the machine healthy, what should wake me, and what's the trend?**

For our agent, tracing turned out to be the most useful part. Here is one turn that took thirty-three seconds end to end:

| span              | time            |
| ----------------- | --------------- |
| HTTP request      | **33s**         |
| ↳ model calls ×5  | **27.8s — 84%** |
| ↳ tool calls ×4   | **62ms total**  |
| ↳ every SQL query | under 25ms      |

Without the trace, I would have started with the tool code or database queries. But those took only milliseconds; five model calls accounted for almost all the time. **In this trace, the lever was how many times the agent called the model**, making this a prompt and control-flow problem rather than a low-level code optimization problem. If your agent feels slow, measure the split before optimizing anything.

## What the classic stack can't tell you

All of that describes the *shape* of a request. None of it describes the *content*.

Our classic APM trace knows how many times the model was called and how long those calls took. It doesn't know what was asked, what came back, or whether the answer was right — and for an agent that last one is the question. Datadog Agent Observability seems to close that gap, but we implemented our own observability stack.

## What Braintrust adds

What stood out to me about Braintrust comes from one design choice: it stores prompts and completions alongside timing data. That makes several useful workflows possible.

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

The same scorer can run against production traces through an [online scoring automation](https://www.braintrust.dev/docs/evaluate/score-online). It runs asynchronously on all matching traffic or on a sampled percentage.

**[Experiments as immutable records.](https://www.braintrust.dev/docs/evaluate/run-evaluations)** Each run is a fixed snapshot you can compare against another, so "did prompt v3 beat v2" has an answer rather than an anecdote — and it can run in CI to catch regressions before release.

**[Production traces become test cases.](https://www.braintrust.dev/docs/annotate/datasets)** You can filter live traces, add interesting cases to a dataset, and reuse them in future experiments. A real failure can become a regression test rather than a note in a ticket.

**[Prompts extractable to a playground.](https://www.braintrust.dev/docs/observe/view-logs#iterate-in-playgrounds)** You can open the prompt and inputs from a production trace in a playground, try variations, and compare them with your datasets and scorers.

## Which one, when

**Datadog's strength is everything around the model**, and that's where the failures actually are: a worker killed for memory, a sync job that stops writing while the agent keeps answering from stale data, a deploy that quietly stops shipping. Its blind spot is the content: it measures the model call without storing what was in it.

**Braintrust's strength is the agent-specific side**: it captures model inputs and outputs, scores their quality, and lets you run and compare experiments as you change prompts, models, or the agent itself.

That difference is what shapes the choice. If you manage the infrastructure, Datadog. If the agent is all you own—the machines are someone else's problem—Braintrust. If both, start with Datadog, because its failures are the ones that take you offline, and add Braintrust when the questions Datadog can't answer start to matter. We're in the third case, and we're building the content half ourselves.

---

## Implementation note: Azure log-trace correlation

Connecting logs and traces exposed one Azure-specific problem that was easy to miss. In the logs Azure sent to Datadog, our stdout JSON arrived as a string inside `properties.Log`, where a custom grok processor extracted it. The pipeline applied cleanly, but the extracted ID did not end up as Datadog's reserved trace attribute. This was the issue Claude Code's configuration missed: nothing looked broken, but the trace had no related logs.

What made the problem clear was inspecting a processed log and checking three things: the trace ID existed, it was a string, and Datadog showed it as the reserved `trace_id`. In our first version, the extracted `dd.trace_id` was present but was not recognized as that reserved field. We renamed it to `trace_id` in the formatter and pointed the Trace Remapper to the renamed key. `get_log_correlation_context()` returns the five dotted Datadog keys, so the formatter has to make that rename explicit:

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
