---
name: blog-post-writing
description: Write, revise, or review technical blog posts in this repository with clear standalone explanations, coherent sections, verified claims, and readable rendering.
---

# Blog post writing

Follow `AGENTS.md` for repository conventions and editorial standards. This skill helps decide what to explain, how much evidence to show, and when to review it. User feedback and accepted preferences take precedence over a suggested structure.

## Start with the reader's question

Locate the canonical post and read the relevant conversation before drafting. For a new post or substantial restructure, keep a short private brief:

- What does the reader already know, and what should they understand afterward?
- What is the author's main observation or opinion? Is the author exploring an existing implementation, reporting personal experience, or proposing a design?
- Which implementation, source revision, versions, environments, and configuration are in scope?
- What throughline will make the explanation easiest to follow?
- Which title, series order, draft status, and other preferences have already been settled?

Resolve routine choices from that context. Ask only about missing information that materially affects correctness or scope; do not add an outline-approval step to work already requested.

Keep this brief current as the user steers the work. Distinguish publication date from source-revision date, and local preview visibility from production publication. Reassess a temporary draft decision when the task moves toward publication; use the latest explicit instruction and make the intended status visible in progress updates.

**Choose the throughline; do not force a scenario.** An architecture post can follow a request, a background job, a stored artifact, or a reader question. Use a running example when it makes an unfamiliar mechanism concrete. A familiar feature such as memory may be clearer as a direct explanation. Keep illustrative artifacts when they teach something; avoid repeatedly inventing a task to motivate a concept the reader already understands.

## Give the opening a reason to read on

When calibrating the author's voice, read two or three existing openings. Identify what prompted each post and how it leads into the subject. Preserve that personal stance without inventing an experience or copying an anecdote into a new setting.

First write the situation plainly: who did what, what result they produced, and who used it. Check research examples against the actual reported workflow. Do not turn human coordination into autonomous agent behavior, or imply that the inspected implementation powered the reported result without evidence.

Then decide what makes that situation worth exploring. For a familiar idea, an application, observation, or concrete engineering question often works better than a textbook definition. “I find X interesting,” vague references such as “groups doing this,” and a closing rhetorical question cannot supply missing meaning. A story, opinion, or question is useful when it adds something specific; none is required.

Read the opening with the first section before expanding the body. A reader should be able to explain the situation, the author's question, and why the implementation comes next without knowing this conversation. Put detailed comparisons and setup in an early overview rather than crowding the opening.

Make series references explain the shared mechanism or the new branch of the earlier flow. When version differences determine scope, introduce the pattern first, compare the relevant versions early, and then state which version we follow and briefly why.

## Check the handoff between paragraphs

Read each revised paragraph together with its neighbors. Identify the concrete action, result, or unresolved question that carries into the next paragraph. Privately explain their relationship in one plain sentence. If that requires context from this chat, rewrite the pair before polishing individual sentences.

A transition should show why the next point follows. Moving from a research example to source code requires naming the common mechanism we want to inspect. A series title, “also,” or “this makes me curious” does not establish that relationship on its own. Remove duplicated conclusions or topic announcements that a new bridge makes unnecessary.

## Plan sections before expanding them

Assign each proposed section one reader question, its short answer, the evidence needed, and the idea it hands to the next section. Keep this plan private unless an outline is the requested deliverable.

Check the plan for overlapping answers and missing prerequisites. Define unfamiliar terms before relying on them. Explain what a version number refers to, what a setting controls, and whether a step concerns a new conversation, an ordinary later turn, or a model call.

Order a walkthrough by the implementation's flow, including asynchronous branches. Order a comparison by the decisions the reader needs to make. Give a version comparison one parent section when its subsections share that purpose. Put detailed entry-point explanations in the series post that owns them; use a brief reminder and an explicit link elsewhere.

Use headings that state the subject or action. Avoid vague headings such as “The lesson we will follow.” A transition should answer the question left by the preceding paragraph; adding “next” does not repair a missing logical connection.

## Verify the claims that shape the draft

Build a private evidence map for the important claims before writing around them: claim, exact source, scope or conditions, and verification status. Separate documented behavior, prompt instructions, reported experimental results, inference, and illustrative output.

Read the relevant reference when its claims are in scope:

- [Code walkthroughs](references/code-walkthroughs.md): pinned source, triggers, configuration, async work, artifacts, source excerpts, and workflow diagrams.
- [Research claims](references/research-claims.md): papers, benchmark numbers, baselines, ablations, and causal wording.

Use primary sources. Preserve the studied source revision for a code walkthrough; verify current claims against current sources. Numbers and caveats that change the post's argument deserve investigation before drafting, rather than a check after the prose is polished.

Keep evidence bookkeeping out of the article. Publish the source revision and consequential assumptions concisely, using a parenthetical source note when appropriate. Put citations beside the claims they support.

## Draft, then cut each section

Lead with a plain-language answer to the section's question, then explain the mechanism and evidence. Introduce a function or type when it helps the reader trace that mechanism. Choose an artifact because it adds understanding, rather than requiring one under every heading.

