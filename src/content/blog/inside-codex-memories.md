---
title: "CodeX Architect Part 2 - Memory"
description: "How Codex builds memory an LLM can retrieve, summarize, and update: a V1 walkthrough, a V2 comparison, and connections to its model-context pipeline."
pubDate: 2026-10-10
heroImage: "../../assets/codex-memory-hero.png"
---

I see building better memory systems as one of the key engineering efforts for pushing the frontier of AI applications. The [VISTA paper](https://arxiv.org/html/2610.02200v1#S3.SS1), coauthored by Kaiming He, describes a system that saves game frames in their original form and lets the model retrieve and inspect them during play. In the [authors' ablation on 25 public ARC-AGI-3 games](https://arxiv.org/html/2610.02200v1#S4.SS3), GPT-5.6 Sol's reported Relative Human Action Efficiency score is **70.05** with text notes and **94.10** with added visual memory and inspection tools.

In this CodeX Architect blog post, we dig into CodeX's memory pipeline: how earlier chats become reusable knowledge, how that knowledge reaches the model, and how corrections get incorporated. We will follow **version 1 (V1)** of that pipeline, the default at the source commit we are studying, then compare **version 2 (V2)**.

## Table of Contents

- [Why useful memory is hard to build](#why-useful-memory-is-hard-to-build)
- [Local memory and project instructions](#local-memory-and-project-instructions)
- [How memory gets updated](#how-memory-gets-updated)
- [1 Starting the background memory job](#1-starting-the-background-memory-job)
- [2 Extracting findings from earlier chats](#2-extracting-findings-from-earlier-chats)
- [3 Building the memory files](#3-building-the-memory-files)
- [4 Loading saved memory context into a new conversation](#4-loading-saved-memory-context-into-a-new-conversation)
- [5 Retrieving relevant memory](#5-retrieving-relevant-memory)
- [6 Tracking which memories get used](#6-tracking-which-memories-get-used)
- [7 Updating memory on request](#7-updating-memory-on-request)
- [8 Exploring V2](#8-exploring-v2)
  - [8.1 Extracting session history](#81-extracting-session-history)
  - [8.2 Building a compact summary](#82-building-a-compact-summary)
  - [8.3 Reading through direct pointers](#83-reading-through-direct-pointers)
- [What stands out in Codex's design](#what-stands-out-in-codexs-design)

## Why useful memory is hard to build

Stored information can help one task and mislead another; it can also become outdated. Keeping it useful calls for ongoing [retrieval and maintenance](https://arxiv.org/html/2606.06448v1#S1). Four engineering problems follow:

- **Finding relevant evidence.** A memory needs enough project and task context to tell us when it applies, plus a route back to its source.
- **Compressing faithfully.** Merge duplicates without losing scope, exceptions, or supporting details. This matters both when shortening an active conversation to fit its context window (**compaction**) and when combining persistent memories (**consolidation**).
- **Handling corrections and freshness.** Later feedback may supersede an earlier preference, while an old API detail may need [fresh verification](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/ext/memories/templates/memories/read_path.md#L51-L73). Frequently used information can still be wrong.
- **Writing reliably.** Concurrent updates need coordination. [Checking file structure and job ownership](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/src/phase2.rs#L406-L474) helps apply updates reliably, but does not establish that their content is true.

We can inspect these decisions in Codex's extraction, consolidation, and retrieval paths.

## Local memory and project instructions

Codex can receive persistent guidance through two sources of context:

- **Project instructions in `AGENTS.md`.** Codex loads applicable `AGENTS.md` files as [instructions](https://learn.chatgpt.com/docs/agent-configuration/agents-md), whether the memory feature is on or off. This is the instructions path we followed in our earlier blog post, [CodeX Architect Part 1 - Context Engineering](/blog/inside-codex-context-engineering/#3-describing-the-world-for-this-step).
- **Generated files in the memory folder.** With local memory enabled, Codex can extract useful information from eligible earlier chats and organize it into memory files. Later conversations can receive [a short summary and instructions for finding more detail](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/ext/memories/src/prompts.rs#L35-L64).

**When memory is enabled, both sources can contribute context to the same conversation.**

_(Source: [commit `c3d3b14` (October 10, 2026)](https://github.com/openai/codex/tree/c3d3b142d10f4316b46e35aad7e5317e7e506cb7). Rust, SQL, and prompt excerpts come from this commit; generated memory-file examples are illustrative.)_

Local memory is **off by default** at this commit because the [master switch, `features.memories`, defaults to `false`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/features/src/lib.rs#L1265-L1270). Enable it in `config.toml`:

```toml
[features]
memories = true
```

**Once the master switch is on**, two [settings default to `true`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/config/src/types.rs#L376-L394):

- [`generate_memories`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/config/src/types.rs#L330-L331) controls whether newly created chats can become future extraction inputs.
- [`use_memories`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/config/src/types.rs#L332-L333) controls whether Codex supplies saved memory context to the model.

Setting `generate_memories = false` does not itself disable background processing of other eligible chats; the [startup checks](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/src/start.rs#L33-L38) gate that pipeline on the master feature flag. The CLI's [`/memories` menu](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/tui/src/chatwidget.rs#L1090-L1103) changes the controls; opening it does not run memory generation.

### How memory gets updated

Within the memory feature, new information can enter through two paths:

- **Automatically, by processing earlier chats.** Starting a new turn with input can [launch a background job](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/app-server/src/request_processors/turn_processor.rs#L704-L717). It [selects eligible older chats and extracts useful findings](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/src/phase1.rs#L56-L94). [Sections 1–3](#1-starting-the-background-memory-job) follow this path; checks can skip or delay the work.
- **Explicitly, when we ask to remember, update, or forget something.** With memory guidance loaded, the model is [instructed to save a small update note](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/ext/memories/templates/memories/read_path.md#L117-L123). [Section 7](#7-updating-memory-on-request) covers note writing and incorporation.

The background job has two stages, called **Phase 1** and **Phase 2** in the source. Both memory versions, V1 and V2, use these stage names:

| Stage | What it does |
| --- | --- |
| **Phase 1: extraction** | Reads eligible earlier chats and [stores per-chat extraction results in the database](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/src/phase1.rs#L366-L395). |
| **Phase 2: consolidation** | Uses [selected extraction results](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/src/phase2.rs#L94-L164) and [explicit update notes](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/templates/extensions/ad_hoc/instructions.md#L4-L13) to update shared memory files for future conversations. |

The automatic path uses both stages; explicit notes enter **Phase 2 directly**. A "*Remember this*" message can also start an ordinary new turn and request the same background pipeline. But [writing the note itself](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/ext/memories/src/local/ad_hoc_note.rs#L15-L39) does not launch another job or guarantee an immediate summary update; an eligible consolidation pass must pick it up.

The diagram follows V1's **write path** from earlier chats into memory files, then its **read path** into conversation context. V1 uses the [`memories` directory](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/protocol/src/memory_version.rs#L15-L23) [under Codex home](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/ext/memories/src/prompts.rs#L35-L39); rendered prompt excerpts below use `/home/engineer/.codex` as Codex home.

<a href="/codex-memory-flow.svg">
  <picture>
    <source media="(max-width: 600px)" srcset="/codex-memory-flow-mobile.svg" />
    <img src="/codex-memory-flow.svg" alt="Codex V1 memory workflow with source function names and comments: background extraction identifies useful findings in earlier chats, consolidation organizes them into memory files, and a later conversation receives a memory summary. The model can retrieve detailed memory during a task. Citations influence future selection, and explicit update notes supply consolidation directly." />
  </picture>
</a>

_[Open the V1 memory flow at full size.](/codex-memory-flow.svg)_

Function names link to their definitions; `//` comments describe each step. We start with the turn that launches background processing.

## 1 Starting the background memory job

Our [context engineering blog post](/blog/inside-codex-context-engineering/#1-the-message-enters-the-core) explains how user input reaches the core in the embedded CLI path:

Terminal UI → `turn/start` → app-server → core input queue → `RegularTask::run` → `run_turn`

After the core [reports a newly started turn](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/app-server/src/request_processors/turn_processor.rs#L684-L702), [`TurnRequestProcessor::turn_start_inner`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/app-server/src/request_processors/turn_processor.rs#L525) can call [`start_memories_startup_task`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/src/start.rs#L24-L94). The highlighted lines at the [call site](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/app-server/src/request_processors/turn_processor.rs#L704-L717) require input, a newly started turn, and a configured primary environment:

```rust {1,3,4}
if turn_has_input && started {
    let config_snapshot = thread.config_snapshot().await;
    if config_snapshot.is_primary_environment_configured() {
        codex_memories_write::start_memories_startup_task(
            Arc::clone(&self.thread_manager),
            Arc::clone(&self.auth_manager),
            thread_id,
            Arc::clone(&thread),
            thread.config().await,
            config_snapshot.permission_profile,
            &config_snapshot.session_source,
        );
    }
}
```

This applies to **each new turn with input**, including later turns in an existing conversation. Steering an already running turn sets `started` to false and skips this call. The startup function also [returns early](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/src/start.rs#L33-L38) for disabled memories, ephemeral conversations, or child agents.

The async job launches at **[`tokio::spawn`, line 59](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/src/start.rs#L59)**. The highlighted lines below show the launch, database and quota checks, and the two stages:

```rust {1,2,21,31,33}
tokio::spawn(async move {
    if context.memory_store().await.is_none() {
        warn!("state db unavailable for memories startup pipeline; skipping");
        return;
    }
    let root = config
        .codex_home
        .join(config.memories.version.directory_name());
    if let Err(err) = ensure_layout(&root).await {
        warn!("failed preparing memories root: {err}");
        return;
    }
    if let Err(err) = seed_extension_instructions(&root).await {
        warn!("failed seeding memory extension instructions: {err}");
    }

    // Clean memories to make preserve DB size. This does not consume tokens so can be
    // done before the quota check.
    phase1::prune(context.as_ref(), &config).await;

    if !guard::rate_limits_ok(&auth_manager, &config).await {
        context.counter(
            MEMORY_STARTUP,
            /*inc*/ 1,
            &[("status", "skipped_rate_limit")],
        );
        return;
    }

    // Run phase 1.
    phase1::run(Arc::clone(&context), Arc::clone(&config)).await;
    // Run phase 2.
    phase2::run(context, config, parent_permission_profile).await;
});
```

The startup function returns without awaiting this task, so the current conversation continues independently. Inside the task, the `.await` calls order **Phase 1 extraction before Phase 2 consolidation**. The [quota guard](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/src/guard.rs#L9-L65) can stop work before extraction when known remaining Codex rate limits fall below the [default threshold of 25 percent](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/config/src/types.rs#L55-L60).

[Loading saved context](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/ext/memories/src/prompts.rs#L35-L64) does not wait for this background job to finish. A new conversation can therefore begin with the previously saved summary. We now follow how Phase 1 selects earlier chats for extraction.

## 2 Extracting findings from earlier chats

**Phase 1: extraction** selects eligible earlier sessions through [`claim_startup_jobs`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/src/phase1.rs#L120-L157). Its [database query](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/state/src/runtime/memories.rs#L138-L256) excludes the current conversation and chats that are not enabled as memory inputs.

At this commit, the [defaults](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/config/src/types.rs#L55-L60) allow up to two session claims per pass. The [query's age window](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/state/src/runtime/memories.rs#L245-L254) uses each chat's last recorded update: it must fall within the past ten days and be at least six hours ago.

The database then [checks whether each candidate session has changed since its last successful extraction](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/state/src/runtime/memories.rs#L91-L134). Unchanged sessions are skipped; a temporary claim, or **lease**, gives a worker ownership of each extraction job.

For each claimed session, [`job::sample`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/src/phase1.rs#L259-L321) loads its recorded conversation, called a **rollout** in the code, and asks an extraction model to summarize it in a specified JSON format. V1 [filters the recorded items and redacts recognized secrets](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/src/phase1.rs#L398-L425) before sending the input.

The extraction prompt [prioritizes user requests and corrections that reveal useful preferences](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/templates/memories/stage_one_system.md#L84-L119). A shortened illustrative response uses the fields required by the [V1 output schema](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/src/phase1_output.rs#L61-L72):

```json
{
  "rollout_summary":
    "Blog revision: user clarified preferred introductions, examples, and titles.",
  "rollout_slug": "blog-writing-style",
  "raw_memory":
    "Blog style: short introductions, familiar examples, and titles without Explore."
}
```

`rollout_summary` is the session recap; `raw_memory` carries candidate reusable knowledge. `rollout_slug` can help name the recap. [`StageOneOutput::parse`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/src/phase1_output.rs#L34-L58) decodes the result and redacts the generated fields too.

The [success path](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/src/phase1.rs#L366-L395) stores these values in the database for consolidation. Even if [no sessions qualify for extraction](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/src/phase1.rs#L60-L72), the background task proceeds to Phase 2, which can use previously extracted results or explicit update notes.

## 3 Building the memory files

**Phase 2: consolidation**, implemented by [`phase2::run`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/src/phase2.rs#L49-L192), combines selected findings into shared memory files. It first [claims the global job](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/src/phase2.rs#L66-L80) to coordinate writers. [An existing lease, retry delay, or cooldown](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/state/src/runtime/memories.rs#L1071-L1150) can defer the pass; the [success cooldown is six hours](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/state/src/runtime/memories.rs#L20-L25) at this commit.

After claiming the job, phase 2 [selects extraction results, writes the inputs into the memory folder, and computes a Git diff](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/src/phase2.rs#L94-L159). For V1, those inputs include candidate findings in `raw_memories.md` and session recaps in `rollout_summaries/`. The diff shows what changed since the previous consolidation. If nothing changed and the required output files are valid, the stage can finish without a model call.

If consolidation is needed, Codex starts a model-powered worker to organize the inputs. [`spawn_consolidation_agent`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/src/runtime.rs#L399-L443) creates an internal `MemoryConsolidation` conversation and submits the task. [`phase2::agent::handle`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/src/phase2.rs#L366-L384) tracks completion in a separate task, so `phase2::run(...).await` can return before the worker finishes.

With Codex-managed sandboxing, the worker's [configuration](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/src/phase2.rs#L294-L353) limits writes to the memory folder and disables network access. It preserves a parent's disabled or externally managed sandbox instead. In all three cases, it disables the worker's own memory use and generation, keeping the consolidation conversation out of future extraction.

The resulting V1 memory folder has several layers, from a compact summary to detailed evidence:

| File or folder | Purpose |
| --- | --- |
| [<code>memory_<wbr>summary.md</code>](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/templates/memories/consolidation.md#L448-L595) | A short summary of preferences and an index of available topics. |
| [`MEMORY.md`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/templates/memories/consolidation.md#L201-L245) | Detailed reusable knowledge, grouped by task or project. |
| [<code>rollout_<wbr>summaries/</code>](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/ext/memories/templates/memories/read_path.md#L19-L40) | Recaps of earlier sessions and pointers to their evidence. |
| [`skills/`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/ext/memories/templates/memories/read_path.md#L19-L40) | Optional procedures for recurring work. |

An illustrative `MEMORY.md` entry follows the [task-grouped format](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/templates/memories/consolidation.md#L206-L281); evidence metadata is omitted here:

```md
# Task Group: /work/blog writing style

scope: Drafting and revising the author's blog posts.
applies_to: cwd=/work/blog; reuse_rule=use for this author's blog drafts

## Task 1: Revise a blog draft, style preferences clarified

...

### keywords

- blog, writing-style, introductions, examples, titles

## User preferences

- For blog drafts, the user asked to "keep the introduction short". [Task 1]
- Use concrete, familiar examples. [Task 1]
- Avoid "Explore" in blog titles. [Task 1]
```

Scope and keywords guide the model's retrieval; they do not isolate memory by project in the runtime.

After the worker finishes, the [completion handler](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/src/phase2.rs#L406-L474) validates its output and checks that it still owns the lease before saving the new workspace baseline. [V1 validation](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/src/workspace.rs#L76-L114) checks file structure, including the summary's `v1` header, and rejects symbolic links. It does not test the factual accuracy of each entry.

Once consolidation completes, the generated memory files are available for later conversations. The read path begins when Codex builds a conversation's initial context.

## 4 Loading saved memory context into a new conversation

When Codex builds a new conversation's initial context, **it automatically loads `memory_summary.md` and memory lookup instructions**, provided the [memory feature and `use_memories` are enabled](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/ext/memories/src/extension.rs#L45-L74) and the saved summary is readable and nonempty.

The V1 summary contains [preferences the model can apply directly](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/templates/memories/consolidation.md#L510-L544) and [a topic index for finding more detail](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/templates/memories/consolidation.md#L568-L639). Codex loads this existing summary regardless of the opening request's topic; [the loader does not take the user's request as input](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/ext/memories/src/prompts.rs#L35-L38).

This connects to the [initial-context step in our context engineering blog post](/blog/inside-codex-context-engineering/#3-describing-the-world-for-this-step). Inside `turn::run_turn`, [context assembly](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/mod.rs#L4319-L4391) asks registered extensions for context. [`MemoriesExtension::contribute_thread_context`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/ext/memories/src/extension.rs#L57-L100) calls [`build_memory_tool_developer_instructions`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/ext/memories/src/prompts.rs#L35-L64). The highlighted lines show the file read, truncation, and empty-summary check:

```rust {2,3,8,9,10,12}
let base_path = codex_home.join(version.directory_name());
let memory_summary_path = base_path.join("memory_summary.md");
let memory_summary = fs::read_to_string(&memory_summary_path)
    .await
    .ok()?
    .trim()
    .to_string();
let memory_summary = truncate_text(
    &memory_summary,
    TruncationPolicy::Tokens(MEMORY_TOOL_DEVELOPER_INSTRUCTIONS_SUMMARY_TOKEN_LIMIT),
);
if memory_summary.is_empty() {
    return None;
}
```

The [budget is 2,500 estimated tokens](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/ext/memories/src/lib.rs#L11-L16), using [byte-length estimates and preserving the beginning and end](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/utils/string/src/truncate.rs#L34-L60). A failed file read or empty summary produces no memory instructions. Otherwise, the builder [wraps the summary in the lookup template](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/ext/memories/src/prompts.rs#L53-L64).

The extension contributes this text as [developer instructions](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/ext/memories/src/extension.rs#L75-L100). Codex [records it in conversation history](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/mod.rs#L4734-L4794), which becomes part of the `input` passed to [`turn::build_prompt`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L1563-L1592). Ordinary later turns [reuse the recorded context](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/mod.rs#L4740-L4772); this is not a fresh summary read before every model call.

The model now has a compact starting context. Detailed `MEMORY.md` entries remain on disk; [section 5](#5-retrieving-relevant-memory) follows how the model decides whether to read them.

## 5 Retrieving relevant memory

**The model decides when to retrieve more detail.** V1's [lookup instructions](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/ext/memories/templates/memories/read_path.md#L6-L17) ask it to consult memory for requests related to saved topics or earlier decisions. Clearly self-contained requests can skip lookup; uncertain cases should get a quick memory pass. This can happen during any turn where memory guidance is available.

With V1 selected and `/home/engineer/.codex` as Codex home, the [prompt gives this lookup sequence](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/ext/memories/templates/memories/read_path.md#L33-L41):

```md
Quick memory pass (when applicable):

1. Skim the MEMORY_SUMMARY below and extract task-relevant keywords.
2. Search /home/engineer/.codex/memories/MEMORY.md using those keywords.
3. Only if MEMORY.md directly points to rollout summaries/skills, open the 1-2
   most relevant files under /home/engineer/.codex/memories/rollout_summaries/ or
   /home/engineer/.codex/memories/skills/.
4. If above are not clear and you need exact commands, error text, or precise evidence, search over `rollout_path` for more evidence.
5. If there are no relevant hits, stop memory lookup and continue normally.
```

These are model-directed tool calls. By default, [`dedicated_tools = false`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/config/src/types.rs#L376-L393), so the model can use ordinary shell and file tools. Enabling it [adds tools scoped to the memory folder](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/ext/memories/src/extension.rs#L135-L159). The prompt [suggests four to six search steps](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/ext/memories/templates/memories/read_path.md#L43-L46) to keep a quick pass lightweight.

Retrieved text returns through the [tool-result loop in our context engineering blog post](/blog/inside-codex-context-engineering/#7-a-tool-result-and-the-loop-repeats). [`turn::drain_in_flight`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L2462-L2490) records the output in conversation history, and [the next model call receives that history](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/session/turn.rs#L523-L541), including the retrieved evidence.

For facts that may have changed, the [verification instructions](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/ext/memories/templates/memories/read_path.md#L51-L73) ask the model to verify them when doing so is cheap. I would also treat saved preferences as a starting point and follow new directions for the current task.

When retrieved memory informs the answer, V1's [citation instructions](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/ext/memories/templates/memories/read_path.md#L75-L115) ask the model to identify what it used. The next section follows how those citations influence future memory selection.

## 6 Tracking which memories get used

When the model uses retrieved memory, the V1 [memory prompt](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/ext/memories/templates/memories/read_path.md#L75-L115) asks it to finish its answer with an `<oai-mem-citation>` block. This markup identifies the memory-file lines it used and the earlier sessions that supplied the evidence.

Codex [parses the hidden citation markup](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/stream_events_utils.rs#L46-L60), then [`record_stage1_output_usage_for_memory_citation`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/stream_events_utils.rs#L201-L213) passes valid session identifiers to the memory database when it is available. [`record_stage1_output_usage`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/state/src/runtime/memories.rs#L58-L85) executes the SQL below. The highlighted lines increment the use count for a cited session's extraction result and record when it was used:

```sql {3,4}
UPDATE stage1_outputs
SET
    usage_count = COALESCE(usage_count, 0) + 1,
    last_usage = ?
WHERE thread_id = ?
```

Future consolidation passes use that feedback when [selecting extraction results](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/state/src/runtime/memories.rs#L438-L515). The [default recency window is 30 days](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/config/src/types.rs#L55-L60): the check uses the last recorded use, falling back to the source's last update if the result has never been used. Eligible, nonempty results from enabled chats are ranked by usage count, then recency, and up to 256 are selected by default.

A file read alone does not update the count, and a citation does not guarantee permanent retention.

Explicit user corrections supply another input to consolidation: update notes.

## 7 Updating memory on request

An explicit request to remember, correct, or forget something reaches the model as an ordinary user message. With memory guidance loaded, the [V1 instructions](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/ext/memories/templates/memories/read_path.md#L117-L123) direct it to write a small Markdown note under `extensions/ad_hoc/notes/`, leaving generated files alone. **Note creation is a model action**, rather than an automatic runtime response to the words "remember this".

The model can use ordinary file tools or, with dedicated tools enabled, [`AddAdHocNoteTool`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/ext/memories/src/tools/ad_hoc_note.rs#L39-L92) (`memories.add_ad_hoc_note`). An illustrative note contains:

```md
For future blog drafts, keep the introduction to one paragraph.
```

The [note-writing backend](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/ext/memories/src/local/ad_hoc_note.rs#L17-L40) saves the file without launching consolidation or updating the summary. The current model already knows the correction from our message; future conversations [receive the generated summary, not pending notes](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/ext/memories/src/prompts.rs#L35-L64).

A background pass can incorporate the note during [Phase 2 consolidation](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/src/phase2.rs#L94-L164). Notes enter this stage directly, so their contents do not depend on Phase 1 rediscovering the request. The startup task [still reaches Phase 2](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/src/start.rs#L88-L92) even when [Phase 1 finds no eligible chats](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/src/phase1.rs#L60-L72). The worker's [note-processing instructions](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/templates/extensions/ad_hoc/instructions.md#L4-L13) tell it to incorporate new or edited notes, keep their files, and tag derived information with `[ad-hoc note]`.

Timing depends on when the pass sees the note: it may compute its workspace diff before the model writes the file. [Leases and cooldowns](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/state/src/runtime/memories.rs#L1071-L1150), together with the quota guard, can delay processing further. **Saving a note does not promise an updated summary in the same turn or the next one.**

V2 keeps this update mechanism while changing what extraction produces and how consolidated memory is organized.

## 8 Exploring V2

**V2 is version 2 of Codex's local memory pipeline.** It changes how earlier chats are extracted, how their findings are organized, and how later conversations retrieve them.

Keep memories enabled and select V2 in `config.toml`:

```toml
[memories]
version = "v2"
```

V2 uses [`memories_v2/`, alongside V1's `memories/`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/protocol/src/memory_version.rs#L15-L23). The optional [`dual_write = true`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/src/start.rs#L40-L59) starts background pipelines for both versions; the [read extension still uses the selected version](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/ext/memories/src/extension.rs#L45-L52). Generating both does not combine their context in a new conversation.

### 8.1 Extracting session history

V2 keeps [extraction followed by consolidation](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/src/start.rs#L64-L92). Within **Phase 1: extraction**, the first important change is how [`rollout_input::serialize_tiered_input`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/src/rollout_input.rs#L38-L244) chooses evidence for the extraction model. Under a limited input budget, it prioritizes human messages over assistant and tool material, chooses newer evidence first within each category, then renders the selected items in their original order.

V2's extraction prompt asks for a [faithful account of the session](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/templates/memories/stage_one_system_v2.md#L9-L30), distinguishing the user's words from agent suggestions and preserving the scope of preferences.

The [V2 output schema](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/src/phase1_output.rs#L73-L81) requires just `rollout_summary` and a string `rollout_slug`. A shortened illustrative result is:

```json
{
  "rollout_summary":
    "Future blog drafts: short intros, familiar examples, no Explore titles.",
  "rollout_slug": "blog-writing-style"
}
```

V1 extracts a separate `raw_memory` as well; V2 carries the session account forward without that field. The [V2 parser](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/src/phase1_output.rs#L34-L58) redacts generated fields and applies a 9,000-byte truncation policy to the recap. That recap becomes the input for V2 consolidation.

### 8.2 Building a compact summary

V2 also uses **Phase 2: consolidation**, through the same [`phase2::run`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/src/phase2.rs#L49-L192) function. Its [input synchronization](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/src/phase2.rs#L194-L208) writes selected session recaps into `rollout_summaries/` for both versions, but rebuilds `raw_memories.md` only for V1. The worker then receives the [version-specific consolidation prompt](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/src/prompts.rs#L85-L88).

The files have different roles in each version:

| Memory role | V1 | V2 |
| --- | --- | --- |
| Inputs from extraction | Recaps and `raw_memories.md` | Recaps; no raw-memory file to rebuild. |
| Generated knowledge | [`MEMORY.md`](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/templates/memories/consolidation.md#L201-L245) and [a short summary](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/templates/memories/consolidation.md#L448-L481) | [A compact <code>memory_<wbr>summary.md</code>](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/templates/memories/consolidation_v2.md#L1-L27) |
| Routes to detail | [Topics and keywords](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/templates/memories/consolidation.md#L568-L639) | [Exact recap filenames and session identifiers](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/templates/memories/consolidation_v2.md#L29-L38) |

V2's worker is instructed to keep reusable, user-supported preferences in the summary and preserve single-task decisions with their tasks. It must [ground its claims and pointers in supplied evidence](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/templates/memories/consolidation_v2.md#L13-L38). A complete illustrative summary could look like this. The session identifier is fictional, and the recap filename follows the [source naming rules](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/src/storage.rs#L161-L237):

```md
v1

## User Profile

Writes blog posts in /work/blog.

## User preferences

- For future blog drafts, keep introductions short.
- Use concrete, familiar examples; avoid "Explore" in blog titles.

## General Tips

## What's in Memory

### /work/blog

#### 2026-10-09

- rollout_summaries/2026-10-09T12-00-00-0001-blog_writing_style.md —
  User feedback on blog introductions, examples, and title wording;
  thread_id=01a12088-d200-7000-8000-000000000001
```

The `v1` first line is intentional: [V2 validation still requires that literal header](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/src/workspace.rs#L118-L129), along with the four headings shown above and a file smaller than 10,000 UTF-8 bytes. The config selects the pipeline version; the file header does not. V2 [does not require a `MEMORY.md` artifact](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/memories/write/src/workspace.rs#L76-L114). These checks verify structure, not whether the worker's account is true.

V2 puts direct recap routes in the injected summary, so detailed retrieval can begin from those pointers.

### 8.3 Reading through direct pointers

V2 uses the [same summary loader and budget of 2,500 estimated tokens](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/ext/memories/src/prompts.rs#L35-L64) as V1. Its lookup instructions route the model directly to referenced session recaps, rather than starting with a keyword search in `MEMORY.md`. With `/home/engineer/.codex` as Codex home, the [V2 template](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/ext/memories/templates/memories/read_path_v2.md#L3-L10) opens with:

```md
Use the injected MEMORY_SUMMARY as historical context: apply the user's actual
preferences, corrections, decisions, and supported task scope. Its exact
rollout, source, pull-request, discussion, and document pointers can guide
independently useful work without an extra lookup merely to rediscover them.
Read a matching rollout under `/home/engineer/.codex/memories_v2/rollout_summaries/` when its
additional evidence, wording, chronology, or uncertainty could change your
answer; otherwise do not retrieve history speculatively. Search selectively
when a genuinely needed route is missing.
```

The extension [splits the rendered developer instructions](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/ext/memories/src/extension.rs#L75-L100) into fragments of at most 8,900 bytes at UTF-8 boundaries. The summary budget applies before template rendering and fragmentation.

V2's [citation rule](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/ext/memories/templates/memories/read_path_v2.md#L22-L40) requires a read session recap to inform the answer and excludes `memory_summary.md`. Recap citations can feed the [usage tracking from section 6](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/core/src/stream_events_utils.rs#L201-L213); applying injected preferences alone does not produce that feedback.

Explicit updates follow [the same note-writing guidance](https://github.com/openai/codex/blob/c3d3b142d10f4316b46e35aad7e5317e7e506cb7/codex-rs/ext/memories/templates/memories/read_path_v2.md#L15-L20) under `memories_v2/extensions/ad_hoc/notes/`, with the consolidation timing explained in [section 7](#7-updating-memory-on-request).

## What stands out in Codex's design

What stands out to me is the boundary between runtime coordination and model judgment. The runtime schedules background work, validates file structure, and supplies memory context. The models decide what findings mean, how to organize them, and when earlier evidence is useful. The files and prompts make those decisions inspectable, while their quality still depends on faithful extraction, consolidation, and retrieval.

I would keep required project rules in `AGENTS.md` or checked-in documentation, as the [official guide recommends](https://learn.chatgpt.com/docs/customization/memories), and use memory to carry context from earlier work.

Our earlier blog post, [CodeX Architect Part 1 - Context Engineering](/blog/inside-codex-context-engineering/), covers how Codex assembles the context for a model request. The next blog post, **CodeX Architect Part 3 - Subagents**, will cover how the main model delegates a task, how the child works, and how its result comes back.
