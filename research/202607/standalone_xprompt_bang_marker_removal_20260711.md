# Should we remove the standalone-xprompt concept (the `#!` marker)?

**Date:** 2026-07-11
**Author:** research agent (SASE)
**Question:** Bryan proposes eliminating the concept of *standalone* xprompt workflows —
i.e. dropping the `#!` invocation marker and always using `#`. His reasoning: the only
thing that makes a standalone workflow un-embeddable is that it has no `prompt_part`
slot, and that constraint isn't real because we could always **prepend** the text before
a `#!name` invocation and **append** the text after it onto the *first agent* of the
workflow.

**Bottom line (TL;DR):**

- **Claim 1 — "standalone xprompts exist because a no-`prompt_part` workflow has no slot
  to embed surrounding text into" → CONFIRMED.** That is exactly and *only* what the
  classifier keys on.
- **Claim 2 — "we can always prepend/append the surrounding text onto the first agent" →
  PARTIALLY CONFIRMED.** The mechanism is sound and already exists (the xprompt-swarm
  path does precisely this), but the word **"always" is false**: a large fraction of real
  standalone workflows have *no agent step at all* (pure bash/python automation) or their
  only agent step is conditional/hidden/not-first. There is no well-defined "first agent"
  to attach prose to in those cases.
- **Recommendation — YES, remove the `#!` marker and unify on `#`, but don't sell it as
  "always embeddable."** The marker is redundant with the `prompt_part` structure that
  already fully determines behavior, and its dominant effect today is a symmetric class of
  "you used the wrong marker" errors with no upside. Replace the two-directional marker
  validation with a single fail-fast error for the one genuinely-meaningless case
  (embedding a *no-agent* workflow mid-prose). Optionally adopt the prepend/append
  generalization for the well-defined case (workflows whose entry point is an agent).

---

## 1. Background: what `#!` actually means today

A "workflow-like" xprompt is classified by exactly one predicate — **does it contain a
`prompt_part` step?** (`src/sase/xprompt/workflow_models.py:261-267`)

```python
def prompt_kind(self) -> WorkflowKind:
    if self.is_simple_xprompt():   # 1 step, that step is a prompt_part
        return WorkflowKind.SIMPLE_XPROMPT
    if self.has_prompt_part():     # has a prompt_part among many steps
        return WorkflowKind.EMBEDDABLE_WORKFLOW
    return WorkflowKind.STANDALONE_WORKFLOW   # no prompt_part at all
```

| Kind | Source | Reference | Embeddable in prose? |
|------|--------|-----------|----------------------|
| `SIMPLE_XPROMPT` | `.md` (one `prompt_part`) | `#name` | yes — the body text is woven in |
| `EMBEDDABLE_WORKFLOW` | `.yml` **with** a `prompt_part` step | `#name` | yes — pre-steps run, `prompt_part` text is woven in at the call site, post-steps run after |
| `STANDALONE_WORKFLOW` | `.yml` **without** a `prompt_part` step | `#!name` | **no — blocked** |

The `prompt_part` step is the **author's explicit opt-in to embeddability**: it is the
designated slot where the surrounding prompt's text meets the workflow. A workflow with no
`prompt_part` has, by the author's own construction, no such slot — so `#!` is really just
a *call-site echo of a decision the author already made in the workflow file.*

