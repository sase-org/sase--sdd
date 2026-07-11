# Reconsidering standalone xprompt workflows and the `#!` marker

Date: 2026-07-11

## Question

Should SASE remove the concept and syntax of standalone xprompt workflows—YAML workflows invoked with `#!name` because
they have no `prompt_part`—and use `#name` for every xprompt instead? In particular, can arbitrary text surrounding a
standalone workflow reference simply be prepended/appended to the first agent prompt in that workflow?

## Executive conclusion

The premise is partly right but the proposed generalization does not hold.

- It is true that the parser does not technically need two markers to identify workflow kind. The resolved catalog
  already classifies a name by whether its workflow has a `prompt_part`, and the legacy top-level `#standalone` form
  still executes today with a warning.
- It is also true that a narrow workflow made only of an unconditionally executed, single first agent could absorb
  surrounding prose into that agent's prompt.
- It is not true that every standalone workflow has a usable "first agent," or that changing the prompt of whichever
  agent happens to come first preserves the workflow's meaning. Standalone workflows are arbitrary pipelines, not
  prompt fragments.

The `#!` marker is therefore not enforcing an accidental implementation limitation. It identifies a real semantic
boundary: `#` substitutes content into the current agent prompt, while `#!` selects and executes a workflow as the run.

## What the code currently means by standalone

All xprompts are represented internally as `Workflow` objects. A simple Markdown xprompt becomes one `prompt_part`
step. `Workflow.prompt_kind()` then gives a complete three-way classification:

| Kind | Shape | Reference behavior |
| --- | --- | --- |
| Simple xprompt | Exactly one `prompt_part` | Inline text substitution |
| Embeddable workflow | Has one `prompt_part`, possibly with pre/post steps | Run pre-steps, substitute the part into the host agent, then run post-steps |
| Standalone workflow | Has no `prompt_part` | Execute its own step graph |

The classification is in `src/sase/xprompt/workflow_models.py:180-267`. Validation permits at most one `prompt_part`
(`src/sase/xprompt/workflow_validator_checks.py:39-96`), so the distinction is structural and unambiguous.

Inline workflow expansion explicitly rejects either a standalone workflow referenced as `#name` or any `#!name` in an
agent prompt (`src/sase/xprompt/workflow_executor_steps_embedded_expand.py:140-154`). Conversely, the top-level
anonymous-workflow flattener selects exactly one no-`prompt_part` workflow and replaces the anonymous wrapper with that
workflow (`src/sase/xprompt/workflow_runner.py:80-141,144-286`).

This behavior was deliberate rather than incidental:

- On 2026-04-29, `.sase/sdd/research/202604/standalone_workflow_xprompt_split.md` recommended surfacing the already
  existing `has_prompt_part()` partition.
- On 2026-04-30, commits `1ea4c04cb`, `eae4c6523`, and `d726b0676` added explicit `#!` execution, rejected unsafe inline
  passthrough, and made catalog/completion insertion kind-aware.
- Compatibility remains: top-level `#standalone` still flattens but emits "use `#!standalone`". The warning is defined
  at `src/sase/xprompt/workflow_runner.py:32-39` and covered at
  `tests/test_xprompt_processor_workflow_flatten.py:168-184`.

## Testing the proposed "attach it to the first agent" rule

### 1. Many workflows have no agent step

The resolved catalog on 2026-07-11 contains nine standalone workflows. Six have no direct `agent` step:

| Workflow | Relevant shape | Result of the proposed rule |
| --- | --- | --- |
| `eval_ifs_loops` | Bash-only conditional/loop test pipeline | No first agent exists |
| `eval_parallel` | Parallel bash/Python and bash verification | No first agent exists |
| `sase/audit_recent_bugs` | Three Python steps; one launches agents internally | No workflow agent prompt exists to modify |
| `sase/audit_recent_improvements` | Three Python steps; one launches agents internally | No workflow agent prompt exists to modify |
| `sase/pylimit_split` | One Python step that constructs and launches child prompts | No workflow agent prompt exists to modify |
| `sase/refresh_docs` | Three Python steps; one constructs and launches child prompts | No workflow agent prompt exists to modify |

An implementation could discard surrounding text for these workflows, invent a new agent step, or recursively inspect
arbitrary Python to find launched prompts. None is equivalent to embedding text in a known prompt part.

### 2. The first agent may be conditional, repeated, or one of several targets

The remaining examples also refute a general first-agent rule:

- `sync` runs two Python steps and only runs its `resolve` agent if conflicts exist. That agent is repeated up to 20
  times until resolution. On a clean sync the surrounding text would never reach an agent; on a conflicted sync it
  would be repeated unless special first-iteration semantics were invented.
- `sase/fix_just` performs checks and may launch a lint fixer, a test fixer, both, or neither. Attaching all surrounding
  prose only to the lexically first agent would arbitrarily privilege lint work; attaching it to all selected agents
  would duplicate instructions and still change the meaning.
