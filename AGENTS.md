# Personal blog working agreement

These instructions apply to writing, editing, researching, and reviewing posts in this repository. Keep unrelated implementation work lightweight; the detailed rules below are for blog work.

## Voice and editorial style

- Write like an experienced engineer explaining what they learned to another engineer. Prefer a direct story over a formal report.
- Use `we`, `us`, and `our` when walking through an implementation or shared reasoning. Use `you` for reader-facing recommendations such as “Which one should you choose?”
- Preserve the author's stated opinions and experience. Do not replace them with generic marketing language.
- Keep the introduction to one or two short paragraphs. It should establish why the post exists, not repeat the TL;DR or later sections.
- Give every paragraph one job. Remove throat-clearing, repeated conclusions, and sentences that do not advance the comparison.
- Explain unfamiliar concepts in plain language before introducing framework-specific class names.
- Prefer the shortest wording that preserves the important distinction. If two products are essentially similar on one dimension, say so once instead of filling both cells with prose.

## Research and factual accuracy

- Treat SDKs, APIs, product behavior, pricing, and documentation as current information. Verify them against current primary sources before drafting or materially revising a claim.
- Prefer official documentation, API references, and source code. Use a repository example only for what that example actually demonstrates.
- Never turn an inference into a documented feature. Mark an inference clearly or omit it.
- Distinguish the framework's common interface from provider-specific behavior. For example, a sandbox provider may offer TTLs or snapshots even when the framework's backend protocol does not standardize them.
- Compare equivalent layers. A lean core SDK, an opinionated agent harness, a graph runtime, and a provider integration are not interchangeable scopes.
- Similar names are not evidence of equivalent behavior. State what each configuration object controls before comparing it with another object.
- Verify every code sample against the current API, including imports, method names, required persistence components, default behavior, and sync versus async variants.
- Put citations next to the claims they support. A link at the end of a large paragraph must support the entire paragraph or be split into more precise citations.
- If a factual statement cannot be verified, remove or narrow it. Do not fill gaps with plausible architecture claims.

## Technical comparison workflow

Before writing the full comparison:

1. Define the products and layers in scope.
2. Build a private evidence map: claim, exact source, documented scope, and any caveat.
3. Create the comparison dimensions from that evidence rather than from symmetry or convenient headings.
4. Draft only after the important differences are understood.

For comparison tables:

- One row must answer one clearly named dimension, and both product cells must answer that same dimension.
- Prefer short, current code examples when API shape communicates the difference better than prose.
- Do not invent a row merely so one product can have a checkmark and the other a cross.
- Say “not part of the common interface” instead of “not supported” when an application or provider can supply the behavior.
- Keep cells compact. Move lifecycle nuance and caveats into the text below the table when they make the table hard to scan.
- Audit every cell individually against a primary source before calling the table verified.

For diagrams:

- Use a diagram only when it makes control flow, ownership, lifecycle, or delegation easier to understand.
- Introduce the basic comparison in prose or a compact table first; place the detailed architecture diagram afterward.
- Every box and arrow is a factual claim. Verify labels, defaults, ownership, and whether sharing is automatic or explicitly wired by the application.

## Full-article review rule

Treat user feedback as a rule for the whole article, not only the quoted sentence.

After any correction:

1. Search the entire post for the same wording, assumption, duplication, attribution problem, or unsupported claim.
2. Fix every analogous occurrence, including the TL;DR, tables, captions, diagrams, and conclusion.
3. Reread the surrounding section to ensure the local fix did not create a contradiction or abrupt transition.
4. Do not say the article was fully reviewed unless a complete paragraph-by-paragraph and table-cell-by-table-cell pass was actually performed.

During the final editorial pass, verify that:

- The introduction, TL;DR, and conclusion do not repeat one another.
- Each paragraph makes it immediately clear which SDK or framework it describes.
- Opinion is phrased as opinion; documented behavior is phrased as fact.
- Terms appear in the section where readers need them. Do not introduce sandbox-specific types in a foundational chat/session section.
- Recommendations follow from differences already explained in the article.
- No stale wording remains after headings, product names, or conclusions change.

## Repository and rendering checks

- Posts live in `src/content/blog/`. Keep frontmatter consistent with the existing content collection.
- A finished new post needs a hero image in `src/assets/` and a working `heroImage` frontmatter entry unless the user explicitly declines one.
- Put code-native diagrams such as SVG architecture figures in `public/` and make them openable at full size.
- Run `npm run build` after content or layout changes.
- Inspect the rendered article, not only the Markdown. Check the article page and blog listing for the hero image, TOC anchors, code wrapping, table overflow, diagram legibility, and balanced use of page width.
- Do not solve a wide-table problem by widening only the table if the surrounding article then looks visually disconnected. Review the whole layout at desktop and narrow widths.
- Report unresolved factual uncertainty or rendering limitations plainly. Do not claim confidence merely because the build passes.
