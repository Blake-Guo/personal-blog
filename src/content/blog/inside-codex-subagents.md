---
title: "CodeX Architect Part 3 - Subagents"
description: "We follow a delegated task through Codex's Rust core: child configuration, a separate conversation, shared files, mailbox messages, and the final answer returned to the parent."
pubDate: 2026-10-10
draft: true
heroImage: "../../assets/codex-subagents-hero.png"
---

In our earlier blog post, [CodeX Architect Part 1 - Context Engineering](/blog/inside-codex-context-engineering/), we followed one user message until it became a model request. Here I wanted to understand delegation: when Codex starts a subagent, what does that agent inherit, how does it work alongside the parent, and how do its findings get back?

The investigation that helped me find these paths was written with another agent. I went back through the local source for this post, especially where the older investigation and the current implementation differed.

## Table of Contents

- [The task we will follow](#the-task-we-will-follow)
- [1 The parent gets delegation tools](#1-the-parent-gets-delegation-tools)
- [2 The model asks for a child](#2-the-model-asks-for-a-child)
- [3 Building the child's configuration](#3-building-the-childs-configuration)
- [4 Starting a separate conversation](#4-starting-a-separate-conversation)
- [5 Sending a clue while the child works](#5-sending-a-clue-while-the-child-works)
- [6 The parent waits for activity](#6-the-parent-waits-for-activity)
- [7 Returning the child's final answer](#7-returning-the-childs-final-answer)
- [The call paths](#the-call-paths)
- [What I would use this for](#what-i-would-use-this-for)

## The task we will follow

We will follow [commit `c3d3b14` (October 10, 2026)](https://github.com/openai/codex/tree/c3d3b142d10f4316b46e35aad7e5317e7e506cb7) and a request like "*Investigate why token refresh sometimes skips expiry checks. Use one subagent to trace the callers of `refresh_token()` while we inspect the expiry logic. Do not edit files, and wait for its findings before concluding.*"

Our example uses multi-agent **V2**, the bundled delegation guidance, medium reasoning effort, read-only permissions, and no custom roles or subagent model defaults. We follow a possible sequence of model decisions: the parent creates a child named `trace_callers` with `fork_turns: "none"`, then sends it a clue and waits for its findings. Tool arguments are illustrative; the Rust excerpts are extracted from the pinned commit. Model catalogs and configuration can change the [protocol selection](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/config/mod.rs#L1628-L1655), [guidance](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/multi_agents.rs#L77-L120), and [tool definitions](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/spec_plan.rs#L1480-L1578).

The parent keeps investigating expiry logic. The child starts a separate conversation for tracing callers, with its own model requests and tool results. Both use the same workspace; the boundary is between their conversations. We can see the [child startup](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/agent/control/spawn.rs#L783-L848) and the [shared-directory guidance](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/prompts/src/multi_agent_instructions.rs#L13-L16) in the source.

<a href="/codex-subagent-task-flow.svg">
  <picture>
    <source media="(max-width: 600px)" srcset="/codex-subagent-task-flow-mobile.svg" />
    <img src="/codex-subagent-task-flow.svg" alt="Codex V2 delegation: the parent model requests a child, the runtime builds its configuration and starts its conversation, parent and child work alongside each other, and the child's final answer returns through the parent mailbox for a later accepted model step. Both agents share the workspace." />
  </picture>
</a>

_[Open the delegation flow at full size.](/codex-subagent-task-flow.svg)_

Function headings use Rust's `module::function` or `Type::method` notation, with comments explaining the work underneath. At full size, those headings link to their source definitions.

The diagram follows the fresh-history branch used by our example. Startup details and optional history copying are condensed; the following sections show where those choices happen.

## 1 The parent gets delegation tools

Our request first reaches the parent's normal turn loop. Before the model can delegate, [`add_collaboration_tools`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/spec_plan.rs#L1480-L1578) registers the V2 collaboration tools when they are enabled for that turn. With direct messaging and waiting enabled, the available operations are:

| Tool | What the parent can do |
| --- | --- |
| [`spawn_agent`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/handlers/multi_agents_v2/spawn.rs#L103-L221) | Start a child on a task. |
| [`send_message`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/handlers/multi_agents_v2/send_message.rs#L33-L49) | Queue a message to an existing agent. |
| [`followup_task`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/handlers/multi_agents_v2/followup_task.rs#L33-L49) | Give an existing child more work and request a turn. |
| [`wait_agent`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/handlers/multi_agents_v2/wait.rs#L40-L96) | Wait for mailbox activity or new user input, up to a timeout. |
| [`list_agents`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/handlers/multi_agents_v2/list_agents.rs#L35-L73) | Inspect agents and their status. |
| [`interrupt_agent`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/handlers/multi_agents_v2/interrupt_agent.rs#L34-L78) | Interrupt an agent's current work. |

For this investigation, the model chooses the caller-tracing task and emits a `spawn_agent` call, which the [spawn handler](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/handlers/multi_agents_v2/spawn.rs#L103-L221) processes.

The instructions accompanying those tools matter too. [`effective_multi_agent_mode`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/multi_agents.rs#L77-L120) first checks a configured hint, then a catalog hint. Without either, Ultra effort selects proactive guidance and other efforts select explicit-request guidance; the catalog can also replace those two messages. In our bundled, medium-effort setup, the instruction says delegation needs an explicit request from the user or applicable project or skill instructions. Our request supplies it. The [bundled messages](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/prompts/src/model_messages/multi_agent.rs#L46-L47) guide the model; startup limits are enforced separately in section 4.

We now have a model that can choose a bounded caller-tracing task. Its next output is the tool call that starts that task.

## 2 The model asks for a child

For our caller-tracing task, [`handle_spawn_agent`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/handlers/multi_agents_v2/spawn.rs#L102-L165) parses the model's arguments and decides whether to copy parent history. With the built-in schema, these illustrative arguments ask for a fresh conversation:

```json
{
  "task_name": "trace_callers",
  "message":
    "Trace refresh_token() callers. Give expiry bypasses + file refs. No edits.",
  "fork_turns": "none"
}
```

The distinction between `"none"` and `"all"` is easy to miss. `"none"` starts without the parent's conversation history. Omitting `fork_turns` defaults to `"all"`, which selects a full-history fork. In [`SpawnAgentArgs::fork_mode`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/handlers/multi_agents_v2/spawn.rs#L280-L305), the highlighted lines show that default, the fresh-history branch, and the full-history branch:

```rust {6,8,12}
let fork_turns = self
    .fork_turns
    .as_deref()
    .map(str::trim)
    .filter(|fork_turns| !fork_turns.is_empty())
    .unwrap_or("all");

if fork_turns.eq_ignore_ascii_case("none") {
    return Ok(None);
}
// Accept legacy turn counts without limiting the inherited history.
if fork_turns.eq_ignore_ascii_case("all") || fork_turns.parse::<NonZeroUsize>().is_ok() {
    return Ok(Some(SpawnAgentForkMode::FullHistory));
}
```

The compatibility branch also accepts a positive numeric string, but it still selects `FullHistory`. It does **not** mean "copy the last N turns". The [advertised schema](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/handlers/multi_agents_spec.rs#L664-L668) asks the model to use only `"all"` or `"none"`.

For this investigation, I would choose `"none"`: the child's question is self-contained, and we do not need to send the whole conversation to explain it. A full fork is useful when earlier requirements or decisions are part of the task. The implementation [loads and prepares copied parent history](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/agent/control/spawn.rs#L1011-L1085) for that branch.

Fresh history still leaves an important question: which instructions, model, and permissions does the child use? The handler resolves those before it creates the conversation.

## 3 Building the child's configuration

Our fresh child now reaches [`prepare_agent_spawn_config`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/agent/child_config.rs#L52-L100). It starts from the parent's effective settings, applies requested model settings and a role where appropriate, and then reapplies the parent's live runtime policy.

The child inherits configuration even when it starts with fresh history. In [`build_agent_spawn_config`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/agent/child_config.rs#L110-L121), the highlighted lines copy the captured model, reasoning effort, and session base instructions:

```rust {3,4,6}
let mut config = build_agent_shared_config(step_context.turn.as_ref())?;
let settings = &step_context.settings;
config.model = Some(settings.model_info.slug.clone());
config.model_reasoning_effort = settings.effective_reasoning_effort();
config.model_reasoning_summary = Some(settings.reasoning_summary);
config.base_instructions = Some(base_instructions.text.clone());
config.base_instructions_provenance = base_instructions.provenance.clone();
Ok(config)
```

Here is the order relevant to our example:

| Layer | What it contributes |
| --- | --- |
| [Parent's captured settings](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/agent/child_config.rs#L110-L164) | Effective configuration, model, reasoning settings, and base instructions. |
| [Requested model and effort](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/agent/child_config.rs#L204-L260) | Explicit spawn values take precedence over the corresponding `[agents]` defaults. Without either, the parent values remain. |
| [Agent role](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/agent/role.rs#L80-L127) | A selected role can replace instructions or model settings and disable selected capabilities. |
| [Live runtime policy](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/agent/child_config.rs#L179-L201) | The parent's approval policy, working directory, and permission-profile snapshot are copied again. |

We omit `agent_type`, so this fresh child takes the [built-in `default` role](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/agent/role.rs#L345-L355), which has no role configuration file. We also omit model overrides and have no configured subagent defaults. The child therefore keeps the captured parent model and effort. If an explicit or configured model is selected without an effort, the [model-override path](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/agent/child_config.rs#L219-L246) instead uses that model's default effort.

### A role customizes the agent within the parent's policy

At this commit, role files are applied through a [bounded set of fields](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/agent/role.rs#L37-L48): instructions, model and reasoning settings, response style, service tier, and selected feature or skill restrictions. The role loader [retains capability changes that disable allowed features or skills](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/agent/role.rs#L91-L117). Permission profiles, provider endpoints, and MCP-server configuration are outside that override set.

There is a useful regression test here. [`apply_role_cannot_expand_parent_authority`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/agent/role_tests.rs#L431-L517) supplies a role containing a different provider, broader sandbox settings, and an extra MCP server. Its assertions check that the parent's permissions, provider, and MCP servers remain in place.

This also narrows a claim in the [current subagent documentation](https://learn.chatgpt.com/docs/agent-configuration/subagents#custom-agents), which describes custom sandbox and MCP overrides. In this pinned role path, those fields are not applied, including a role's request for a tighter sandbox. Our read-only policy comes from the parent session.

With the child's configuration resolved, the controller can admit the spawn and create its thread.

## 4 Starting a separate conversation

Our `trace_callers` child next reaches [`spawn_agent_internal`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/agent/control/spawn.rs#L676-L735). The local controller checks execution capacity, reserves room for a resident child, and reserves its place in the agent registry before starting it.

The [bundled V2 concurrency default](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/config/mod.rs#L254-L258) is four slots including the root. The [effective child limit](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/config/mod.rs#L1657-L1671) subtracts one, leaving three concurrently running children. That limits active work; idle agents can remain in the tree and be [unloaded to make room](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/agent/control/residency.rs#L215-L262). Our example needs one child.

Because we selected `"none"`, the controller skips copied parent history. It still passes the inherited environment into startup. The highlighted lines below show the actual child creation and the guard that owns the unfinished spawn; the [surrounding function](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/agent/control/spawn.rs#L783-L848) shows both history branches:

```rust {2,3,4,5,6,7,8,9,13,14}
let child_create_started_at = Instant::now();
let new_thread = state
    .spawn_child_thread(
        start_options,
        self.clone(),
        parent_thread_id,
        inherited_exec_policy,
    )
    .await
    .map_err(|err| err.with_agent_context(AgentErrorContext::ChildStartup))?;
let child_create = child_create_started_at.elapsed();
agent_metadata.agent_id = Some(new_thread.thread_id);
let mut pending_spawn =
    PendingSpawn::new(Arc::clone(&state), new_thread.thread_id, membership);
```

That guard, [`PendingSpawn`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/agent/control/spawn_guard.rs#L15-L104), schedules cleanup if startup fails before the first input is accepted. On the successful path, the controller [delivers the initial message and commits the reservations](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/agent/control/spawn.rs#L937-L961).

The task message uses `TriggerTurn`. The recipient's [pending-work scheduler](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tasks/mod.rs#L456-L547) starts a regular task, and [`RegularTask::run`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tasks/regular.rs#L40-L114) calls the same [`run_turn`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L164-L198) used by the parent. A Codex "thread" here is a managed conversation with its own history and execution.

With the default `hide_spawn_agent_metadata = true`, the [spawn result](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/handlers/multi_agents_v2/spawn.rs#L245-L256) identifies the child by its canonical task path:

```json
{ "task_name": "/root/trace_callers" }
```

The call waits for startup and input admission, then returns without waiting for the investigation to finish. The parent can inspect expiry logic while the child traces callers. Their separate conversations still read the same workspace, as the [V2 guidance explicitly states](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/prompts/src/multi_agent_instructions.rs#L13-L16). If we were delegating edits, we would need to assign file ownership; the [worker-role description](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/agent/role.rs#L374-L381) gives that guidance.

The parent now has a task path it can use to send a clue to the running child.

## 5 Sending a clue while the child works

Suppose our expiry inspection suggests that a cache path deserves attention. The parent's `send_message` call reaches [`handle_message_string_tool`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/handlers/multi_agents_v2/message_tool.rs#L42-L100). With the built-in schema, the illustrative arguments are:

```json
{
  "target": "trace_callers",
  "message": "Check cache-hit callers for an expiry-guard bypass."
}
```

Routing happens before delivery. [`AgentPath::resolve`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/protocol/src/agent_path.rs#L59-L72) appends a relative target to the sender's path, and the controller [looks up that path in the registry](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/agent/control/target.rs#L45-L60) to obtain the recipient's thread ID. From `/root`, `trace_callers` means `/root/trace_callers`. From a nested agent, the same relative name would identify its own child; an absolute path identifies the intended agent across those levels.

`send_message` and `followup_task` use the same handler but choose different delivery modes:

| Operation | Mode | Effect on an idle recipient |
| --- | --- | --- |
| [`send_message`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/handlers/multi_agents_v2/send_message.rs#L33-L49) | `QueueOnly` | Queues mail; normally leaves the agent idle. |
| [`followup_task`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/handlers/multi_agents_v2/followup_task.rs#L33-L49) | `TriggerTurn` | Requests a turn to process the new task. |

For our already running child, queueing a clue is sufficient. In [`inter_agent_communication`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/handlers.rs#L82-L97), the highlighted lines enqueue the message and show the wake-up condition. An outstanding durable sleep is the exception to the usual queue-only behavior:

```rust {7,8,9,12,13,14}
pub async fn inter_agent_communication(
    sess: &Arc<Session>,
    sub_id: String,
    communication: InterAgentCommunication,
    start_options: codex_protocol::turn_input::TurnStartOptions,
) {
    let trigger_turn = communication.trigger_turn;
    sess.input_queue
        .enqueue_mailbox_communication(communication, start_options)
        .await;
    crate::agent_communication::emit_agent_communication_receive(&sub_id);
    if trigger_turn || sess.has_outstanding_durable_sleep() {
        sess.maybe_start_turn_for_pending_work_with_sub_id(sub_id)
            .await;
    }
}
```

For a loaded recipient, the controller [submits an `Op::InterAgentCommunication`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/agent/control.rs#L287-L310) to that thread. A later accepted step [drains pending mailbox input](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/input_queue.rs#L400-L431), and [`record_inter_agent_communication`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/mod.rs#L3995-L4052) records it in the recipient's conversation history and storage. The model sees the clue through its next request's history. The [delivery receipt](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/agent/api.rs#L227-L232) confirms acceptance; processing happens afterward in the child.

One detail affects inspecting these tasks afterward. The built-in V2 task-message schema [marks the message field as encrypted](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/handlers/multi_agents_spec.rs#L643-L656), and the handler [distinguishes encrypted model-tool messages from direct plaintext input](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/handlers/multi_agents_v2.rs#L54-L65). The JSON above shows the logical arguments before that delivery representation.

The child continues its own model-and-tool loop with the task and any delivered clues. Meanwhile, the parent finishes its expiry inspection and waits for the caller findings.

## 6 The parent waits for activity

After inspecting expiry logic, our parent calls V2 `wait_agent` to wait for the caller findings. Its [`handle_call`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/handlers/multi_agents_v2/wait.rs#L40-L96) subscribes to the parent's input-queue activity. It can wake for mail, new user input, or its timeout. It does not take a child ID or wait on an "all children completed" condition.

The bundled [wait defaults](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/config/mod.rs#L256-L258) are a 30-second timeout, a 10-second minimum, and a one-hour maximum. The handler [raises a shorter request to the minimum and rejects one above the maximum](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/handlers/multi_agents_v2/wait.rs#L53-L65). Those defaults are configurable.

The parent is now waiting for activity while the child continues tracing callers. On our successful path, the child's completion supplies that activity.

## 7 Returning the child's final answer

When our caller-tracing child finishes successfully, [`agent_status_from_event`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/agent/status.rs#L6-L31) turns the completion event into `Completed(last_agent_message)`. The session's [`maybe_notify_parent_of_terminal_turn`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/mod.rs#L2463-L2531) forwards terminal outcomes from V2 spawned children to the agent controller.

The controller's [`notify_parent_of_terminal_turn`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/agent/control/completion.rs#L27-L128) uses the child's recorded parent thread ID and derives the parent path from `/root/trace_callers`. It formats the result and sends it back with `trigger_turn: false`—queue-only delivery to `/root`.

The normal result contains the child's last assistant message. The completion formatter does not bundle its searches, command output, or reasoning into that handoff. In [`format_inter_agent_completion_message`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session_prefix.rs#L20-L37), the highlighted lines copy the completed message and exclude nonterminal states:

```rust {2,10}
let payload = match status {
    AgentStatus::Completed(Some(message)) => message.clone(),
    AgentStatus::Completed(None) => String::new(),
    AgentStatus::Errored(error) => {
        let error = truncate_text(error, TruncationPolicy::Tokens(ERROR_MAX_TOKENS));
        format!("Agent errored: {error}\n\n{ERROR_NEXT_ACTION}")
    }
    AgentStatus::Shutdown => "Agent shut down.".to_string(),
    AgentStatus::NotFound => "Agent was not found.".to_string(),
    AgentStatus::PendingInit | AgentStatus::Running | AgentStatus::Interrupted => return None,
};
Some(InterAgentCompletionMessage::new(task_name, sender, payload).render())
```

Notice that the successful message is copied in full at this handoff. Error text is truncated in a different match arm. I would ask for concise findings here: the successful handoff itself has no length cap, so a child can send a large payload back to its parent.

With `/root` as recipient and `/root/trace_callers` as sender, the [actual completion template](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/context/inter_agent_completion_message.rs#L23-L46) renders this envelope. The last line below is a placeholder for our child's findings:

```text
Message Type: FINAL_ANSWER
Task name: /root
Sender: /root/trace_callers
Payload:
<child's final answer>
```

`Task name` identifies the recipient; `Sender` identifies the child that did the work. This is a separate assistant-role message delivered through the mailbox. If the task had requested a report file, the child could write one and return its path, but the completion path itself copies text.

There are two limits to keep in mind. Ordinary interruption and budget exhaustion become `Interrupted`, which [does not pass the terminal-result filter](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/agent/status.rs#L13-L30); the [session has a separate exception for repeated denials by the Guardian approval reviewer](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/mod.rs#L2491-L2511). Also, parent delivery is best effort: this [send path logs failure and returns](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/agent/control/completion.rs#L121-L141). A missing result is a reason to inspect agent status, rather than treating a timeout as a successful empty answer.

### Reading the returned findings

On this path, mailbox activity wakes the parent's waiting `wait_agent` call. Its built-in result format is:

```json
{ "message": "Wait completed.", "timed_out": false }
```

The [result structure](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/handlers/multi_agents_v2/wait.rs#L141-L174) contains a wait summary and a timeout flag. The child's `FINAL_ANSWER` arrives separately. A completed wait means there is activity to inspect, so the parent still needs to read the sender and payload before deciding whether its task is finished.

Back in [`run_turn`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L164-L198), the [continuation check](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L548-L574) requests another model step when the response requires follow-up or accepted input is pending. The parent's [input queue drains the mailbox](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/input_queue.rs#L400-L431), and the same [history-recording function](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/mod.rs#L3995-L4052) we saw in section 5 records the returned findings. The parent can then combine them with its own expiry investigation.

### What if the answer arrives after the parent finishes?

The parent has a delivery gate for that race. Final-answer output can [defer queue-only mail to the next turn](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/input_queue.rs#L319-L343), unless explicit same-turn work is still pending. Once the gate closes, [`has_pending_input`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/input_queue.rs#L438-L458) does not request another sampling step just because mail is waiting. Queue-only delivery also [does not normally start an idle turn](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/handlers.rs#L88-L96), with the durable-sleep exception described earlier.

That is why our request tells the parent to wait before concluding. A late child result should not be assumed to reopen a completed answer.

### Reusing the same child

If the findings leave a related question, the parent can send `followup_task` to `/root/trace_callers`. It reuses the conversation and requests another turn. A child [unloaded under residency pressure](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/agent/control/residency.rs#L215-L262) can be [loaded again for that follow-up](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/agent/control/api.rs#L141-L169). The child remains available for related work after completing its turn.

Our investigation can now end with both sets of findings in the parent's conversation. The call paths below collect the boundaries we crossed.

## The call paths

The main definitions behind our example are [`handle_spawn_agent`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/handlers/multi_agents_v2/spawn.rs#L102), [`prepare_agent_spawn_config`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/agent/child_config.rs#L52), [`LocalAgentControl::spawn`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/agent/control/api.rs#L58), and [`spawn_agent_internal`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/agent/control/spawn.rs#L676). Startup hands the child a message through its submission queue; its regular task then runs the model loop.

```text
parent model emits spawn_agent
└─ handle_spawn_agent
   ├─ prepare_agent_spawn_config
   └─ LocalAgentControl::spawn
      └─ spawn_agent_internal
         ├─ check capacity; reserve residency and registry slots
         ├─ create child conversation
         └─ submit initial message with TriggerTurn
            ↳ child's submission queue
               └─ inter_agent_communication
                  └─ pending-work scheduler → RegularTask → run_turn
```

The return path starts with [`maybe_notify_parent_of_terminal_turn`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/mod.rs#L2463), then [`notify_parent_of_terminal_turn`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/agent/control/completion.rs#L27) and [`format_inter_agent_completion_message`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session_prefix.rs#L20). Sending and consuming are separate asynchronous steps:

```text
child turn completes
└─ maybe_notify_parent_of_terminal_turn
   └─ AgentControl::turn_finished
      └─ notify_parent_of_terminal_turn
         ├─ format FINAL_ANSWER from child's last assistant message
         └─ send result to parent with trigger_turn = false
            ↳ parent's mailbox
               └─ next accepted step records mail into history
                  └─ parent model request includes the result
```

These paths describe model-requested V2 delegation. For comparison, the [older V1 wait schema](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/tools/handlers/multi_agents_spec.rs#L519-L553) returns agent statuses in the wait tool output. The V2 schema returns the wait summary separately from the mailbox result.

## What I would use this for

I would start with investigations like this one: give the child a precise question and ask for a concise answer with source references, while the parent handles a different piece of the problem. That fits both the separate conversation histories and the shared workspace. It also follows the [official guidance to start with independent, read-heavy tasks](https://learn.chatgpt.com/docs/agent-configuration/subagents#why-subagent-workflows-help).

For implementation work, I would add explicit file ownership and a validation requirement to each task. And I would make the expected handoff clear: findings in the final answer, edits in named files, or a report at a requested path. The runtime supplies the conversations and delivery mechanism; the task still needs to tell the child what a useful result looks like.