- `new_pr_desc` has one agent between setup and apply steps, so it is mechanically compatible, but arbitrary appended
  prose can conflict with its structured-output contract or alter what the final apply step writes.

Workflow steps also support conditions, loops, parallel branches, HITL, typed outputs, and `finally` cleanup. A source
order scan for the first `agent` does not define which agent executes first, how many instances execute, or whether it
executes at all.

### 3. Text position would no longer match execution position

For an invocation such as:

```text
BEFORE #!workflow AFTER
```

prepending/appending both fragments to a later internal agent means `BEFORE` does not run before the workflow: earlier
bash/Python steps have already executed. Likewise, `AFTER` runs before every workflow step that follows that agent.
The textual composition implies a before/after boundary around the whole reference, while the proposed rewrite puts
both fragments inside one interior node of the graph.

That mismatch is especially risky for workflows whose surrounding steps perform side effects based on typed agent
output. A `prompt_part` has well-defined orchestration semantics: pre-steps run before the host agent and post-steps run
after it. An inferred first-agent splice has no equivalent contract.

### 4. Surrounding material is not necessarily plain prose

The supported wrapper form commonly contains:

```text
#gh:sase #!sase/pylimit_split %auto
```

Here `#gh:sase` selects/prepares a workspace and `%auto` is a launch directive. They should not be copied as literal
text into `pylimit_split`'s first child prompt. The current launch pipeline resolves directives and VCS context at their
own layers, then flattens the one selected standalone workflow. It even carries model and HITL overrides as internal
workflow metadata rather than prompt text (`src/sase/xprompt/workflow_runner.py:179-205`).

The current flattener returns only `(workflow, positional_args, named_args)`; it does not return prefix/suffix text.
Mixed-reference tests verify selection/ambiguity behavior, not text preservation
(`tests/test_xprompt_processor_workflow_flatten.py:203-314`). Implementing the proposal would therefore require a new
composition model, not a small relaxation of an existing invariant.

### 5. Multiple standalone references remain ambiguous

If a prompt contains two standalone workflow references, SASE currently raises because there is no defined way to
compose two workflow graphs (`src/sase/xprompt/workflow_runner.py:49-54,122-136`). Removing the marker or injecting text
into first agents does not answer whether the graphs should run sequentially, in parallel, nested, or share context.

## What is actually feasible

### Feasible: remove only the marker while preserving the semantic category

SASE could use `#name` for both kinds and dispatch by catalog metadata:

- If the name has a `prompt_part`, expand it inline.
- If the name has no `prompt_part` and appears in an executable top-level wrapper, select it as the standalone workflow.
- If the name has no `prompt_part` inside an agent prompt or arbitrary prose, reject it as non-embeddable.

This is technically straightforward because the legacy top-level behavior already does most of it. It would remove
syntax, but it would **not** remove the standalone concept or make standalone workflows generally embeddable. It would
also make the same token mean either text substitution or run selection only after catalog lookup, hiding a useful type
signal from readers, completion UI, validators, and tools.

### Feasible: add an explicit workflow composition contract

If there is demand for customizing a pipeline with caller prose, make that author-controlled. Options include:

- a typed `context`/`prompt` workflow input referenced by the intended agent step;
- an explicit workflow-level `accepts_surrounding_prompt` contract naming the target agent and defining behavior for
  skip/repeat/parallel cases;
- a wrapper workflow with a deliberate `prompt_part` boundary when the desired operation really is host-agent
  pre/post-processing.

These approaches make the author choose where caller text belongs and preserve validation. They should be introduced
only for concrete workflows that need the capability.

### Tradeoff of keeping `#!`

`#!` adds one concept and can require shell quoting because of history expansion. SASE already mitigates that with
single-quoted examples and kind-aware catalog/completion insertion. In exchange, it provides an immediate visual
distinction between "expand this text here" and "execute this pipeline," rejects category mistakes early, and lets a
mixed wrapper identify its one run-defining workflow explicitly. There are currently 480 `#!` occurrences across 120
files under `src`, `tests`, `docs`, `xprompts`, and `.sase/sdd`, so reversing it also carries nontrivial migration and
compatibility cost only about two months after the distinction was introduced.

## Final recommendation

Do **not** remove standalone xprompt workflows or the `#!` syntax.

The central claim—"we can always put surrounding text into the first agent"—is false for the current workflow model:
most resolved standalone workflows have no direct agent, and the ones that do may skip, repeat, or branch across agent
steps. More importantly, an internal agent node is not semantically equivalent to the explicit host-prompt boundary
provided by `prompt_part`.

Keep the current rule:

- `#name` means inline/embeddable prompt contribution.
- `#!name` means select and execute a no-`prompt_part` workflow.

If syntax reduction remains desirable, the only safe simplification is to accept `#name` for top-level standalone
selection while retaining the standalone classification and rejecting inline embedding—but that is already the legacy
compatibility behavior and is less explicit than `#!`. Prefer an explicit typed workflow input for the narrower cases
where callers genuinely need to add prose to a particular workflow agent.
