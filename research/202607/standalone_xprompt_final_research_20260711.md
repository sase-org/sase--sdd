# Standalone xprompts: should `#!` be removed?

Date: 2026-07-11

## Executive conclusion

The two claims need to be separated.

1. **A standalone workflow is structurally defined by the absence of a `prompt_part`: confirmed.** That is the exact
   classifier used by SASE. However, the category also represents a meaningful orchestration boundary: it is a
   workflow that supplies its own step graph rather than content for the current agent prompt.
2. **Surrounding text can always be attached to the first agent: denied.** Many real standalone workflows have no
   direct agent, and others have conditional, repeated, parallel, or multiple agent steps. Even when one agent is
   present, changing its prompt is not equivalent to inserting a host prompt at that point in the workflow graph.

**Final recommendation: remove the `#!` invocation marker and use `#` for all xprompt references, but retain the
structural standalone-workflow category and its non-embeddability rule.** In other words: remove the redundant sigil,
not the semantic distinction. Do not implement implicit first-agent splicing as part of that simplification.

## Current semantics

`Workflow.prompt_kind()` in `src/sase/xprompt/workflow_models.py:261-267` classifies every resolved entry solely from
its shape:

| Kind | Shape | Current marker and behavior |
| --- | --- | --- |
| Simple xprompt | One `prompt_part` | `#name`; substitute prompt text |
| Embeddable workflow | A `prompt_part` plus optional pre/post steps | `#name`; execute pre-steps, substitute at the explicit boundary, then execute post-steps |
| Standalone workflow | No `prompt_part` | `#!name`; select and execute the workflow's own graph |

The marker is not required to discover the kind: resolution already loads the workflow and calls
`has_prompt_part()`. This is demonstrated by the compatibility path in
`src/sase/xprompt/workflow_runner.py:80-286`: a top-level `#name` that resolves to a no-`prompt_part` workflow still
executes, although it emits a deprecation warning asking for `#!name`.

Inline expansion is a separate context. Both `BEFORE #!sync AFTER` and `BEFORE #sync AFTER` are rejected today by
`workflow_executor_steps_embedded_expand.py:140-154` and
`main/query_handler/_embedded_workflows.py:100-107`. The marker changes the suggested correction, not whether the
workflow is actually embeddable.

The top-level flattener also does **not** currently preserve arbitrary surrounding prose. It selects one resolved
standalone workflow and returns only that workflow and its arguments. Its return type contains no prefix/suffix text,
and prompts not beginning with a reference do not enter this flattening path. Therefore the proposed splice is a new
composition rule, not a relaxation of behavior that already exists.

## Why the first-agent rule is not general

The resolved catalog on 2026-07-11 contains nine standalone workflows:

| Direct agent shape | Workflows |
| --- | --- |
| No direct agent (6) | `eval_ifs_loops`, `eval_parallel`, `sase/audit_recent_bugs`, `sase/audit_recent_improvements`, `sase/pylimit_split`, `sase/refresh_docs` |
| One agent (2) | `new_pr_desc`, `sync` |
| Multiple agents (1) | `sase/fix_just` |

The six agentless workflows immediately disprove “always.” Some launch agents from Python, but recursively inspecting
arbitrary Python and modifying prompts constructed there would not be a defined workflow operation.

The remaining workflows expose further ambiguity:

- `sync` runs setup and sync Python steps first. Its only agent is conditional on conflicts and repeated during
  resolution. On the normal no-conflict path, caller prose would never reach an agent; on the conflict path it could be
  duplicated unless special iteration semantics were invented.
- `sase/fix_just` may launch lint and test fixers, either, or neither. Lexically choosing the first privileges one
  branch; broadcasting the text duplicates it.
- `new_pr_desc` has one mechanically identifiable agent, but that agent has a structured title/body contract consumed
  by a later apply step. Arbitrary caller prose can alter the contract and therefore the workflow's side effects.

