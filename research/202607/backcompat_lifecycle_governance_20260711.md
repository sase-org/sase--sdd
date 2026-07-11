# Governing the lifecycle of backward-compatibility logic

Date: 2026-07-11

## Problem statement

SASE accumulates backward-compatibility ("backcompat") logic that never gets removed. A grep for
`deprecat|backcompat|back-compat|backward.compat|legacy` across `src/` touches **235 files** today, yet the project
has no external consumers, so essentially none of that logic protects a real caller. The shims survive because:

1. They are cheap to add and invisible once merged.
2. Their removal condition lives only in a human's head (or nowhere), so nobody knows *when* a given shim is safe to
   delete.
3. Nothing fails when a shim outlives its purpose — there is no forcing function.

The request is explicitly two-sided and should not be collapsed into "stop writing backcompat":

- We **want** agents to keep adding backcompat, because once SASE is popular, silently breaking real users is far
  worse than carrying a shim for a release or two.
- We want a **policy governing how/when** a shim is deprecated, and — most importantly — a mechanism that **guarantees
  every shim is eventually removed** rather than accreting forever.

So the real target is *lifecycle management*: make removal as guaranteed as introduction. A shim should be born with an
expiry condition and a tracked owner, and the system should refuse to let it quietly become permanent.

### A note on styles observed

The existing markers are maximally inconsistent — `# Backward compat`, `# Back-compat shim`, `# backward compat
defaults`, `"""Deprecated compatibility wrapper..."""`, `wraps_all: bool = False  # Deprecated`, prose in docstrings,
schema `"description": "Legacy/deprecated..."`, etc. None of them are machine-parseable and none carry a removal
trigger, an owner, or an introduction version. Any solution must first impose a single structured form.

## What SASE already gives us (build on these, do not reinvent)

The recommendation leans on four capabilities that already exist in this repo:

1. **Custom linters gated by `just check`.** `just lint` already runs `ruff + mypy + pyscripts + pyvision + pylimit +
   keep-sorted + SASE validation` (`Justfile:124`), and `just check` (`Justfile:245-252`) is mandatory before any code
   change lands. Adding one more project linter is an established, low-friction pattern — not a new mechanism.

2. **A self-cleaning, bead-keyed allowlist precedent.** `pyvision` accepts `--epic-symbol <bead_id>(<symbol>)`
   (`Justfile:152-153,375-376`) and is invoked as `BD_COMMAND=tools/sase_bead ... pyvision src/sase`. That means a
   linter in this repo already **queries the bead store to decide whether a code allowance is still legitimate**, and
   tells you to drop the entry when the bead is missing/closed or the symbol is now really used. This is almost exactly
   the forcing function we want, applied to backcompat instead of epic scaffolding.

3. **Beads as a dependency-aware tracker with agent execution.** `sase bead` supports `create / dep / ready / blocked /
   close / work` (`sase bead --help`). Dependencies + a `ready` state give us "this removal becomes actionable when its
   trigger fires," and `sase bead work` can *launch an agent to actually perform the removal*. Beads are git-backed
   event streams under `sdd/beads/`, so tracking state travels with the code.

4. **Bidirectional structured-file enforcement precedent.** `keep-sorted` (`Justfile:184`) already shows CI enforcing
   invariants on a declarative block. A backcompat registry can be validated the same way (every marker ↔ every
   registry row).

Caveat worth stating up front: SASE beads are **SDD-plan-shaped**, not generic issues. `sase bead create` requires a
`-T {plan(...),phase(...)}` type and a `--tier {plan,epic}` (`sase bead create --help`); they attach to plan files and
ChangeSpecs. A literal "one bead per shim" therefore does not fit the current bead model cleanly — see Option D and the
recommendation for how to handle that.

## Design dimensions

Any option can be scored on:

- **Removal guarantee** — does something *hard-fail* when a shim overstays, or is removal merely encouraged?
- **State location** — is the removal condition in the code, in an external registry, or in a tracker?
- **Authoring friction** — how much work to add a compliant shim (matters doubly because *agents* write most code and
  must comply reliably)?
- **Blast radius / time-bomb risk** — when the forcing function fires, does it block an unrelated author's PR, or is it
  routed to an owner?
- **Governs introduction too?** — does it also answer *when a shim is even warranted*, not just when it dies?

## Options

### Option A — Policy + code review only (baseline)

A written "Backward-Compatibility Lifecycle" section in `CLAUDE.md`/`AGENTS.md` plus reviewer diligence.

- Removal guarantee: **none** (this is the status quo that produced 235 files).
- Friction: lowest. State: in humans' heads.
- Verdict: necessary as the *policy layer*, insufficient as the *enforcement layer*. Agents and reviewers both forget.

### Option B — Structured in-code expiry markers + a custom "time-bomb" linter

Impose one machine-parseable marker and let a new project linter fail `just check` when it expires.

```python
# BACKCOMPAT[bd-1234]: since=v0.4.0 remove_by=v0.6.0 reason="old 'commit' action alias"
```

