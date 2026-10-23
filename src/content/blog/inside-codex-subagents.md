---
title: "CodeX Architect Part 3 - Subagents"
description: "Inside Codex's multi-agent V2 implementation: how the model delegates, what a child inherits, and how separate conversations coordinate through shared files and mailboxes."
pubDate: 2026-10-23
heroImage: "../../assets/codex-subagents-hero.png"
---

In the subagent pattern, a main agent delegates a focused task to a child agent. The parent can continue independent work while the child works, then use the returned findings in its answer.

In [CodeX Architect Part 1 - Context Engineering](/blog/inside-codex-context-engineering/), we followed the model loop of one Codex conversation. Delegation branches that flow: the parent calls `spawn_agent` to request a child conversation with its own model loop. The tools and messages connecting parent and child differ between Codex's V1 and V2 protocols, so we start by comparing them.

*(Source note: we inspect the local implementation at [commit `221e4e7` (October 16, 2026)](https://github.com/openai/codex/tree/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f). Code excerpts are verbatim. Tool arguments and the returned findings are illustrative.)*

## Table of Contents

- [Codex's V1 and V2 protocols](#codexs-v1-and-v2-protocols)
- [1 Delegation begins in the parent's model loop](#1-delegation-begins-in-the-parents-model-loop)
- [2 Choosing the child's starting context](#2-choosing-the-childs-starting-context)
- [3 Inheriting configuration and applying a role](#3-inheriting-configuration-and-applying-a-role)
- [4 Starting the child's own model loop](#4-starting-the-childs-own-model-loop)
- [5 Sending messages between conversations](#5-sending-messages-between-conversations)
- [6 Returning findings and waiting for them](#6-returning-findings-and-waiting-for-them)
- [7 Managing capacity and child lifetimes](#7-managing-capacity-and-child-lifetimes)
- [What stands out to me](#what-stands-out-to-me)

## Codex's V1 and V2 protocols

Codex has two generations of its multi-agent protocol: the tools and message conventions used to coordinate agents. **V1 and V2 refer to that protocol, not releases of Codex itself.** Both create child conversations that run the normal Codex model loop. V2 uses a mailbox, a queue of incoming messages for each conversation, and changes how agents are addressed and managed.

| Dimension | V1 | V2 |
| --- | --- | --- |
| Addressing | Thread IDs. | Task paths, such as `/root/audit`. |
| Initial history | Fresh by default; [`fork_context`](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/tools/handlers/multi_agents_spec.rs#L610-L631) requests a copy. | Copies parent history by default; [`fork_turns`](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/tools/handlers/multi_agents_v2/spawn.rs#L214-L243) can start fresh. |
| Sending more work | [`send_input`](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/tools/handlers/multi_agents_spec.rs#L269-L558), optionally interrupting. | Separate [`send_message` and `followup_task`](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/tools/spec_plan.rs#L1488-L1507) modes. |
| Reading results | [`wait_agent`](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/tools/handlers/multi_agents/wait.rs#L1-L280) returns agent statuses and completed messages. | Completion arrives through the mailbox; [`wait_agent`](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/tools/handlers/multi_agents_v2/wait.rs#L141-L174) reports activity. |
| Child lifecycle | [`close_agent` and `resume_agent`](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/tools/spec_plan.rs#L1537-L1615) tools. | Automatic idle unloading and reloading, plus `interrupt_agent`. |
| Default nesting | [Depth one](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/config/mod.rs#L256-L266): children do not receive delegation tools. | [Nested delegation](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/tools/spec_plan.rs#L728-L740) allowed when the child's model supports V2. |

Which protocol a conversation uses depends on [configuration and the model catalog](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/config/mod.rs#L1641-L1668). Enabling `features.multi_agent_v2` forces V2. That [override defaults to off](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/features/src/lib.rs#L1473-L1484), but a model's catalog entry can still select V2.

We focus on **V2** to inspect the newer coordination design: child answers arrive independently of wait calls, and the runtime can unload and reload idle agents. We will follow a delegated task from the parent's model loop through child startup, communication, and the returned answer.

## 1 Delegation begins in the parent's model loop

As we saw in the context-engineering post, [`run_turn`](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/session/turn.rs#L166-L198) repeatedly builds a model request, processes its response and tool calls, and decides whether another model step is needed. A request to spawn a subagent comes from the model during this loop. The runtime then handles configuration, startup, and communication.

Before the model can make that choice, [`spec_plan::add_collaboration_tools`](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/tools/spec_plan.rs#L1438-L1536) registers the tools exposed for the step. With V2's native tools, direct messaging, and waiting enabled, the set is:

| Tool | Purpose |
| --- | --- |
| `spawn_agent` | Create a child conversation and give it a task. |
| `send_message` | Queue a message to an existing agent. |
| `followup_task` | Give an existing child more work and request a turn. |
| `wait_agent` | Wait for mailbox activity, user input, or a timeout. |
| `list_agents` | Inspect the agent tree and statuses. |
| `interrupt_agent` | Interrupt an agent's current work. |

Tool availability and delegation guidance are separate. [`effective_multi_agent_mode`](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/session/multi_agents.rs#L77-L120) selects a developer message: a configured hint takes precedence over the model catalog's hint; without either, Ultra reasoning effort selects proactive guidance and other efforts select explicit-request guidance. The [bundled explicit-request instruction](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/prompts/src/model_messages/multi_agent.rs#L46-L47) asks the model to delegate only when the user or applicable `AGENTS.md` or skill instructions request it. This is prompt guidance; the runtime's capacity and configuration checks happen when a tool is called.

For this walkthrough, assume we explicitly ask Codex to use a subagent for an independent code investigation. We follow the model's choice to start one child, while the parent continues other work.

The consolidation worker in [CodeX Architect Part 2 - Memory](/blog/inside-codex-memories/) was launched by the runtime. Here, the parent model requests the child through `spawn_agent`.

## 2 Choosing the child's starting context

The model's `spawn_agent` call names the task, supplies its instructions, and chooses whether to copy parent history. Here are illustrative logical arguments using the [built-in spawn schema](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/tools/handlers/multi_agents_spec.rs#L110-L163):

```json
{
  "task_name": "review_auth",
  "message":
    "Check src/auth for missing auth guards. Return file:line findings. No edits.",
  "fork_turns": "none"
}
```

*(Task text is shown for readability. The [native V2 schema marks it as encrypted](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/tools/handlers/multi_agents_spec.rs#L649-L683), and [the handler distinguishes model-tool messages from direct plaintext input](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/tools/handlers/multi_agents_v2.rs#L54-L66).)*

The message is enough to explain this independent task, so we choose `"none"`. The child starts without the parent's conversation history. If earlier requirements or decisions were necessary, `"all"` would select a full-history fork. **Omitting `fork_turns` defaults to `"all"`.** These branches come directly from [`SpawnAgentArgs::fork_mode`](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/tools/handlers/multi_agents_v2/spawn.rs#L214-L243):

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

The numeric branch is compatibility handling: a positive number also selects `FullHistory`; it does not copy only the last N turns. And a full fork is still prepared for a child: [the fork preparation code](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/agent/control/spawn.rs#L1022-L1240) filters or rebuilds context items as needed. The optional `preserve_fork_prefix` setting changes that preparation and defaults to false.

Starting fresh only controls copied conversation history. The child still receives parent-derived configuration and an inherited environment. That distinction determines which model, instructions, tools, and permissions it can use.

## 3 Inheriting configuration and applying a role

[`child_config::prepare_agent_spawn_config`](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/agent/child_config.rs#L52-L100) builds the child's configuration from the spawning step. It takes the parent's effective settings, applies model and role choices, and reapplies the live runtime policy before handing the result to the agent controller.

| Layer | What it contributes |
| --- | --- |
| [Parent's captured settings](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/agent/child_config.rs#L110-L164) | Parent-derived configuration, the step's selected model and reasoning settings, and base instructions. |
| [Requested model and effort](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/agent/child_config.rs#L204-L260) | Spawn arguments override corresponding configured subagent defaults. |
| [Selected role](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/agent/role.rs#L51-L127) | Allowed instruction and model overrides, plus selected capability restrictions. |
| [Live runtime policy](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/agent/child_config.rs#L179-L201) | Parent's current approval policy, approval reviewer, working directory, and permission-profile snapshot. |

One detail in the model-selection layer: when a model override is supplied without an effort override from either source, Codex uses the selected model's default effort. A role can then replace model settings.

Which role or model arguments the model actually sees depends on configuration and catalog-supplied tool definitions; [the built-in schema builder](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/tools/handlers/multi_agents_spec.rs#L110-L163) can hide those fields. Our example omits them and uses the default role.

### A role is a preset for the same engine

Roles give the parent a vocabulary for choosing specialized work. [`role::spawn_tool_spec::build`](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/agent/role.rs#L273-L338) assembles their descriptions into an available-role menu. The [built-in declarations](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/agent/role.rs#L345-L384) include `default`, `explorer` for focused codebase questions, and `worker` for implementation work. A role can also supply a configuration file; it does not introduce a different execution engine.

The important boundary is what that file may change. [`apply_role_to_config_inner`](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/agent/role.rs#L69-L127) projects it onto a small set of allowed overrides. It can change instructions and model settings, and disable selected features or skills. Sandbox, provider, and MCP-server settings are outside that override set. The [`apply_role_cannot_expand_parent_authority` test](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/agent/role_tests.rs#L431-L517) feeds in a role requesting broader permissions, a different provider, and an extra server, then checks that those settings still equal the parent's.

This also means a role's `sandbox_mode = "read-only"` does not create a tighter sandbox at this revision: that field is not applied. Likewise, the `"No edits"` in our task is an instruction to the model. Enforced filesystem restrictions come from the inherited permission policy.

The child now has a task and an effective configuration. The next step creates the conversation that will execute them.

## 4 Starting the child's own model loop

[`spawn::handle_spawn_agent`](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/tools/handlers/multi_agents_v2/spawn.rs#L94-L195) parses the tool call and delegates to [`thread_spawn::spawn`](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/thread_spawn.rs#L58-L167). This shared function prepares configuration, constructs a path such as `/root/review_auth`, records the parent relationship, and calls the agent backend with the initial message in `TriggerTurn` mode.

For the local backend, [`LocalAgentControl::spawn`](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/agent/control/api.rs#L58-L98) calls [`spawn_agent_internal`](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/agent/control/spawn.rs#L684-L1006), which checks capacity, reserves startup slots, creates the child thread, and submits the initial message. The child's [communication handler](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/session/handlers.rs#L82-L97) queues that message. Its [pending-work scheduler](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/tasks/mod.rs#L471-L561) then starts a regular turn through `Session::start_task`.

The actual independent task launch is this line in [`Session::start_task`](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/tasks/mod.rs#L288-L423):

```rust {1}
let handle = tokio::spawn(storage_originator.scope(task_future));
```

Inside that task, [`RegularTask::run`](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/tasks/regular.rs#L40-L114) calls the same `run_turn` used by the parent. A Codex **thread** here is a managed conversation with its own history and execution. The child uses the normal model loop and its available tools to carry out its task.

Startup is guarded. [`PendingSpawn`](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/agent/control/spawn_guard.rs#L15-L105) owns the provisional child and schedules teardown if the spawn is cancelled or fails before the controller commits it. [The controller keeps that ownership across its final awaits](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/agent/control/spawn.rs#L945-L976), then commits and disarms the guard before returning to its backend caller. Publishing the tool result happens outside that guarded boundary.

With [metadata hidden by default](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/config/mod.rs#L1399-L1415), the spawn tool returns the child's canonical task path:

```json
{ "task_name": "/root/review_auth" }
```

That return means startup and initial-input admission succeeded. It does not mean the investigation finished. Parent and child can now make separate model requests and tool calls. Their conversation histories diverge, but the local path passes the [spawning step's environment](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/thread_spawn.rs#L58-L167) into child startup; the [bundled V2 guidance](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/prompts/src/multi_agent_instructions.rs#L11-L17) explicitly describes a shared working directory and filesystem. Delegated edits therefore become visible to the other agents without a merge step.

The diagram collects this successful path. Function names link to their definitions; `//` comments describe their role. It condenses startup checks and follows the fresh-history branch.

<a href="/codex-subagent-task-flow.svg">
  <picture>
    <source media="(max-width: 600px)" srcset="/codex-subagent-task-flow-mobile.svg" />
    <img src="/codex-subagent-task-flow.svg" alt="Codex V2 delegation: the spawn handler and shared spawn function prepare a child; parent and child run independent conversation loops in a shared workspace; successful completion queues the child's final answer for the parent's next accepted model step." />
  </picture>
</a>

_[Open the delegation workflow at full size.](/codex-subagent-task-flow.svg)_

Separate conversations need a way to exchange information while they run. That is the mailbox's job.

## 5 Sending messages between conversations

To send a message, an agent names the recipient's task path. [`AgentPath::resolve`](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/protocol/src/agent_path.rs#L59-L72) interprets a relative target beneath the sender's own path, and the controller [resolves that path to a thread ID](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/agent/control/target.rs#L45-L60). From `/root`, `review_auth` means `/root/review_auth`. A sibling sending to that same child uses the absolute path `/root/review_auth`.

[`handle_message_string_tool`](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/tools/handlers/multi_agents_v2/message_tool.rs#L42-L101) handles both `send_message` and `followup_task`. Their delivery modes differ:

| Tool | Mode | Effect on an idle recipient |
| --- | --- | --- |
| [`send_message`](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/tools/handlers/multi_agents_v2/send_message.rs#L33-L49) | `QueueOnly` | Queues mail; ordinarily leaves it idle. |
| [`followup_task`](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/tools/handlers/multi_agents_v2/followup_task.rs#L33-L49) | `TriggerTurn` | Requests a turn to process the task. |

For a loaded recipient, the controller [submits `Op::InterAgentCommunication`](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/agent/control.rs#L257-L325) to that thread. Its [`handlers::inter_agent_communication`](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/session/handlers.rs#L82-L97) enqueues the message and checks whether to wake an idle conversation:

```rust {1,3,6,7}
let trigger_turn = communication.trigger_turn;
sess.input_queue
    .enqueue_mailbox_communication(communication, start_options)
    .await;
crate::agent_communication::emit_agent_communication_receive(&sub_id);
if trigger_turn || sess.has_outstanding_durable_sleep() {
    sess.maybe_start_turn_for_pending_work_with_sub_id(sub_id)
        .await;
}
```

The durable-sleep check is an exception to the usual queue-only behavior: it lets incoming mail wake an agent that has entered this runtime-supported sleep state. Otherwise, a message to an idle child stays queued until a turn processes it; a follow-up task requests that turn.

For a running recipient, the [input queue drains accepted mail](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/session/input_queue.rs#L400-L431) into a model step. [`Session::record_inter_agent_communication`](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/session/mod.rs#L4048-L4143) records it in conversation history and persistence, so a subsequent model request can include it. The sender's successful tool call confirms acceptance, not that the recipient has read or acted on the message.

This is also the route the child's findings take back to the parent. The difference is who initiates the send: normal completion triggers it in runtime code.

## 6 Returning findings and waiting for them

When a child successfully completes its turn, [`agent_status_from_event`](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/agent/status.rs#L6-L31) produces `Completed(last_agent_message)`. [`Session::maybe_notify_parent_of_terminal_turn`](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/session/mod.rs#L2505-L2573) recognizes terminal events from V2 spawned children and forwards the outcome to [`LocalAgentControl::notify_parent_of_terminal_turn`](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/agent/control/completion.rs#L27-L141). The controller formats a message for the direct parent and sends it with `trigger_turn: false`.

The normal completion payload is the child's last assistant message. Its searches, command output, and reasoning are not bundled into that handoff. In [`format_inter_agent_completion_message`](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/session_prefix.rs#L20-L37), successful text is copied in full, while error text is truncated:

```rust {2,5,10}
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

The [completion template](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/context/inter_agent_completion_message.rs#L23-L46) renders an assistant-role mailbox message like this, with illustrative findings in the payload:

```text
Message Type: FINAL_ANSWER
Task name: /root
Sender: /root/review_auth
Payload:
<findings with file and line references>
```

`Task name` identifies the recipient; `Sender` identifies the child. If the task asks for a report file, the child can write it in the shared workspace and return a path. The automatic completion mechanism itself copies text, and this formatter places no length cap on a successful answer. Asking for concise, self-contained findings matters because that answer will occupy the parent's context.

### Waiting observes activity; the answer arrives separately

The parent can continue independent work after spawning. When it needs the child's findings, [V2's `wait_agent`](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/tools/handlers/multi_agents_v2/wait.rs#L40-L132) subscribes to activity on the parent's input queue. It takes no child ID and does not wait on an “all children finished” condition. Mail, new user input, or a timeout can end the wait. The [default timeout is 30 seconds](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/config/mod.rs#L256-L266), with a 10-second minimum and a one-hour maximum; those settings are configurable.

For mailbox activity, the [tool result](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/tools/handlers/multi_agents_v2/wait.rs#L141-L174) is:

```json
{ "message": "Wait completed.", "timed_out": false }
```

The `FINAL_ANSWER` is a separate message. A completed wait therefore tells the parent there is activity to inspect, not that a particular child succeeded. The parent's [`run_turn` continuation check](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/session/turn.rs#L581-L604) considers both model-requested follow-up and accepted pending input; the next step records delivered mail into history before constructing another model request.

### What if the parent finishes first?

[Final-answer output can close a delivery gate](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/stream_events_utils.rs#L122-L134), [deferring queue-only mail to the next turn](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/session/input_queue.rs#L322-L343). Once that delivery gate closes, [waiting mail alone does not request another model step](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/session/input_queue.rs#L438-L458). Because completion uses queue-only delivery, it also ordinarily leaves an idle parent idle. A late child answer should not be assumed to reopen a finished response; the parent needs to wait while the result is still required.

Ordinary interruption and budget exhaustion are another case: they map to `Interrupted`, which the [terminal-result filter excludes](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/agent/status.rs#L6-L31). The parent should inspect status with `list_agents` rather than treating a wait timeout as completion. There is a [separate notification for repeated Guardian approval denials](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/session/mod.rs#L2505-L2573). Even for a terminal result, [delivery is best effort](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/agent/control/completion.rs#L27-L141): the controller records whether queueing succeeded and logs a failure instead of retrying it here.

## 7 Managing capacity and child lifetimes

The runtime limits how much delegated work can run at once. The [V2 concurrency default is four including the root](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/config/mod.rs#L256-L266); [`effective_agent_max_threads` subtracts one](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/config/mod.rs#L1670-L1683), leaving three child execution slots. The [execution limiter](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/agent/control/execution.rs#L33-L92) checks capacity for V2 subagent work. This limits active work, rather than the total number of task paths that can exist over a conversation's lifetime.

Idle children can remain available for related work. [The residency manager](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/agent/control/residency.rs#L215-L262) can unload eligible idle agents to make room, and [a later `followup_task` reloads the child](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/agent/control/api.rs#L122-L169) and requests another turn. For locally evicted children, queue-only mail can instead [wait in a runtime mailbox](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/agent/control.rs#L257-L325) without forcing a reload. [Pending turn-triggering mail and outstanding durable sleep](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/agent/control/residency.rs#L313-L327) prevent idle eviction.

Children can also delegate. [The V2 tool gate](https://github.com/openai/codex/blob/221e4e7c05386d06aa7b91da2a9b04f7f2bd921f/codex-rs/core/src/tools/spec_plan.rs#L728-L740) has no nesting-depth check, but requires a nested agent's model to advertise V2 support before exposing the native collaboration tools. The same tree's capacity limits still apply; nesting does not create a separate allowance of child slots.

## What stands out to me

I find the reuse of Codex's conversation engine the most interesting part. Once startup has prepared its context and policy, a child runs the same model loop as its parent. The additional machinery sits around those loops: task paths, capacity reservations, message queues, and lifecycle management.

The parent receives the answer the child chose to return, while file changes are shared directly. That makes the task boundary important: a child needs a clear question or file ownership, and its answer needs to contain what the parent will use. More agents add useful capacity when those boundaries are clear; they also add coordination work that the prompts and runtime have to handle together.