Workflow steps may also be conditional, repeated, looped, nested in parallel branches, hidden, typed, or followed by
`finally` cleanup. “First in source order” does not imply first executed, exactly once, or executed at all.

Position is another semantic problem. In `BEFORE #workflow AFTER`, attaching both fragments to an interior agent does
not make `BEFORE` run before the workflow or `AFTER` run after it: earlier steps have already run, and later steps have
not. An explicit `prompt_part` is different because the workflow author deliberately defines the host-agent boundary
and its pre/post orchestration.

## Why swarm splicing and the dormant helper do not prove otherwise

Xprompt swarms do splice the caller's prefix and suffix around the first generated segment
(`src/sase/agent/xprompt_swarm.py:405-430`). That is safe because every swarm segment is agent-prompt text. YAML
workflows are heterogeneous graphs of agent, bash, Python, parallel, and control-flow steps, so the analogy does not
generalize.

`expand_workflow_for_embedding()` in `workflow_runner.py:588-650` is also weaker evidence than it first appears. It has
no callers in `src` or `tests`. For a workflow without `prompt_part`, it returns an empty prompt string, no pre-steps,
and all steps as post-steps. It neither captures caller prefix/suffix text nor finds or rewrites an agent step. It is
not an almost-complete implementation of the proposed behavior.

## Is the second marker still necessary?

No. The failed first-agent proposal establishes that no-`prompt_part` workflows must remain non-embeddable by default;
it does **not** establish that callers must spell them with a different marker.

The workflow structure already provides authoritative dispatch information. `#!` asks the caller to duplicate that
information and creates two wrong-sigil cases (`#` on standalone and `#!` on embeddable). It also leaks an
implementation detail—whether the author included `prompt_part`—into normal invocation syntax. Meanwhile `#` is not a
reliable signal for “text only”: embeddable `#` workflows may already execute substantial pre/post steps. The main
benefit of `#!` is visual call-site signaling, but it is not needed for correctness or disambiguation after catalog
resolution.

A one-marker design can preserve all meaningful safety rules:

1. Resolve `#name` and inspect the workflow shape.
2. If it has `prompt_part`, expand it at the call site as today.
3. If it lacks `prompt_part`, allow it only as the single run-defining workflow in a top-level wrapper. Workspace
   references and launch directives may still configure that run.
4. If arbitrary prose attempts to embed that workflow, fail with one marker-independent error explaining that it has
   no prompt boundary. Reject multiple run-defining workflows as ambiguous.
5. Keep `#!` temporarily as a compatibility alias during migration, then remove it from completion, canonical
   insertion, parsing, tests, and documentation.

This is nearly the legacy top-level `#standalone` behavior already present, with the deprecation direction reversed.
It simplifies syntax without claiming that all workflows are prompt fragments.

If a concrete workflow later needs caller-supplied prose, make the attachment explicit and author-controlled: add a
typed workflow input consumed by the intended agent, add a real `prompt_part`, or define a workflow-level contract that
names the target and specifies skip/repeat/parallel behavior. Do not infer the target from source order.

## Final recommendation

**Yes: retire `#!` and standardize invocation on `#`. No: do not eliminate standalone execution semantics or prepend
and append arbitrary prose to the first agent.** Retain `STANDALONE_WORKFLOW` (or a clearer internal name such as
`RUN_WORKFLOW`) as the structural classification for workflows without `prompt_part`, and retain fail-fast rejection
when such a workflow is used inside prose.

This conclusion preserves the strongest point from each prior report: standalone workflows are genuinely not
universally embeddable, while the bang marker itself is redundant with resolved workflow structure.

## Verification

- Inspected parsing, classification, top-level flattening, inline expansion, completion/display, swarm expansion, and
  the history that introduced `#!`.
- Queried the effective resolved catalog and verified nine standalone workflows with direct-agent counts of 0, 1, or
  2 as summarized above.
- Ran 145 focused tests covering reference parsing, flattening, inline rejection, standalone launch behavior, catalog
  display/completion, and dry expansion: all passed (four expected legacy-marker warnings).