A `_lint-backcompat` stage parses every marker and errors when `remove_by` (a released version, or a date) has passed,
or when a marker is malformed / missing a field.

- Removal guarantee: **strong** — expiry blocks CI, so a shim cannot silently become permanent.
- State: in the code, next to the logic it governs (no drift between marker and shim).
- Friction: low; one comment line, easily emitted by agents.
- Time-bomb risk: a date/version that lapses fails the *next* PR to touch CI, which may be an unrelated author. Needs
  routing/grace (see mitigations).
- Governs introduction? Partially — the linter can *require* the marker whenever backcompat-ish language appears, but
  it can't judge whether the shim was warranted.

### Option C — Central deprecation registry file

A single `deprecations.toml` (id, path, since, remove_by, reason, owner, tracking bead). CI validates bidirectional
consistency: every in-code marker has a row and vice versa, and fails on expiry.

- Removal guarantee: strong (same expiry check as B).
- State: centralized "single pane of glass" — easy to answer "what backcompat do we owe and when does it lapse?"
- Friction: higher — two edits per shim (code + registry), and drift between them is the main failure mode (mitigated
  by the bidirectional lint, cf. `keep-sorted`).
- Best when someone (release manager / audit agent) needs a portfolio view rather than per-file annotations.

### Option D — Bead-backed tracking (+ agentic removal)

Each shim references a **removal bead**; the marker carries the bead id; the linter cross-checks that the bead is still
open and, when the trigger fires, that the bead is `ready`. `sase bead work` can launch an agent to do the deletion.

- Removal guarantee: strong *if* paired with the linter (a bead alone is just a TODO nobody runs). The
  `--epic-symbol` precedent shows this exact loop already working in-repo.
- Unique upside: removal becomes **schedulable, assignable, and executable by an agent**, and dependencies express
  triggers ("removal is blocked until vX ships").
- Friction/fit: highest, and blocked by the plan-shaped bead model — you'd need a lightweight `chore`/`removal` bead
  type (or reuse phase beads) so a shim doesn't require a whole plan file. This is the main cost.

### Option E — Version-gated compatibility windows (Go `GODEBUG` / semver N-2 model)

Name every backcompat behavior and gate it on a supported-version window; removal is mechanical once the window
elapses. Mirrors Go's `GODEBUG` (named, defaulted by the `go` directive, guaranteed-then-removed) and the common
"support current + previous two releases" policy.

- Removal guarantee: strong and *predictable* — removal dates are derived from the release train, not chosen per-shim.
- Prerequisite: an actual release cadence with versions consumers pin to. SASE has prior work here
  (`202606/automated_semver_releases_consolidated.md`), but until releases + external consumers exist, the "window" is
  effectively zero — which is itself the useful insight for the current backlog.
- Best as the *policy that sets `remove_by`*, feeding B/C/D rather than standing alone.

### Option F — Scheduled agentic sweep (dogfood SASE)

A cron/xprompt audit agent periodically finds expiring/expired markers, opens or advances removal beads, and launches
removal agents / PRs. SASE already has `audit_recent_*` xprompt workflows and `CronCreate` scheduling, so removal can
be *actively driven* instead of waiting for CI to explode.

- Removal guarantee: medium on its own (an agent can be lazy or wrong), high as the *driver* in front of a hard linter
  backstop.
- Unique upside: converts "guaranteed to fail eventually" into "guaranteed to get *worked* proactively," which avoids
  the unrelated-author time-bomb problem.

## Comparison

| Option | Removal guarantee | Friction | State | Governs introduction | Agent-execution |
| --- | --- | --- | --- | --- | --- |
| A Policy only | none | lowest | heads | weakly | no |
| B In-code markers + linter | strong (CI hard-fail) | low | code | requires marker | no |
| C Registry file | strong | medium | central | requires row | no |
| D Bead-backed | strong (with linter) | high | tracker | requires bead | **yes** |
| E Version windows | strong+predictable | low | policy | **yes** | no |
| F Scheduled sweep | medium (driver) | low | agent | no | **yes** |

The columns are complementary, not competing: B is the *teeth*, E is the *clock*, D is the *work item*, F is the
*driver*, A is the *rule*. The failure mode of the codebase today is that only a weak version of A exists.

## Recommended solution

**Adopt a layered system: structured in-code markers, enforced by a bead-aware `just check` linter, with removal
windows anchored to releases and a scheduled agent that proactively works the removals — modeled directly on the
existing `pyvision --epic-symbol` loop.** Concretely:

**Layer 0 — Policy (the rule).** Add a "Backward-Compatibility Lifecycle" section to the agent instructions (with user
approval, since `memory/*` and provider shims are permission-gated). Two governing rules:

- *Introduction rule (governs "when"):* only add a shim when there is a consumer to protect. Encode a **compatibility
  baseline** — the oldest version/state we owe compatibility to. Below the baseline we owe nothing. Pre-popularity the
  baseline is "now," so the default window is short (or the shim is simply not added).
- *Birth-certificate rule (governs "guarantee"):* **no shim merges without a removal trigger and a tracking id.** A
  shim with no declared death is a lint error, full stop.

