---
title: "Part 1: Context Engineering within CodeX"
description: "The first article in a series exploring CodeX's architecture: how instructions, workspace context, conversation history, and tools shape a model request."
pubDate: 2026-10-03
heroImage: "../../assets/codex-context-hero.png"
---

After learning that Codex is open source, I started reading its Rust code to understand how it works. This is the first post in a series sharing what I learned.

We will follow [commit `c248f6d` (September 29, 2026)](https://github.com/openai/codex/tree/c248f6d48b97eb4a2aa56147a0b11b7d763278b9) to trace how Codex turns a user message into a model request: which instructions it selects, what context it adds, and how it sends the conversation history and tools. The exact request depends on the model, configuration, enabled extensions, and previous turns.

## 1 The path from a user message to the model

At a high level, Codex records the user's message and the context for the turn, prepares the conversation history, then combines it with the session's base instructions and available tools. The request builder chooses how to send those pieces based on the model. In this commit, [`gpt-6-sol` uses Responses Lite](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/models-manager/models.json#L178-L197): base instructions and tools become developer items in `input`. The other branch uses separate `instructions` and `tools` fields. The diagram shows one model request; a turn can repeat the path after a tool call.

<a href="/codex-model-context-flow.svg">
  <picture>
    <source media="(max-width: 600px)" srcset="/codex-model-context-flow-mobile.svg" />
    <img src="/codex-model-context-flow.svg" alt="Codex user-to-model workflow: record a user message and context, prepare history, join the session's base instructions and tools, then build a model request. Responses Lite places base instructions and tools in developer input items; the other branch uses separate request fields." />
  </picture>
</a>

_[Open the request flow diagram at full size.](/codex-model-context-flow.svg)_

The function name heads each code step; the line beneath says what that function contributes. The three arrows into `build_prompt` represent prepared history, base instructions already selected for the session, and tool definitions. After `ModelClient::build_responses_request`, the two boxes show the model-dependent request shape. This is the logical request assembly path: input routing, optional compaction, context injections, and final transport preparation are condensed. WebSocket can send only the new input items when it can reuse a previous response. We will trace those details and the tool-result loop below.

## 2 Which base instructions does Codex use?

The [model manager bundles a catalog](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/models-manager/src/lib.rs#L11-L17) from [`models.json`](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/models-manager/models.json). Each of its ten model entries at this commit has a `model_messages.instructions_template` value. Depending on the provider and discovery settings, the [manager can refresh the catalog](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/models-manager/src/manager.rs#L540-L581) from the model endpoint and [cache it under the Codex home directory](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/models-manager/src/manager.rs#L304-L322) in a file named [`models_cache.json`](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/models-manager/src/manager.rs#L34). The active template therefore depends on the selected model and the catalog available to that run.

### What the model template tells Codex

At this commit, the `gpt-6-sol` entry offers a concrete example. Its [18,992-character template](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/models-manager/models.json#L249) begins:

```md
You are Codex, an agent based on GPT-6. You and the user share one workspace, and your job is to collaborate with them until their intended goal is completely handled.
```

The rest is structured as Markdown. These sections show the kinds of behavior the template establishes:

| Section | What it asks the agent to do |
| --- | --- |
| `# Personality` and `## Writing style` | Explain technical work clearly, like a colleague; use direct language and one point per paragraph. |
| `# When to ask the user for permission` | Use existing authorization and complete reviewable work before asking for approval. |
| `# Autonomy and persistence` | Finish the authorized task and make progress on reversible work without repeatedly asking for approval. |
| `# Working with the user` | Treat new messages as steering the active task; continue from compaction summaries. |
| `## Intermediate commentary` | Share progress while working and keep user-facing questions out of progress updates. |
| `## Final answer` | Give a self-contained response with clickable file links and diagrams when they help explain the result. |
| `# Rules for getting work done` | Prefer `rg` for searches, take care with shell commands, and run meaningful checks. |
| `# Using skills`, `# Apps (Connectors)`, `# Plugins` | Explain when and how to use the extensions made available to the session. |

One revealing instruction is about compaction: “Compaction does not end the task.” The same section tells the agent to continue from a summary without restarting finished work. This does not perform compaction by itself; the [runtime compacts history](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/compact.rs#L345-L373), and the template tells the model how to behave afterward. A [local compaction request](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/compact.rs#L265-L292) also uses the session's current base instructions.

This template describes how the agent should work. It does not contain this session's working directory, sandbox rules, `AGENTS.md` contents, or actual list of skills. Codex adds those details through other context paths, which we will trace next.

### How Codex selects the base text

When a session starts, [`Session::spawn_internal`](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/session/mod.rs#L713-L748) checks these sources in order and uses the first one available:

1. An explicit override from configuration, including [`model_instructions_file`](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/config/mod.rs#L3989-L4005).
2. Instructions saved with the conversation being resumed or forked.
3. The selected model's instruction template.

Here is that selection in the code. Line breaks in the excerpts below are adjusted to fit the page:

```rust
let base_instructions = config
    .base_instructions
    .clone()
    .or_else(|| conversation_history
        .get_base_instructions()
        .map(|s| s.text))
    .unwrap_or_else(||
        render_model_instructions(&model_info)
    );
```

[`render_model_instructions`](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/prompts/src/model_instructions.rs#L8-L18) reads that last option from the selected model's `instructions_template`. If a template is missing, this function logs a warning and returns empty instructions. A [`model_instructions_file` override replaces the base text](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/config/mod.rs#L3989-L4005); [`AGENTS.md` enters separately as project context](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/context/user_instructions.rs#L10-L35). When no base-instructions override is set, [personality `none`](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/models-manager/src/model_info.rs#L42-L58) removes the template's personality section. [`Session::get_prompt_base_instructions`](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/session/mod.rs#L1489-L1507) can also remove `update_plan` guidance from the request copy under specific settings.

The [Markdown `default.md` prompt](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/protocol/src/prompts/base_instructions/default.md) I first found backs [`BaseInstructions::default`](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/protocol/src/models.rs#L1535-L1565). An identical [`models-manager/prompt.md`](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/models-manager/prompt.md) supplies [fallback model metadata](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/models-manager/src/model_info.rs#L95-L157) for an unknown model slug. Those files are useful to read, but neither is the standard template for every listed model.

## 3 Runtime context and available tools

Base instructions are only one part of what the model receives. Before each regular sampling step, [`Session::build_world_state_for_step`](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/session/world_state.rs#L40-L337) gathers current instructions and environment details. Other paths add messages when a turn starts or an extension is active. The main sources I found are:

- **Instructions and operating mode.** Codex can add [configured](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/session/mod.rs#L4316-L4328) or [managed](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/session/world_state.rs#L326-L336) developer instructions, [permission guidance](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/session/world_state.rs#L164-L204), [collaboration guidance](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/session/world_state.rs#L204-L213), [persistent-mode guidance from a separate catalog field](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/session/world_state.rs#L210-L226), and [V2 multi-agent guidance](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/session/world_state.rs#L315-L326) when applicable.
- **Workspace and environment.** Codex adds the [loaded `AGENTS.md` instructions](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/session/world_state.rs#L160-L164). If environment context is enabled, it may also describe the [working directory, shell, date, timezone, filesystem and network rules, and active subagents](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/context/world_state/environment.rs#L27-L128).
- **Skills and integrations.** Depending on the setup, the model can receive an [available-skills catalog](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/ext/skills/src/fragments.rs#L11-L64), [selected skill instructions](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/session/turn.rs#L1075-L1118), or [app, plugin, and deferred-tool guidance](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/session/world_state.rs#L251-L287).
- **Tool definitions.** Codex builds the [tool list available to the model](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/tools/spec_plan.rs#L477-L493) separately. It can include [built-in, MCP, and dynamic tools](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/tools/spec_plan.rs#L150-L174).
- **Extensions and special modes.** [Context contributors](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/session/mod.rs#L4353-L4388) can add instructions; [turn-input contributors](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/session/turn.rs#L1177-L1225) can add messages. Codex can also add [model-switch, token-budget, or realtime guidance](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/session/world_state.rs#L117-L162) when those features apply.

The instruction and environment pieces become messages; tool definitions join them when Codex builds the request. Among the messages, the role matters. In this commit, Codex renders [`AGENTS.md` in a user-role context message](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/context/user_instructions.rs#L10-L35), while [multi-agent guidance uses a developer-role message](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/context/multi_agent_mode_instructions.rs#L25-L46). The [initial context builder](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/session/mod.rs#L4449-L4514) puts them into the conversation. On later turns, Codex [adds updates when that state changes](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/session/mod.rs#L4682-L4715). At the start of a turn, it also [records any selected skill and plugin injection items](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/session/turn.rs#L310-L321) [before the first model request](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/session/turn.rs#L390-L409).

Inside `build_initial_context_with_world_state`, the general cases sort those pieces by role:

```rust
"developer" => developer_sections
    .push(fragment.render_fragment()),
"user" => contextual_user_sections
    .push(fragment.render_fragment()),
```

Some developer fragments need their own message or a specific order; the full function handles those cases. A project instruction and a developer policy can therefore both reach the model in different message roles. [See the full match and message construction.](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/session/mod.rs#L4449-L4514)

Switching models is another reason to add context. The session can keep its original base text while [`ModelInstructionsState::render_diff`](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/context/world_state/model.rs#L44-L60) adds the new model's instructions in a `<model_switch>` developer message. Reading only the session's base text would miss that update.

## 4 The user message joins conversation history

Before the turn runner sees a request, core routes it. [`SessionIo::submit_turn_input`](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/session/mod.rs#L999-L1020) sends an `Op::TurnInput` to the submission loop. For `StartOrSteer`, [`start_or_steer`](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/session/turn_input.rs#L276-L373) queues input for an active regular turn or starts a `RegularTask` when there is no active turn. [`RegularTask::run`](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/tasks/regular.rs#L40-L114) calls `run_turn`.

The text request reaches [`run_turn`](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/session/turn.rs#L163-L198) as `TurnInput::UserInput`. Codex captures the step's context and tools, records the current context updates, then [checks and records the user's input](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/session/turn.rs#L284-L400). The [recording path](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/session/mod.rs#L4963-L5023) turns that input into a message in conversation history. The conversion sets its role explicitly:

```rust
Self::Message {
    role: "user".to_string(),
    content,
    phase: None,
}
```

[Source: `ResponseInputItem::from_user_input`.](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/protocol/src/models.rs#L2004-L2084)

The model also receives a prepared version of the conversation so far. It can include earlier user and assistant messages, tool calls and results, and the context messages from the previous section. [Source: history prepared for the model.](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/context_manager/history.rs#L579-L591) [User messages](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/protocol/src/models.rs#L875-L901) and [tool results](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/protocol/src/models.rs#L2093-L2120) can contain text, images, or audio when the selected model supports them.

History can also hold [reasoning items](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/protocol/src/models.rs#L1048-L1058) and [updates to reasoning settings](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/protocol/src/models/configuration_update.rs#L7-L13), although the [request builder may filter those updates](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/client.rs#L895-L905).

When the configured compaction budget or usable context window is exhausted, Codex can [compact before recording a new turn](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/session/turn.rs#L1298-L1325) or [between requests in a continuing turn](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/session/turn.rs#L601-L650). After [summarizing compaction](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/compact.rs#L345-L373), a summary can stand in for older turns. Local compaction [retains user messages within a budget](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/compact.rs#L662-L715), which can truncate retained text and replace media with text. Compaction is conditional; a new request does not imply the full earlier transcript is still present.

For a regular sampling request, `ContextManager::for_prompt` delegates to `for_prompt_annotated`, which prepares the history with these two lines:

```rust
self.normalize_history(input_modalities);
Arc::unwrap_or_clone(self.items)
```

[`ContextManager::normalize_history`](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/context_manager/history.rs#L933-L947) first supplies missing tool outputs and removes orphaned outputs that require a matching call. It also [replaces unsupported images and audio in messages and tool outputs with text placeholders](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/context_manager/normalize.rs#L330-L420). These changes apply to the history copy prepared for the request.

As a turn proceeds, Codex may add [new user input or hook context](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/session/turn.rs#L870-L905). It can also add [time reminders](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/session/time_reminder.rs#L156-L201) or [budget reminders](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/session/rollout_budget.rs#L9-L37) when their conditions apply. Those items enter a later model call.

## 5 Building the model request

With the history prepared, [`run_sampling_request`](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/session/turn.rs#L1621-L1670) retrieves the selected base instructions and calls `build_prompt` with that history and the step context. The step context supplies the model-visible tool definitions.

These selected lines from `build_prompt` show where the current tool list joins the prepared input and base instructions. The `Prompt` enables parallel tool calls here; the final builder [sets that request flag to false for Lite models](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/client.rs#L989-L996).

```rust
input,
tools: step_context
    .tool_router
    .model_visible_specs(),
parallel_tool_calls: true,
base_instructions,
```

[`ModelClient::build_responses_request`](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/client.rs#L885-L1008) then converts that `Prompt` into a request. It first calls [`Prompt::get_formatted_input_for_request`](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/client_common.rs#L60-L117), which adjusts image-detail settings for the model. It then chooses between these two shapes:

- **Responses Lite (`gpt-6-sol` at this commit):** Tool definitions and nonempty base text become developer items prepended to `input`. The top-level `instructions` string is empty and `tools` is unset.
- **Non-Lite:** Prepared history stays in `input`; base text goes in `instructions` and tool definitions go in `tools`.

After that choice, the builder assigns the request fields:

```rust
model: model_info.slug.clone(),
instructions,
input,
tools,
tool_choice: "auto".to_string(),
```

[See the full `build_prompt` function.](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/session/turn.rs#L1583-L1600) The excerpts show selected lines from larger struct literals; neither is the whole request. For a Lite model, `instructions` and `tools` in the second excerpt hold the empty and unset values described above.

This explains why `AGENTS.md` can contain instructions yet appear in `input`: Codex renders it as a context message. Base text and tools follow the model's request branch. The request also carries [model, reasoning, tool-choice, and output-format settings](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/client.rs#L979-L1008) outside the conversation. To know what any particular turn received, we would need to inspect its assembled request, including the selected branch, extensions, and current history.

There is one more distinction between the assembled request and what travels over the connection. On WebSocket, [`ModelClientSession::prepare_websocket_request`](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/client.rs#L1380-L1445) checks whether the request extends the previous input and response without changing the other request settings. If it can reuse that response, the [wire payload carries `previous_response_id` and only the new input items](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/client.rs#L1994-L2020). The earlier context is reused through that response ID.

From this assembly path, I infer that Codex does not automatically send every file in a repository. Files the agent reads reach the model through added context or a user or tool message. [Source: request input is prepared from conversation history.](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/session/turn.rs#L506-L521)

### A tool result changes the next request

When Codex needs to inspect a file or run a command, the first request may produce a tool call. [`handle_output_item_done`](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/stream_events_utils.rs#L315-L357) records that call and queues its execution. After the tool finishes, [`drain_in_flight`](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/session/turn.rs#L2466-L2489) records the result in conversation history. The result is a response item associated with the call, so the model can use its content in the next request. [Source: tool output conversion.](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/tools/registry.rs#L191-L215)

That result may already be shorter than the tool's original output. [`ContextManager::record_item_with_metadata`](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/context_manager/history.rs#L532-L555) applies a truncation budget to function and custom-tool outputs as they enter live history. A successful file read therefore does not guarantee the model receives the entire file.

The main [`run_turn` loop](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/session/turn.rs#L427-L566) then prepares history again. It records accepted input that arrived during the turn, reuses or captures the step's context and tools, and records any world-state changes before preparing the request. The history preparation call is below, with line breaks adjusted for the page:

```rust
sess.clone_history()
    .await
    .for_prompt(
        &step_context.settings
            .model_info
            .input_modalities,
    )
```

This is the point where a tool result, an accepted user steer, or a changed context message can affect the next model call. The model's [follow-up flag and pending input](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/session/turn.rs#L540-L566) determine whether another sampling step is needed. When neither requests continuation, Codex [runs Stop hooks before finishing](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/session/turn.rs#L653-L701); a hook can add a prompt and keep the loop going.

### The call tree

Here is the core submission path for `StartOrSteer`. The submission channel and spawned task connect the first two groups; they run asynchronously:

```text
SessionIo::submit_turn_input  session/mod.rs:999
└─ SessionIo::submit_with_id  session/mod.rs:984
   ↳ submission_loop  session/handlers.rs:420
      └─ turn_input::handle  session/turn_input.rs:205
         └─ start_or_steer  session/turn_input.rs:276
            ├─ Session::steer_input  session/turn_input.rs:632
            │  (queue accepted input for the active regular turn)
            └─ Session::spawn_task  tasks/mod.rs:271
               └─ Session::start_task  tasks/mod.rs:286
                  ↳ RegularTask::run  tasks/regular.rs:40
                     └─ run_turn  session/turn.rs:163
```

Inside `run_turn`, the main sampling path looks like this. It includes initial context, user-message recording, history normalization, both transport choices, and a tool result. It condenses task wrappers, metadata preparation, error handling, and special modes. File references are relative to `codex-rs/core/src/`, except for `protocol/src/models.rs` and the transport calls marked `codex-api`:

```text
run_turn  session/turn.rs:163
├─ run_pre_sampling_compact  session/turn.rs:1298
│  (if needed, compact before recording the new user input)
├─ Session::capture_step_context_with_required_mcp_servers
│  (session/mod.rs:3761; capture model, environment, and tools)
├─ Session::record_context_updates_and_set_reference_context_item
│  (session/mod.rs:4677)
│  ├─ Session::build_world_state_for_step  session/world_state.rs:40
│  └─ Session::build_initial_context_with_world_state
│     (session/mod.rs:4301)
│     (full context when needed; otherwise record changes)
├─ build_skills_and_plugins  session/turn.rs:1032
│  (collect turn-specific injection items)
├─ run_hooks_and_record_inputs  session/turn.rs:866
│  └─ record_pending_input  hook_runtime.rs:716
│     └─ Session::record_user_prompt_and_emit_turn_item
│        (session/mod.rs:4963)
│        └─ Session::response_item_from_user_input_with_image_positions
│           (session/mod.rs:3495)
│           └─ ResponseInputItem::from_user_input
│              (protocol/src/models.rs:2004)
├─ Session::record_conversation_items  session/mod.rs:3538
│  (record collected skill, plugin, or extension items)
├─ Session::record_step_world_state_if_changed
│  (session/mod.rs:3684; refresh context for this sampling step)
│  └─ Session::build_world_state_for_step  session/world_state.rs:40
├─ ContextManager::for_prompt  context_manager/history.rs:581
│  └─ ContextManager::for_prompt_annotated
│     (context_manager/history.rs:589)
│     └─ ContextManager::normalize_history
│        (context_manager/history.rs:933; repair call pairs, filter media)
└─ run_sampling_request  session/turn.rs:1612
   ├─ Session::get_prompt_base_instructions  session/mod.rs:1489
   ├─ build_prompt  session/turn.rs:1583
   │  └─ ToolRouter::model_visible_specs  tools/router.rs:137
   └─ try_run_sampling_request  session/turn.rs:2519
      ├─ ModelClientSession::stream  client.rs:2218
      │  ├─ ModelClientSession::stream_responses_websocket
      │  │  (client.rs:1835)
      │  │  ├─ ModelClient::build_responses_request  client.rs:885
      │  │  ├─ ModelClientSession::prepare_websocket_request
      │  │  │  (client.rs:1429; full input or continuation delta)
      │  │  │  └─ ModelClientSession::get_incremental_items
      │  │  │     (client.rs:1384)
      │  │  └─ ResponsesWebsocketConnection::stream_request → model
      │  │     (codex-api/src/endpoint/responses_websocket.rs:239)
      │  └─ ModelClientSession::stream_responses_api
      │     (client.rs:1645)
      │     ├─ ModelClient::build_responses_request  client.rs:885
      │     └─ ResponsesClient::stream_request → model
      │        (codex-api/src/endpoint/responses.rs:70)
      ├─ OutputItemDone → handle_output_item_done
      │  (stream_events_utils.rs:315)
      │  ├─ record_completed_response_item
      │  │  (stream_events_utils.rs:79; record the call)
      │  └─ ToolCallRuntime::handle_tool_call  tools/parallel.rs:77
      │     └─ ToolCallRuntime::handle_tool_call_with_source
      │        (tools/parallel.rs:125)
      │        ↳ ToolRouter::dispatch_tool_call_with_state
      │           (tools/router.rs:327; dispatch to the tool registry)
      └─ drain_in_flight  session/turn.rs:2466
         └─ Session::record_annotated_conversation_items
            (session/inject.rs:112; record the result)

↳ run_turn loop → refresh context → for_prompt → next model request
```

[`ModelClientSession::stream`](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/client.rs#L2210-L2268) chooses WebSocket when available or the HTTP Responses path. Both the [WebSocket](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/client.rs#L1855-L1885) and [HTTP](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/client.rs#L1680-L1715) branches call `ModelClient::build_responses_request`; that function makes the Lite versus non-Lite choice described above. The continuation at the bottom is why context engineering happens throughout a turn: each follow-up request uses the history and available tools as they stand at that step.

One possible call in that loop is `spawn_agent`. When enabled, Codex can add [V2 multi-agent instructions](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/session/multi_agents.rs#L77-L106) and expose the [`spawn_agent` tool](https://github.com/openai/codex/blob/c248f6d48b97eb4a2aa56147a0b11b7d763278b9/codex-rs/core/src/tools/handlers/multi_agents_spec.rs#L100-L153). In Part 2, we'll follow what happens when the model calls it.