The `#!` marker is fully plumbed across the stack (see the full surface map in
[Appendix A](#appendix-a--code-surface-of-the--marker)): lexing, classification,
display/insertion, TUI completion filtering, inline-expansion, and launch/validation.

---

## 2. Claim 1 — "the reason standalone xprompts exist is the missing `prompt_part`"

**Verdict: CONFIRMED.** This is literally the whole definition, not a side effect.

- Classification keys on nothing but `has_prompt_part()`
  (`workflow_models.py:261-267`, `:191-197`).
- The embed guard raises precisely when there is no `prompt_part`:

  ```python
  # src/sase/xprompt/workflow_executor_steps_embedded_expand.py:147-154
  if ref.is_standalone_marker or not workflow.has_prompt_part():
      raise WorkflowExecutionError(
          format_inline_workflow_reference_error(
              name=name, raw=ref.raw,
              has_prompt_part=workflow.has_prompt_part(),
          )
      )
  ```
  The same guard is duplicated at `src/sase/main/query_handler/_embedded_workflows.py:100-107`.

- The error text itself states the rationale
  (`src/sase/xprompt/workflow_executor_steps_embedded_types.py:40-44`):
  *"`#!name` is a standalone workflow and cannot be embedded inside inline prompt text.
  Run it as a standalone workflow instead."*

So Bryan's understanding of *why* the category exists is exactly right.

---

## 3. Claim 2 — "we can always prepend/append onto the first agent"

**Verdict: PARTIALLY CONFIRMED.** The mechanism is real and already implemented for one
class of xprompt; the failure is the universal quantifier ("always").

### 3a. The mechanism already exists — for xprompt *swarms*

`src/sase/agent/xprompt_swarm.py` does exactly what Bryan describes. When a swarm xprompt
(a `.md` body with top-level `---` separators) is embedded in a segment, the first body
segment is spliced in at the call site and the rest become follow-up agents
(`xprompt_swarm.py:424`, and for the leading-prose case `:458-487`):

```python
# _expand_embedded_xprompt_swarm_reference
first = segment[: ref.start] + sub_segments[0] + segment[ref.end :]
follow_ups = _prepend_inherited_vcs_ref(sub_segments[1:], _leading_vcs_ref_text(segment))
```

And the enabling library function for *workflows* already exists too — it just is not
wired into any live path:

```python
# src/sase/xprompt/workflow_runner.py:588-650  (expand_workflow_for_embedding)
# "If the workflow has no prompt_part, all steps are returned as post-steps."
```

`get_post_prompt_steps()` already returns **all** steps when there is no `prompt_part`
(`workflow_models.py:221-230`). **However, `expand_workflow_for_embedding` has no
non-test callers** — it is exported public API (`__init__.py:89,179`) that nothing in
`src/` uses. The live inline-embed paths *raise* instead (§2). So the plumbing to "embed a
standalone workflow" is ~half-built and dormant.

**Takeaway:** the capability is a small step away, not a rewrite. Bryan is right that the
constraint is not fundamental.

### 3b. Why "always" is false — the empirical reality of standalone workflows

I classified every built-in `.yml` workflow (`src/sase/xprompts/**`). Of the 18, only a
handful are standalone (no `prompt_part`), and here is what they actually look like:

| Workflow | Has agent step? | Shape | Can prose attach cleanly to "first agent"? |
|----------|-----------------|-------|--------------------------------------------|
| `eval_parallel.yml` | **No** | 13 bash/python steps exercising parallel joins | **No agent exists** |
| `eval_ifs_loops.yml` | **No** | pure bash steps exercising if/for/while | **No agent exists** |
| `steps/shared/check_changes.yml` | No | reusable `use: shared` step | N/A (not top-level referenceable) |
| `examples/.../tester.yml`, `improve_plan.yml` | mixed | example agent-families | example-only |
| `sync.yml` | Yes, but **1, conditional + hidden, and not first** | python `setup` → python `sync_attempt` → **agent `resolve` (`if: has_conflicts`, `repeat`)** → python `report` | **No** — the agent may never run |

`sync.yml` is the instructive real-world case. Its only agent step is `resolve`, which:
- is **not the first step** (two hidden python steps run first), and
- runs **only `if: {{ sync_attempt.has_conflicts }}`**.

So if a user wrote `please be careful #!sync then report back`, "prepend/append onto the
first agent" would attach *"please be careful"* / *"then report back"* to a step that
**does not run on the common (no-conflict) path** — the prose silently vanishes. That is a
worse outcome than today's fail-fast error.

For `eval_parallel` / `eval_ifs_loops` there is **no agent at all** — there is literally
nowhere to put the surrounding text. Embedding a pure-automation workflow in prose is
semantically meaningless ("do X, then <run this bash pipeline>, then do Y" — instructions
to *whom*?).

### 3c. The deeper reason the swarm case works and the workflow case doesn't

Swarm segments are **homogeneous** — every `---` segment is agent prompt *text*, so
splicing prose into the first segment is always well-defined. YAML workflow steps are
**heterogeneous** — `bash` / `python` / `agent` / `parallel` — so "the first agent" is:
- **absent** (pure-automation workflows),
- **ambiguous** (if the first agent-bearing step is a `parallel` fan-out — which branch?), or
- **misleading** (if it's conditional/hidden/deep, as in `sync`).

There's also a subtler authorship point: for embeddable workflows the `prompt_part`
lets the *author* choose exactly where surrounding text lands. Auto-picking "first agent"
removes that control — and for a standalone workflow the author has, by omitting
`prompt_part`, explicitly declared "there is no such slot." The proposal overrides that
declaration.

**Net:** Bryan's mechanism works for the class of workflows that *have* a clear agent
entry point, which is exactly the class the swarm code already handles. It does **not**
generalize to no-agent or conditional-entry workflows, so it cannot be the *universal*
replacement for the constraint — a residual "can't embed this" case always remains.

---

## 4. Is the `#!` distinction earning its keep?

### 4a. The marker is redundant with structure

Behavior is already 100% determined by `has_prompt_part()`. The marker adds no
information the workflow file doesn't already carry; it only asks the *caller* to restate
it. When caller and structure disagree, you get an error — in **both** directions:

- `#name` on a standalone workflow → deprecation warning / embed error
  (`workflow_runner.py:32-39`; `format_inline_workflow_reference_error` `:45-48`).
- `#!name` on an embeddable workflow → hard error
  (`invalid_explicit_standalone_message`, `workflow_runner.py:42-46`;
  `_invalid_explicit_xprompt_message`, `xprompt_swarm.py:374-379`).

Every one of these errors is *purely* "you picked the wrong sigil." None of them catches a
real defect that would exist under a single-marker world. This is a self-inflicted error
class.

### 4b. The marker's scope has already churned twice in ~2 months

The `#!` concept is young and unstable (git history):

| Date | Commit | Change |
|------|--------|--------|
| 2026-04-29 | `74d7bd318` (sase-1g.1) | Introduce shared reference model + `SIMPLE`/`EMBEDDABLE`/`STANDALONE` classification |
| 2026-04-30 | `1ea4c04cb` (sase-1g.2) | `#!` standalone execution; keep legacy `#standalone` with deprecation warning |
| 2026-04-30 | `eae4c6523` (sase-1g.3) | Enforce inline-expansion safety for standalone workflows |
| 2026-05-04 | `03b5367e6` | **Require** `#!` for multi-agent (swarm) markdown xprompts too |
| 2026-05-04 | `79cde7233` | Add ability to *embed* multi-prompt (swarm) prompt-parts inline |
| 2026-07-08 | `494ba4ecf` | Rename "multi-agent xprompt" → "xprompt swarm" |

The current state has **reverted** the 2026-05-04 decision: swarms now use `#`, and `#!`
is *rejected* for them (`xprompt_swarm.py:591-610`; `docs/xprompt.md:256-257`:
*"`#!name` is still accepted for xprompt swarms, but new prompts should use `#name`."*).
So in ~10 weeks the bang went: *standalone-only → also-required-for-swarms →
standalone-only again.* A distinction that keeps being re-scoped is a distinction that
isn't paying rent.

### 4c. What keeping it buys (the honest counter-case)

- **Call-site legibility:** `#!deploy` signals "heavyweight process" without opening the
  file. Minor, and already undercut by the fact that `#embeddable_wf` *also* silently runs
  pre/post steps — `#` already hides process-ness.
- **Accidental-embed safety:** today, referencing a heavy workflow mid-prose fails fast
  with a clear message. Under "auto-attach to first agent," that becomes a silent surprise
  (§3b). This is a real regression *only if* we adopt auto-attach without a guard — which
  is exactly why the recommendation keeps a guard.

---

## 5. Options

**Option 0 — Status quo.** Keep `#!`. Cost: the redundant marker, the symmetric
wrong-marker error class, and continued churn.

**Option 1 — Remove the `#!` marker; keep the "no-embed" rule (minimal).**
- Accept `#` everywhere; delete `#!` from the lexer, classifier display, completion
  filter, and both validation directions.
- `#standalone` as a *sole* reference or rollover (`#gh:sase … #standalone`) runs the
  workflow (this already works today, just warns — drop the warning).
- Embedding a no-`prompt_part` workflow *mid-prose* still errors — but with **one** clear
  message ("this workflow has no prompt_part slot; run it standalone"), replacing the
  two-directional marker validation.
- Cheapest; fully honors "get rid of `#!`, always use `#`"; keeps fail-fast safety.

**Option 2 — Option 1 + implement Bryan's prepend/append (full vision).**
- Additionally, when a standalone workflow *with a well-defined agent entry point* is
  embedded, wire up the dormant `expand_workflow_for_embedding` path: prepend leading prose
  / append trailing prose onto that agent (treat the first top-level agent step as an
  implicit `prompt_part`).
- Still error for the residual ill-defined cases: **no** agent step, or the first
  agent-bearing step is `parallel`/conditional/hidden.
- More work and more surface area, but delivers the "embed anything" ergonomics Bryan is
  after for the cases where it's meaningful.

---

## 6. Recommendation

**Remove the standalone-xprompt *marker* (`#!`) and unify on `#` — Option 1 now, with
Option 2 as a follow-up if/when embedding a process into prose proves useful in practice.**

Reasoning:

1. **The marker is redundant.** `has_prompt_part()` already determines behavior
   end-to-end; `#!` only asks the caller to restate the author's decision and punishes
   mismatches. That is a pure friction / error class with no compensating bug-catching
   value.
2. **Bryan's premise is correct that the constraint isn't fundamental** — the swarm code
   already proves prepend/append works, and the workflow-side plumbing
   (`expand_workflow_for_embedding` + `get_post_prompt_steps`) is 90% there but dormant.
3. **But don't market it as "always embeddable."** A real, non-trivial fraction of
   standalone workflows (`eval_parallel`, `eval_ifs_loops`, and `sync`'s common path) have
   no meaningful agent to attach prose to. Keep **one** fail-fast error for that residual
   case — it is strictly less friction than today's symmetric two-marker validation, and
   it preserves the accidental-embed safety net.
4. **Low switching cost / already-churning surface.** The feature is ~10 weeks old, has
   already been re-scoped twice, and the deprecation machinery for `#standalone → #!` is
   still in place — so this reverses a recent, unsettled direction rather than unwinding a
   load-bearing one.

The cleanest mental model to land on: **"`#` references an xprompt; whether it embeds as
text or runs as a process is decided by the workflow's own structure (`prompt_part` or
not), and the only hard rule is that a process with no text slot and no agent entry can't
be spliced into the middle of prose."** One marker, one rule, no wrong-sigil errors.

---

## Appendix A — Code surface of the `#!` marker

Removing the marker touches (from the full map):

- **Lexing:** `src/sase/xprompt/_parsing_references.py:18-19,36-51,78-81,156-186`
  (marker fragment, `XPromptReferenceMarker`, `is_standalone_marker`); the older
  `_WORKFLOW_REF_PATTERN` in `workflow_executor_steps_embedded_types.py:21-25`; and
  `src/sase/ace/tui/widgets/_xprompt_arg_assist_detection.py:23`.
- **Classification:** `workflow_models.py:26-31,180-267` (`WorkflowKind`, `prompt_kind`,
  `has_prompt_part`, `get_pre/post_prompt_steps`).
- **Display / insertion:** `src/sase/xprompt/reference_display.py` (whole file);
  consumers in `xprompt_handler.py`, `_catalog_structured.py`, `_catalog_models.py:62`,
  `xprompt_select_modal.py`, `xprompt_browser_*.py`, `_xprompt_arg_assist_*`.
- **Completion / inline-expansion (TUI):** `xprompt_completion.py:43-54` (menu filter),
  `xprompt_inline_expansion.py:40,93-104`, `xprompt_select_modal.py:244-254,467-482`.
- **Launch / validation / errors:** `workflow_runner.py:32-141,207-268`,
  `workflow_executor_steps_embedded_expand.py:140-154`,
  `main/query_handler/_embedded_workflows.py:100-107`,
  `main/query_handler/special_cases.py:116,132-134`,
  `ace/tui/actions/agent_workflow/_workflow_exec.py:78,111-118`,
  `sdd/_expand.py:63-81`, `agent/xprompt_swarm.py:358-379,591-610`.
- **Dormant enabling API (would be wired up for Option 2):**
  `workflow_runner.py:588-650` (`expand_workflow_for_embedding`), backed by
  `workflow_models.py:210-230`.
- **Tests:** ~66 test files reference `#!`/standalone; the load-bearing ones are
  `tests/test_xprompt_references.py`, `tests/test_workflow_executor.py:527,548`,
  `tests/test_special_cases.py:62,79,99`,
  `tests/test_xprompt_processor_workflow_flatten.py`, `tests/test_expand_for_spec.py`,
  `tests/ace/tui/widgets/test_xprompt_completion.py`,
  `tests/ace/tui/widgets/test_xprompt_inline_expansion.py`,
  `tests/ace/tui/modals/test_xprompt_select_modal.py`,
  `tests/main/test_xprompt_handler.py`.
- **Docs to update:** `docs/xprompt.md` (`:5,99-100,205-210,256-257,268-281,828,905-921`),
  `docs/workflow_spec.md:158-195`, `docs/cli.md:187`, `docs/architecture.md:43`,
  `docs/workspace.md:151,169`, `memory/xprompts.md:11`.

## Appendix B — Built-in standalone workflow inventory (empirical)

Only these built-in `.yml` workflows lack a `prompt_part` (are "standalone"):
`eval_parallel` (no agent), `eval_ifs_loops` (no agent), `sync` (1 conditional+hidden
agent, not first), plus non-top-level/example files `steps/shared/check_changes`,
`examples/agent_families/tester`, `examples/agent_families/improve_plan`. Every other
built-in `.yml` (`pr`, `commit`, `propose`, `git`, `file`, `json`, `fork`, `fork_by_chat`,
`with_feedback`, `with_q_and_a`, `mentor`, `make_mentor_changes`) has a `prompt_part` and
is already `#`-embeddable.