**Layer 1 — One structured marker (source of truth, in the code):**

```python
# BACKCOMPAT[bd-1234]: since=v0.4.0 remove_by=v0.6.0 reason="old 'commit' action alias"
```

with a `@backcompat(bead="bd-1234", since=..., remove_by=...)` decorator variant for whole callables. Machine-parseable,
greppable, trivially emitted by agents. This replaces all 235 ad-hoc styles.

**Layer 2 — A custom `_lint-backcompat` stage in `just lint`/`just check` (the teeth).** Built exactly like the other
project linters and run with `BD_COMMAND=tools/sase_bead` as pyvision is. It fails the build when:

- a backcompat marker is malformed or missing `remove_by`/`bead`;
- `remove_by` has elapsed against the current released version (or date);
- the referenced bead is closed while the marker still exists, or open with no marker (bidirectional, self-cleaning —
  identical spirit to `--epic-symbol` and `keep-sorted`);
- (opt-in) heuristic backcompat language (`backward compat`, `legacy`, `deprecated`) appears with no structured marker,
  to stop new ad-hoc shims regressing.

This is the piece that makes removal *guaranteed* rather than *hoped for*.

**Layer 3 — Beads as the tracked work item.** Each shim's `remove_by` is expressed as a bead whose trigger (a
dependency on "vX released," or a dated milestone) flips it to `ready`; `sase bead work` launches an agent to perform
the deletion. Because the current bead model is plan-shaped, introduce a lightweight removal/chore bead type (or a
convention over phase beads) so a shim doesn't require a full plan file — this is the one net-new bit of plumbing.

**Layer 4 — Version anchor (the clock).** Once semver releases exist, set `remove_by` from the release train (e.g.
supported = current + previous two), so removal dates are mechanical, not bespoke. Until then, the compatibility
baseline of "now" makes most windows short — which directly licenses cleaning up today's backlog.

**Layer 5 — Scheduled sweep agent (the driver).** A cron/xprompt audit agent lists ready removal beads and
soon-to-expire markers, opens PRs / launches removal agents, and nudges owners *before* the hard linter fails. This
converts the CI time-bomb into proactively-worked hygiene and avoids blocking an unrelated author's unrelated PR.

### Immediate backlog remediation

Because the problem statement says no current consumer needs the existing logic, do not wait for the system to age out
235 files. Declare the compatibility baseline at the current version and run a **one-time sweep to delete dead shims
now** (the shims protect nobody). Then require the marker+linter+bead system for anything added afterward, so the
backlog cannot re-form.

### Why this and not a single mechanism

- Markers-only (B) hard-fails on a random future PR author — bad blast radius. Beads+sweep (D/F) route the work to an
  owner *before* that.
- Beads-only (D) is a TODO nobody runs unless a linter makes ignoring it fail — so the linter (B) is non-negotiable as
  the backstop.
- Registry-only (C) duplicates state and drifts; in-code markers keep the truth next to the logic, with the bead as the
  work handle rather than a second copy of the metadata.
- The design is *already proven in-repo*: `pyvision --epic-symbol` is this exact loop (bead-keyed, lint-enforced,
  self-cleaning) for a different kind of temporary code. Generalizing a known-good pattern beats inventing one.

### Risks and mitigations

- **Time-bomb hits an unrelated PR.** Mitigate with Layer 5 (proactive sweep) + a short grace/escalation window in the
  linter so the first lapse warns and only a grossly-overdue shim hard-fails.
- **Agents don't comply.** Ship a `#backcompat` xprompt/skill that emits the marker *and* opens the bead in one step,
  and encode Layer 0 in the always-loaded gotchas so every runtime follows it uniformly (no per-runtime special-casing).
- **Bead-model fit.** The lightweight removal bead type is the main build cost; if it proves heavy, fall back to
  Option C's registry as the tracking handle and keep only Layers 1–2 + 4–5.
- **Over-deletion of a shim that did protect something.** The introduction rule's compatibility baseline is the guard:
  only delete below the baseline; anything above it still owes compatibility.

## Verification

- Confirmed the scale of the problem: `rg -li 'deprecat|backcompat|backward.compat|legacy' src` → 235 files, with a
  representative sample of mutually-inconsistent, metadata-free marker styles.
- Confirmed the enforcement substrate: `Justfile:124,152-153,245-252,375-376` (custom linters gated by `just check`;
  `pyvision` run with `BD_COMMAND=tools/sase_bead` and `--epic-symbol <bead_id>(<symbol>)`), and the `pyvision`
  long-term memory documenting the self-cleaning, bead-keyed whitelist behavior.
- Confirmed the tracking substrate: `sase bead --help` (`create/dep/ready/blocked/close/work`) and `sase bead create
  --help` (plan/phase/epic-tier, plan-file/ChangeSpec-attached — hence the "lightweight bead type" caveat).
- Confirmed adjacent prior art in this research corpus: `202606/automated_semver_releases_consolidated.md` (release
  cadence that Layer 4 depends on).
