# Code walkthroughs

Use this reference when explaining implementation behavior, configuration, source excerpts, or architecture diagrams. Keep the evidence map private and tied to the source revision.

## Follow ownership and control flow

For the mechanism being explained, establish:

| Question | What to verify |
| --- | --- |
| Who requests the work? | The actual caller and its conditions; distinguish a client request handler from the core model loop. |
| Who launches or executes it? | Queue admission, scheduling, and the exact async launch or worker dispatch. |
| What runs independently? | Task ownership, await boundaries, and whether returning means dispatch or completion. |
| What state changes? | Inputs, outputs, database rows, files, and whether writing and consuming are separate steps. |
| Who decides to use the output? | Runtime injection, an instructed model decision, or an explicit tool call. |
| What can skip or delay work? | Feature gates, eligibility, leases, cooldowns, permissions, and budget checks relevant to the claim. |

An explicit user request, a prompt instruction, and a runtime trigger are different mechanisms. Trace their connection before describing it. “Remember this” is not evidence of a keyword handler. A saved note is not evidence of an immediately updated summary. A worker dispatch is not evidence that consolidation finished.

Verify a claimed absence by following the actual path. Do not infer “this does not start another job” from a function name or tool description alone. Distinguish live-conversation compaction from background consolidation of persistent knowledge.

Check scope words such as “only,” “always,” and “default” across the relevant cases before presenting one as special or representative. When an artifact can come from a catalog, a compiled default, or configuration, identify the source used on the traced path. Preserve distinctions between dropping and replacing data, or recording and sending it.

## Explain configuration in layers

Identify the master feature gate, defaults within that feature, per-conversation eligibility, and read/write controls separately. “Feature off by default” can coexist with enabled settings that apply after that gate is on. Say what each control governs before describing its effect.

Keep version and environment scope explicit. A local embedded CLI path does not establish behavior for every client. A new memory-pipeline version is not a new version of the whole product. A rule applying under managed sandboxing does not establish behavior under disabled or externally managed sandboxing.

Verify limits at the enforcing code. Preserve units and ordering: bytes versus tokens, estimated versus tokenizer-counted tokens, file size versus injected context, and truncation before template rendering or fragmentation. Script aggregate counts or calculations when they are easy to miscount; read literal defaults from their definitions.

## Show enough source to establish the claim

- Extract verbatim code from the pinned revision with `git show <commit>:<path>` and record the source range. Dedenting is fine; silently rewriting or reflowing source is not.
- Choose a contiguous excerpt that exposes the important condition or state change. Omit unrelated setup when the remaining excerpt is understandable. Use separately labeled excerpts when needed; do not disguise invented omission markers as source.
- Link the full function for surrounding details. A symbol-labeled definition link should start at its definition; label a call-site link as a call site or launch point.
- Highlight the lines discussed with Shiki metadata, such as `rust {1,3,4}`, and verify those positions. Keep closing fences on their own lines.
- Compare quoted blocks programmatically with their recorded source ranges. A range-bounds check is useful, but it does not prove that the source supports the prose.
- Render prompt examples using the template's actual substitutions. Check illustrative JSON against the relevant schema. State consequential assumptions, while avoiding a separate fictional story for every output.

Prompt instructions describe what the model is asked to do; runtime code establishes scheduling and mechanical effects. Preserve this distinction when saying “can,” “is instructed to,” or “will.” Do not present an illustrative tool choice as guaranteed behavior.

## Use diagrams for relationships that prose cannot show compactly

Verify every box and arrow against ownership, ordering, and data flow. Mark optional or model-selected paths appropriately; a condensed successful path should not imply unconditional execution or immediate availability.

Label executable steps with actual `module::function` or `Type::method` names and short `//` comments. Label files, queues, and model actions as those artifacts or actions. Function headings link to definitions. Preserve full names with line breaks instead of abbreviations.

Store code-native SVGs in `public/`, make them openable at full size, and provide a mobile variant for figures wider than 600px. Validate XML and inspect text bounds, arrows, captions, and legibility at both widths. Verify the diagram after prose corrections as well as after direct SVG edits.

For a comparison table, name one dimension per row and verify both cells at equivalent layers. Separate shared mechanisms from version differences. Prefer “inputs from extraction” over a broad input label if notes and existing files are also inputs elsewhere in the pipeline.
