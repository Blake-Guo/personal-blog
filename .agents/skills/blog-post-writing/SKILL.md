---
name: blog-post-writing
description: Write or restructure a technical blog post for this repository as a coherent, example-driven walkthrough, then audit it for factual accuracy, header fit, and smooth transitions before calling it done.
---

# Blog post writing

Use this skill when asked to write a new post, restructure an existing one, or audit a draft in `src/content/blog/`. It builds on the voice and research rules in `AGENTS.md`; read that file first and treat this skill as the workflow on top of it.

## 1. Decide what the post follows

A post in this repository is a story about one thing moving through a system, not a tour of files. Before drafting, settle three things and write them at the top of the post:

- **The pinned source.** For code walkthroughs, name the exact commit (short hash, date, link) and verify every claim against that commit, never against `main`.
- **The running example.** Pick one concrete input, such as a single user request, a single HTTP call, or a single config change, with enough setup to exercise every path the post covers. State the setup once, in plain words, and return to it in every section. Mention the input inline in the setup sentence; give it a code block only if later sections quote it. Keep only the setup details that later sections render from, and cut any setup sentence that restates the journey paragraph or the table of contents.
- **The order.** Sections follow the order the system executes, not the order you discovered things. If a function runs third, it is explained third, even if it was the most interesting find.

Build a private evidence map before writing: claim, exact file and line at the pinned commit, and any caveat. If a claim cannot be mapped, it does not go in the post. Compute every count and size in the post with a script (catalog entries, template characters, flag values across all entries) and keep the output in the map; never estimate or extrapolate from one entry.

## 2. Structure the walkthrough

Open with one or two short paragraphs on why the post exists, then a table of contents, then a section that introduces the running example and the overview diagram. Number the walkthrough sections in execution order and put reference material, such as a full call tree, at the end rather than in the middle.

Each walkthrough section has the same shape:

1. **Where we are.** The first sentence says which step of the example this is and which function or component is running now.
2. **What it does.** Explain the mechanism in plain language before naming types. Introduce a class or function name only when the reader needs it to follow a link.
3. **What it looks like.** Show the real artifact: a short code excerpt, a rendered prompt, a request shape, or a table of where text comes from. Prefer rendered output from real templates or snapshot tests over paraphrase.
4. **What the next section needs.** End by naming the thing the next step consumes. The reader should never arrive at a new heading without knowing why it exists.

Keep each paragraph to one job. When two mechanisms differ only in one dimension, use a compact table with one row per dimension rather than parallel prose.

When a design decision branches, say which branch the running example takes and why, then cover the other branch briefly.

## 3. Draft the content

- Use `we` for the walkthrough and `you` only for reader-facing advice.
- Code excerpts are verbatim from the pinned commit, with indentation trimmed but no reflowing. Produce them with `git show <commit>:<path> | sed -n 'A,Bp'`, never by retyping, and record the range. Show enough of the function for the reader to see the context, then mark the lines the prose discusses with Shiki's meta syntax, for example ```` ```rust {8,9,10,11} ````, and name those lines in the sentence before the block. Link the full function beside the excerpt. A closing fence must sit alone on its line; text after it silently swallows the next block.
- A link labeled with a function name points at that function's definition. When the sentence is about where it is called, link the call site with a separate label such as "checks and records the user's input".
- An illustrative output that depends on configuration, such as a rendered message stack, states the configuration it assumes in the sentence before it.
- Put the citation next to the sentence it supports. A link at the end of a paragraph must support the whole paragraph.
- When text can come from more than one place, such as a per-model catalog versus a compiled-in default, name both sources and say which one the example uses.
- Prefer "nine of ten bundled models" over "this model uses". Check how many other cases share a behavior before presenting one as representative or special.
- Mark inferences as inferences ("From this path, I infer...") and keep them to one sentence.
- Preserve the author's opinions and fun facts. Do not flatten them into neutral summary.
- Wrap invented example inputs in double quotes and italics, such as "*Fix the failing test in `src/parser.rs`*". Text quoted verbatim from source, templates, or documentation uses double quotes without italics.