Prefer one primary artifact per section. Additional blocks can be useful when they show different things, such as the call site requesting work and the async function launching it. A call tree, source excerpt, output example, and surrounding prose should each contribute something different.

Before expanding the next section, reread the current one and its neighbors:

- Does every paragraph add a mechanism, evidence, consequence, or needed distinction?
- Does the prose merely narrate every line of a code block or repeat a quoted prompt?
- Has the overview, another paragraph, a table, or the previous section already answered this question?
- Can a short example replace several paragraphs? Is the example itself unnecessary?
- Does a caveat change the reader's interpretation, or merely make the section longer?
- Are separate mechanisms explained separately before their relationship is described?

Cut repetition before adding headings or transitions. A long section is a signal to examine its scope, not a reason to impose a universal word limit. Preserve distinctions such as automatic context loading versus model-directed retrieval, or saving an update versus incorporating it.

Give the introduction, overview, and conclusion different jobs. The introduction establishes why the post matters; the overview orients the reader; the conclusion states what the author learned. Preserve the author's viewpoint without repeating the same thesis in all three.

## Preserve the series voice

For **CodeX Architect**, the author explores existing Codex architecture. Use “we follow” or “we inspect” for shared reasoning, and name Codex, its runtime, or its model as the actor doing implementation work. Do not imply that the author built Codex. Preserve stated opinions and their scope instead of replacing them with generic claims.

Use the established title style, **CodeX Architect Part N - Topic**, without “Explore” in the title. At the first reference to another post, link its actual title; later references may use a clear descriptive label. Avoid bare “Part 1” or “Part 2.” Keep series endings, draft status, and publication order consistent with the user's decisions.

Use `we`, `us`, and `our` for the walkthrough and `you` for reader-facing recommendations. Keep terminology consistent. Mark invented inputs and artifacts as illustrative; distinguish them from verbatim source or research results. Do not mention another agent's investigation as reader-facing provenance unless the author requests that attribution.

## Handle feedback as a rule, not a patch location

Identify the underlying issue before editing: undefined terminology, ambiguous actor or timing, duplicated explanation, an unnecessary scenario, overstated evidence, or the wrong author stance.

When several questions expose the same ambiguity, repair the section's explanation and order instead of appending an answer to each question. Introduce the missing distinction where readers first need it, then remove later paragraphs made redundant by that explanation.

Locate the passage the user means in the active article and read the surrounding paragraph pair. For transition feedback, revise the connection between the ideas, not only the quoted sentence's wording. State the specific bridge being changed so a mistaken interpretation can be corrected early.

Apply the correction across analogous prose, headings, TOC entries, tables, diagrams, captions, and the ending. Reread neighboring sections after the edit. Carry accepted preferences through later revisions; do not reintroduce a removed example or explanation because an old template expects it.

Keep durable preferences and reusable failure patterns in this workflow when asked to improve it. Keep a particular post's dates, scores, file paths, and publication decisions in that post or its working notes. Do not turn one successful edit into a universal ban on examples, long sections, or multiple code blocks.

## Finish with proportionate review

For a new post, substantial restructure, or full audit, perform both passes over the finished article:

1. **Reader pass:** read it as a standalone post, without relying on the conversation. Check every paragraph, heading, table cell, artifact introduction, and transition. A reader should know who acts, what changes, and why the next section follows.
2. **Evidence pass:** check each factual claim against its source and conditions, including captions and examples. Confirm that a linked line or function actually supports the sentence, rather than merely existing.

For a narrow edit, review the containing section, its neighbors, and analogous occurrences throughout the post. Recheck changed claims and affected artifacts; reuse valid evidence for unchanged passages. Do not report a whole-article audit when only a section was reviewed.

Keep frontmatter consistent with the content schema and add `updatedDate` when materially revising a published post. After content or layout changes, run `npm run build` and inspect the rendered article and listing at desktop and narrow widths. Check the hero image, anchors, code, tables, and diagrams; internal code scrolling on mobile is distinct from page overflow. In this environment, use a 500px narrow headless capture. Investigate stale content caches when the browser serves an older revision; do not clear caches routinely.

Make the author-facing review work at the requested local URL. Check the actual branch, file, server mode, and draft filtering when an edit is not visible; an already committed edit will not appear in `git diff`. Do not remove a draft flag merely to expose a local preview. Clearly distinguish local development, production build output, an open PR, and the live site.

During active paragraph review, make edits visible locally and reread the affected prose before running publication checks. Group the current corrections into a coherent revision and reuse valid evidence for unchanged claims and artifacts. A wording-only skill edit needs skill validation and diff checks; it does not require rebuilding the website unless content or layout also changed.

Report the concrete changes, verification performed, and any remaining limitation. When the feedback concerns unclear prose, show the revised passage or paragraph pair so the user can assess the meaning directly. Build, rendering, and CI results supplement the reader pass; they do not prove that the writing is clear.
