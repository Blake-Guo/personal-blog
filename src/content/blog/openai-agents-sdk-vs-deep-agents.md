---
title: "OpenAI Agents SDK vs. LangChain DeepAgent: which one to use"
description: "A practical comparison of OpenAI Agents SDK and LangChain's Deep Agents, with brief looks at Claude Agent SDK and CrewAI."
pubDate: 2026-09-05
heroImage: "../../assets/openai-deep-agents-hero.png"
---

I have worked with several agent SDKs, starting with LangGraph in its early days and later turning to OpenAI Agents SDK at Galvant.AI right now. Along the way, I have had the chance to explore and use different agent architectures, including OpenAI Agents SDK, Deep Agents, and Claude Agent SDK. It is fun to see how different teams think about and apply the architecture and philosophy of an agent harness. As more agent frameworks appear and people like to discuss which one to use, I wrote this post to summarize some learning.

This article mainly compares **OpenAI Agents SDK** and **Deep Agents**, LangChain's agent harness built on LangGraph. We will also take a brief look at Claude Agent SDK and CrewAI. The details are current as of **September 2026**.

## Table of Contents

- [Which one should you choose?](#which-one-should-you-choose)
- [The shared foundation: build one basic agent](#the-shared-foundation-build-one-basic-agent)
  - [Subagents: returning a result or taking over](#subagents-returning-a-result-or-taking-over)
- [Sandboxes: the main architectural difference](#sandboxes-the-main-architectural-difference)
  - [OpenAI: sandbox lifecycle is part of the SDK](#openai-sandbox-lifecycle-is-part-of-the-sdk)
  - [Deep Agents: the sandbox is a backend](#deep-agents-the-sandbox-is-a-backend)
  - [The sandbox difference in one table](#the-sandbox-difference-in-one-table)
- [Other differences beyond sandboxes](#other-differences-beyond-sandboxes)
  - [Lean primitives vs. a prebuilt harness](#lean-primitives-vs-a-prebuilt-harness)
  - [Tool search versus dynamic tools](#tool-search-versus-dynamic-tools)
- [Other agent harnesses we have looked at](#other-agent-harnesses-we-have-looked-at)
  - [Claude Agent SDK](#claude-agent-sdk)
  - [CrewAI](#crewai)
- [What we use](#what-we-use)

## Which one should you choose?

If you only have a minute, either framework is a reasonable choice for a general agent application.

Choose **OpenAI Agents SDK** if you want to stay in the OpenAI ecosystem, are building realtime or voice agents, or need explicit, deeply customizable control over sandbox lifecycle and composition. ([OpenAI voice agents](https://developers.openai.com/api/docs/guides/voice-agents))

Choose **Deep Agents** if you want one high-level interface for both normal and sandbox-backed agents (if you need them later), stronger model agnosticism, and a larger ecosystem and community.

## The shared foundation: build one basic agent

At the foundation, the frameworks are very similar. We define an agent, give it instructions and tools, run it with a user message, and read the answer. Here is the same small weather agent in both SDKs.

To run the examples, use Python 3.11 or later and install `openai-agents`, `deepagents`, and `langchain-openai`. Set `OPENAI_API_KEY` as well. The later LangSmith sandbox examples additionally require `langsmith[sandbox]` and LangSmith credentials.

### OpenAI Agents SDK

```python
import asyncio

from agents import Agent, Runner, function_tool


@function_tool
def get_weather(city: str) -> str:
    """Return the weather for a city."""
    return f"The weather in {city} is sunny."


agent = Agent(
    name="Weather bot",
    model="gpt-5.5",
    instructions="Answer weather questions. Use get_weather when needed.",
    tools=[get_weather],
)


async def main() -> None:
    result = await Runner.run(agent, "What is the weather in Seattle?")
    print(result.final_output)


asyncio.run(main())
```

### Deep Agents

```python
from deepagents import create_deep_agent


def get_weather(city: str) -> str:
    """Return the weather for a city."""
    return f"The weather in {city} is sunny."


agent = create_deep_agent(
    model="openai:gpt-5.5",
    system_prompt="Answer weather questions. Use get_weather when needed.",
    tools=[get_weather],
)

result = agent.invoke(
    {"messages": [{"role": "user", "content": "What is the weather in Seattle?"}]}
)
print(result["messages"][-1].content)
```

The surface difference is small. With OpenAI, we define an `Agent`, then pass it to `Runner.run()`, the SDK entry point that executes the agent. The runner calls the model, executes any requested tools or handoffs, and repeats until it reaches a final answer or another stopping condition. With Deep Agents, `create_deep_agent()` returns a compiled LangGraph agent that we invoke directly; the graph runtime handles the same loop. ([OpenAI running agents](https://developers.openai.com/api/docs/guides/agents/running-agents), [Deep Agents quickstart](https://docs.langchain.com/oss/python/deepagents/quickstart))

### Subagents: returning a result or taking over

Both frameworks let us split work across agents. Deep Agents follows a manager pattern: the supervisor calls a synchronous subagent through `task()`, waits for its result, and remains responsible for the conversation. One interesting detail is that every deep agent also receives a `general-purpose` subagent by default unless we disable or replace it. ([Deep Agents subagents](https://docs.langchain.com/oss/python/deepagents/subagents))

OpenAI makes one subtle semantic choice explicit. When one agent calls another, who owns the next user-facing response?

- With `Agent.as_tool()`, the specialist is a bounded helper. It returns a result, then the manager continues and answers the user.
- With a **handoff**, the specialist takes over the conversation and produces the next response.

The first one is similar to the Deep Agents subagent. OpenAI's two APIs make it explicit which agent will respond to the user next. ([OpenAI orchestration](https://developers.openai.com/api/docs/guides/agents/orchestration))

[![A diagram comparing an OpenAI agent used as a tool, an OpenAI handoff, and a Deep Agents task subagent.](/agent-delegation-patterns.svg)](/agent-delegation-patterns.svg)

_[Open the delegation diagram at full size.](/agent-delegation-patterns.svg)_

## Sandboxes: the main architectural difference

To me, sandbox management is the biggest difference between these frameworks. It is also one of the clearest recent directions in agent-harness design: the framework is no longer responsible only for model and tool calls; it increasingly needs to account for where model-directed work executes and how that environment survives a long-running task.

Suppose we ask an agent to fix a bug. It may clone a repository, install dependencies, edit files, run tests, wait for our review, and continue tomorrow. The commands are not known in advance, and every later step depends on files and packages created earlier. A sandbox gives the agent an isolated workspace that we can inspect, preserve, reconnect to, or discard without running those commands directly on our application server or laptop. ([OpenAI Sandbox Agents](https://developers.openai.com/api/docs/guides/agents/sandboxes), [Deep Agents sandboxes](https://docs.langchain.com/oss/python/deepagents/sandboxes))

Both frameworks support that workflow, but they put the abstraction in different places.

### OpenAI: sandbox lifecycle is part of the SDK

OpenAI introduces a specialized `SandboxAgent` rather than adding a backend to the regular `Agent`. The API is currently in beta. If we do not provide a capability list, it includes filesystem, shell, and compaction capabilities by default. A `Manifest` describes the starting workspace, while `RunConfig(sandbox=SandboxRunConfig(...))` selects the sandbox client, live session, resumable session state, manifest, snapshot, and provider options for a run.

**Basic example.** The smallest local setup looks much like a regular agent run, except that we create a `SandboxAgent` and supply a sandbox client through `RunConfig`:

```python
import asyncio

from agents import Runner
from agents.run import RunConfig
from agents.sandbox import Manifest, SandboxAgent, SandboxRunConfig
from agents.sandbox.entries import File
from agents.sandbox.sandboxes.unix_local import UnixLocalSandboxClient


manifest = Manifest(
    entries={
        "task.md": File(
            content=b"Create hello.py, run it, and write the output to result.txt."
        ),
    }
)

agent = SandboxAgent(
    name="Coding agent",
    model="gpt-5.6",
    instructions="Read task.md, complete the task, and summarize what you changed.",
    default_manifest=manifest,
)


async def main() -> None:
    result = await Runner.run(
        agent,
        "Complete the task in the workspace.",
        run_config=RunConfig(
            sandbox=SandboxRunConfig(client=UnixLocalSandboxClient()),
        ),
    )
    print(result.final_output)


asyncio.run(main())
```

The `Manifest` is the starting contract for a fresh workspace: it can contain files, directories, repositories, mounts, environment values, and sandbox users. Here the runner materializes `task.md` before the agent starts. `UnixLocalSandboxClient` gives us the shortest development loop, but it is for local development rather than strong environment isolation. We can replace it with Docker or a hosted client without changing the agent or its manifest. ([OpenAI Sandbox Agents](https://developers.openai.com/api/docs/guides/agents/sandboxes#create-the-workspace))

The interesting part is how explicitly the SDK models time. `RunState` resumes the agent workflow, serialized sandbox session state reconnects to the same environment, and a snapshot seeds a new environment from saved workspace contents. The runner has a documented resolution order: reuse a live session, resume from `RunState`, resume explicit session state, or create a fresh session. These concepts are more machinery to learn, but they give us precise control over long-running work. ([OpenAI Sandbox Agents](https://developers.openai.com/api/docs/guides/agents/sandboxes))

**Advanced example: expose sandbox agents as tools.** A normal `Agent` can coordinate sandbox specialists. Here each specialist gets a separate run configuration, so each nested agent run creates its own workspace:

```python
import asyncio

from agents import Agent, Runner
from agents.run import RunConfig
from agents.sandbox import Manifest, SandboxAgent, SandboxRunConfig
from agents.sandbox.entries import File
from agents.sandbox.sandboxes.unix_local import UnixLocalSandboxClient


dependency_manifest = Manifest(
    entries={
        "requirements.txt": File(content=b"requests==2.32.3\n"),
    }
)
test_manifest = Manifest(
    entries={
        "calculator.py": File(content=b"def add(a, b):\n    return a + b\n"),
        "test_calculator.py": File(
            content=b"from calculator import add\n\ndef test_add():\n    assert add(2, 3) == 5\n"
        ),
    }
)

dependency_reviewer = SandboxAgent(
    name="Dependency reviewer",
    model="gpt-5.6",
    instructions="Inspect requirements.txt, run any useful checks, and return an audit.",
    default_manifest=dependency_manifest,
)
test_runner = SandboxAgent(
    name="Test runner",
    model="gpt-5.6",
    instructions="Inspect the Python files, run the tests, and return the result.",
    default_manifest=test_manifest,
)

manager = Agent(
    name="Engineering manager",
    model="gpt-5.6",
    instructions="Call both specialists, then combine their findings.",
    tools=[
        dependency_reviewer.as_tool(
            tool_name="review_dependencies",
            tool_description="Audit dependencies in its own workspace.",
            run_config=RunConfig(
                sandbox=SandboxRunConfig(client=UnixLocalSandboxClient()),
            ),
        ),
        test_runner.as_tool(
            tool_name="run_tests",
            tool_description="Run tests in its own workspace.",
            run_config=RunConfig(
                sandbox=SandboxRunConfig(client=UnixLocalSandboxClient()),
            ),
        ),
    ],
)


async def main() -> None:
    result = await Runner.run(manager, "Run both reviews and summarize the risks.")
    print(result.final_output)


asyncio.run(main())
```

The outer manager remains a normal agent. Each specialist carries its own `default_manifest`, while the `run_config` on each `as_tool()` call controls where that manifest is materialized and the work executes. We can instead pass the same live `SandboxSession` when specialists should work in one shared workspace. A handoff is another option when the sandbox agent should take over the current top-level run. ([OpenAI sandbox composition](https://developers.openai.com/api/docs/guides/agents/sandboxes#compose-sandbox-agents))

### Deep Agents: the sandbox is a backend

Deep Agents is **filesystem-native, but a filesystem is not the same as a sandbox**. Filesystem tools let the agent list, read, and write files. A sandbox also gives those files an isolated process environment where the agent can execute shell commands. Deep Agents always exposes filesystem tools; without a backend argument, they use `StateBackend`, which stores virtual files in LangGraph state and cannot run shell commands. Passing a sandbox backend keeps the same agent interface but gives the agent the power to run commands inside an isolated environment through the built-in `execute` tool.

**Basic example.** Here we create a LangSmith sandbox, wrap it as a Deep Agents backend, and pass that backend to the same `create_deep_agent()` function used for a normal agent:

```python
from deepagents import create_deep_agent
from deepagents.backends import LangSmithSandbox
from langsmith.sandbox import SandboxClient


client = SandboxClient()
sandbox = client.create_sandbox()
backend = LangSmithSandbox(sandbox=sandbox)

agent = create_deep_agent(
    model="openai:gpt-5.5",
    backend=backend,  # Automatically adds filesystem tools and execute
    system_prompt="Work in the sandbox, make the requested change, and run the tests.",
)

try:
    result = agent.invoke(
        {
            "messages": [
                {
                    "role": "user",
                    "content": "Create hello.py, then use execute to run python hello.py.",
                }
            ]
        }
    )
    print(result["messages"][-1].content)
finally:
    client.delete_sandbox(sandbox.name)
```

There is no separate sandbox-agent class, and we do not add `execute` to `tools` ourselves. Deep Agents detects that the backend supports sandbox execution and automatically gives the agent its filesystem tools and the `execute` tool. The provider client still creates and removes the environment. ([Deep Agents sandboxes](https://docs.langchain.com/oss/python/deepagents/sandboxes#basic-usage))

The agent API therefore stays the same while the backend determines where files live and whether shell execution is available. Deep Agents also exposes one pluggable filesystem interface across thread state, local disk, durable stores, and sandbox compute. `CompositeBackend` can even route different path prefixes to different backends—for example, ephemeral scratch files in state and `/memories/` in durable storage. ([Deep Agents backends](https://docs.langchain.com/oss/python/deepagents/backends))

The trade-off is where environment lifecycle lives. The common Deep Agents sandbox backend standardizes file operations and command execution, not container creation or retention. The production guide shows how our application can look up a provider sandbox by thread or assistant ID, rely on provider TTLs, and use provider-specific snapshots. Deep Agents therefore supports long-lived sandboxes, but their lifecycle is application and provider wiring rather than one standardized runner contract. ([Deep Agents production guide](https://docs.langchain.com/oss/python/deepagents/going-to-production#sandboxes))

By default, Deep Agents wires the same backend into the supervisor and its declarative synchronous subagents. A subagent can write a file, return a short result, and leave the detailed artifact for the supervisor or another subagent to read later. That shared workspace fits Deep Agents' manager-style delegation naturally.

**Advanced example: share one sandbox with subagents.** We can add specialists without configuring a separate sandbox for each one:

```python
from deepagents import create_deep_agent
from deepagents.backends import LangSmithSandbox
from langsmith.sandbox import SandboxClient


client = SandboxClient()
team_sandbox = client.create_sandbox()
team_backend = LangSmithSandbox(sandbox=team_sandbox)

team = create_deep_agent(
    model="openai:gpt-5.5",
    backend=team_backend,  # Shared by the supervisor and its subagents
    system_prompt="Create the code, delegate review and testing, then fix any issues.",
    subagents=[
        {
            "name": "code-reviewer",
            "description": "Reviews code already present in the shared workspace.",
            "system_prompt": "Review the workspace code and report concrete issues.",
        },
        {
            "name": "test-runner",
            "description": "Writes and runs tests in the shared workspace.",
            "system_prompt": "Inspect the code, add useful tests, run them, and report failures.",
        },
    ],
)

try:
    result = team.invoke(
        {
            "messages": [
                {
                    "role": "user",
                    "content": "Create calculator.py, ask both specialists to check it, and fix it.",
                }
            ]
        }
    )
    print(result["messages"][-1].content)
finally:
    client.delete_sandbox(team_sandbox.name)
```

The supervisor delegates through Deep Agents' `task` tool. Because the backend is shared, the reviewer can read the supervisor's files, the test runner can add tests, and the supervisor can inspect those changes afterward. This is convenient when the team should collaborate in one workspace. ([Deep Agents backends](https://docs.langchain.com/oss/python/deepagents/backends), [Deep Agents subagents](https://docs.langchain.com/oss/python/deepagents/subagents))

If we want a different environment for one subagent, its dictionary has no direct `backend` field. We can replace that subagent's filesystem middleware or provide a `CompiledSubAgent` with a different setup, but we must wire it ourselves. This is the accurate distinction: per-agent sandbox configuration is **first-class in OpenAI and custom in Deep Agents**, not possible in one and impossible in the other. ([Deep Agents backends](https://docs.langchain.com/oss/python/deepagents/backends), [Deep Agents subagents](https://docs.langchain.com/oss/python/deepagents/subagents))

### The sandbox difference in one table

| Dimension | OpenAI Sandbox Agents | Deep Agents with a sandbox backend |
| --- | --- | --- |
| **Entry point** | `SandboxAgent(...)` plus `RunConfig(sandbox=SandboxRunConfig(...))` | `create_deep_agent(..., backend=sandbox)` |
| **Lifecycle owner** | The runner resolves sessions; the sandbox client implements supported lifecycle operations | Our application or provider creates and reuses the environment; the backend operates inside it |
| **Fresh workspace** | `Manifest` or a supported snapshot | Provider creation plus file uploads, backend storage, or a provider-specific snapshot |
| **Multiple agents** | Each sandbox agent used as a tool can have its own client, manifest, and run configuration | Declarative subagents share the top-level backend by default; separate setups require middleware or a custom graph |
| **Filesystem model** | A specialized sandbox workspace with files, mounts, commands, ports, and snapshots | One backend interface across virtual files, disk, stores, and sandboxes, with optional path routing |

[![A diagram comparing OpenAI's explicit sandbox lifecycle with the Deep Agents backend interface.](/sandbox-architecture-comparison.svg)](/sandbox-architecture-comparison.svg)

_OpenAI standardizes session resolution and saved-state inputs in the runner. Deep Agents standardizes the filesystem/backend interface, while sandbox creation and reuse stay in the graph factory or application and provider. [Open the diagram at full size.](/sandbox-architecture-comparison.svg)_

## Other differences beyond sandboxes

Sandbox architecture is the largest difference, but it is not the only one.

### Lean primitives vs. a prebuilt harness

The basic examples look similar, but the libraries start at different levels. OpenAI gives us a lean `Agent` and runner; we add the tools, sessions, handoffs, and guardrails our application needs. Deep Agents gives us a prebuilt harness on top of LangGraph: planning with `write_todos`, filesystem tools, automatic summarization, and a general-purpose subagent are already wired in. ([OpenAI agent definitions](https://developers.openai.com/api/docs/guides/agents/define-agents), [Deep Agents overview](https://docs.langchain.com/oss/python/deepagents/overview))

The practical difference is how much architecture we choose ourselves. OpenAI starts with fewer assumptions; Deep Agents provides more behavior by default.

### Tool search versus dynamic tools

OpenAI's `tool_search` is worth calling out, but it belongs primarily to the OpenAI model and Responses API layer rather than being a new agent-orchestration pattern. We mark functions, namespaces, or MCP servers for deferred loading, and the model loads only the definitions it needs. In hosted mode, OpenAI searches the deferred tools declared in the request. In client-executed mode, the model sends a search call and our application returns the matching tool definitions. It currently requires GPT-5.4 or later. ([OpenAI tool search](https://developers.openai.com/api/docs/guides/tools-tool-search))

The closest framework-level counterpart in Deep Agents is LangChain's dynamic-tool machinery. Middleware can filter a pre-registered tool set before a model call or register and execute tools discovered at runtime. The distinction is architectural: OpenAI tool search is a provider/model feature designed around deferred tool definitions and prompt caching; LangChain dynamic tools are application-side middleware and remain model agnostic. Neither approach is universally better. ([LangChain dynamic tools](https://docs.langchain.com/oss/python/langchain/agents#dynamic-tools))

## Other agent harnesses we have looked at

### Claude Agent SDK

Claude Agent SDK is best understood as **Claude Code's agent loop packaged as a Python or TypeScript library**. It runs the loop in our process and includes the built-in file, shell, search, and editing tools, along with sessions, permissions, hooks, skills, MCP, and subagents. ([Claude Agent SDK overview](https://code.claude.com/docs/en/agent-sdk/overview))

It is a capable, ready-made harness, but it is designed around the Claude model family. We can access Claude directly through Anthropic or through Amazon Bedrock, Google Vertex AI, and Microsoft Foundry. We do not use it because we want the freedom to switch model families. ([Claude deployment options](https://code.claude.com/docs/en/third-party-integrations))

### CrewAI

I looked at CrewAI some time ago. Its language is approachable: we describe agents with roles and goals, assign tasks, and organize the agents into crews. CrewAI AMP also includes Crew Studio, a no-code/low-code interface. That presentation can be especially approachable for non-engineering teams or teams with limited engineering support. ([CrewAI agents](https://docs.crewai.com/en/concepts/agents), [CrewAI AMP](https://docs.crewai.com/enterprise/introduction))

CrewAI is not limited to that high-level crew metaphor. Its Flows provide explicit, event-driven steps, shared state, branching, and loops. Our decision was about fit rather than a lack of capability: we expected to customize and optimize the runtime deeply, and preferred to work closer to the underlying abstractions. ([CrewAI Flows](https://docs.crewai.com/en/concepts/flows))

## What we use

As stated at the beginning, for many use cases either framework is a reasonable choice. At Galvant.ai, the startup where I work, we use OpenAI Agents SDK. Many of our products already use several OpenAI APIs, so it was the natural choice for us. And the SDK supports non-OpenAI models as well, at least for now, and we can use Anthropic models in some cases. ([OpenAI models and providers](https://developers.openai.com/api/docs/guides/agents/models))