## 4. Diagrams

Add a diagram only when it makes control flow, ownership, or lifecycle easier to see. Place it after the example is introduced, before the detailed sections. Every box and arrow is a factual claim and must match the text. Store SVGs in `public/`, provide a mobile variant when the desktop one is wider than 600px, and keep footer text within the viewBox width at the declared font size. Check both variants after any text change.

## 5. Audit before finishing

Run two full passes over the finished Markdown, not the draft in your head.

**Factual pass.** For every function name, line reference, number, quoted string, and "X does Y" sentence:

- Open the file at the pinned commit and confirm the claim, including line numbers in call trees and link anchors. For line anchors that span a range, confirm the end line as well as the start.
- Diff every code block against its recorded source range with a script (dedent both sides) and confirm each highlighted line number lands on the line the prose names.
- Sentences about the running example ("nothing is added here", "the model will call a tool") are claims too. Verify them or soften them to what the code guarantees.
- Confirm scope words: "only", "always", "the default", "at this commit". If a behavior is shared by other models, configs, or branches, say so.
- Confirm provenance words: catalog versus bundled, config versus runtime, user role versus developer role.
- Confirm rendered examples against real templates or snapshot tests, with placeholders substituted the way the code substitutes them.
- Distinguish "drops" from "replaces", "repairs" from "filters", and "sends" from "records". These words are easy to blur and readers rely on them.

**Structure pass.** For every heading:

- The content under it matches the heading. A paragraph that does not serve the heading moves to the section it serves.
- The first sentence says where the running example is now.
- The last sentence hands off to the next section.
- Sub-headings within a section follow cause then effect, or first time then later times, never discovery order.

After any correction, search the whole post, the table of contents, the diagram, its alt text and captions, and the conclusion for the same wording or assumption, and fix every occurrence.

## 6. Repository checks

- Keep frontmatter consistent with `src/content.config.ts`. Add `updatedDate` when materially revising a published post.
- Run `npm run build` and confirm every table-of-contents anchor resolves in `dist/`.
- Astro's content-layer cache in `.astro/` can keep serving a broken version of a post to a dev server started before the fix. After fixing a rendering bug, delete `.astro/` and `node_modules/.astro`, rebuild, and tell the reader to restart `npm run dev`.
- Validate edited SVGs with `xmllint --noout`.
- Look at the rendered page, not only the HTML. Without a browser tool, serve `dist/` with `npm run preview -- --port 4399` and capture it with headless Chrome, then slice the PNG and inspect it:

  ```bash
  "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless=new --disable-gpu \
    --hide-scrollbars --window-size=1280,14000 --screenshot=desktop.png \
    http://127.0.0.1:4399/blog/<slug>/
  ```

  Use 500px, not 390px, for the narrow pass; headless Chrome enforces a minimum window width, so a 390px capture shows the right edge cut off on every page and proves nothing.
- Check code blocks in the capture: no line should be cut off at the right edge at desktop width, and highlighted lines should run edge to edge of the block.
- Report what was verified and what was not. If the rendered page was not viewed, say so.

## 7. Mistakes caught in past reviews

Check for each of these explicitly; every one slipped past a draft once.

- Presenting one case as special when it is the default: "gpt-6-sol uses Responses Lite" when nine of ten bundled models did.
- Blurring "drops" and "replaces": local compaction drops media and keeps text; history normalization replaces media with placeholders. The two are different mechanisms.
- Writing "each section" when only some sections behave that way.
- Leaving a sentence on a closing fence line, which merged two code blocks and a heading into one.
- Linking a function name to a call-site range instead of its definition.
- Giving the running example a code block when nothing later quoted it, then surrounding it with sentences that repeated the journey paragraph.
- Asserting what the example does not trigger without reading the code path.
- Treating a 390px headless capture as a layout bug when a sibling post showed the same clipping.
- Blaming the post for stale rendering when the `.astro/` cache was serving the old version.
