# Shared SDD Clone Research

Date: 2026-07-09

## Question

SASE currently prepares a workspace-local SDD companion repo clone for each agent workspace. Could each machine instead
keep one local clone of the SDD repo and let agents share it, syncing and pushing when needed? Can the concurrent writes
problem be solved when multiple agents mutate SDD files at the same time?

## Short Answer

Do not replace the current per-agent writable SDD checkout with a single shared mutable checkout as the default. It is
not strictly necessary to have a full independent object clone per agent, but agents do need either:

1. an isolated writable working tree per agent, or
2. a mandatory SASE-owned write transaction/broker that serializes every SDD mutation.

The best next design is a machine-local SDD cache or bare mirror plus per-agent linked worktrees/reference clones. That
keeps local write isolation while cutting duplicate network fetches and much of the duplicated Git object storage. A
single shared writable checkout is viable only if every SDD writer, including plan approval, bead mutation, commit
finalization, agent-authored research edits, and human CLI commands, goes through one lock/transaction path.

## Current Behavior

The documented separate-repo mode says the companion repo clone lives at `{workspace}/.sase/sdd`, and numbered
workspaces get their own best-effort clone or fast-forwarded copy from the primary checkout. See
`docs/sdd_storage.md:43`.

The resolver enforces that model:

- `src/sase/sdd/store.py:130` says separate-repo storage resolves to a workspace-local effective working tree.
- `src/sase/sdd/store.py:367` returns `{workspace}/.sase/sdd` for `separate_repo`.
- `src/sase/sdd/store.py:179` ensures a workspace SDD clone when a materialized separate-repo record exists.
- `src/sase/axe/run_agent_runner_setup.py:73` runs `ensure_workspace_sdd_clone()` after workspace preparation.

The clone helper is intentionally best-effort:

- `src/sase/sdd/_store_link.py:19` creates or refreshes a workspace-local clone.
- Existing clones are pulled with `git pull --rebase` at `src/sase/sdd/_store_link.py:103`.
- If remote clone fails, it may clone from the primary checkout at `src/sase/sdd/_store_link.py:65`.
- Failures are warnings, not agent-launch blockers.

Commits are local first:

- `src/sase/sdd/_commit.py:212` stages and commits changed SDD paths in a local Git repo.
- `src/sase/sdd/_commit.py:274` wraps that for SDD stores and pushes only after a local commit.
- The docstring at `src/sase/sdd/_commit.py:283` states that push failure preserves the local commit.
- `src/sase/sdd/_commit.py:428` may push asynchronously, which is safe enough for per-agent clones but unsafe for a
  single shared mutable checkout unless the async push is also serialized.

There are direct filesystem writers:

- `src/sase/sdd/_write.py:55` writes prompt snapshots with `Path.write_text()`.
- `src/sase/sdd/_write.py:117` updates Q&A blocks with `Path.write_text()`.
- `src/sase/axe/run_agent_exec_plan_accept.py:291` materializes the store, writes prompt/plan files, then commits them.

Bead mutation is also a shared-SDD writer:

- Python opens bead projects from the resolved SDD store and auto-commits non-in-tree bead mutations.
- The Rust core mutation path loads the full bead store into memory at
  `../sase-core/crates/sase_core/src/bead/mutation.rs:145` and saves it later at
  `../sase-core/crates/sase_core/src/bead/mutation.rs:766`.
- `../sase-core/crates/sase_core/src/bead/jsonl.rs:92` writes `issues.jsonl` through an atomic rename, but there is no
  store-wide mutation lock around read/modify/write.

Git and SQLite both protect against some low-level corruption, but not against SASE-level lost updates. Git worktrees
are explicitly designed for multiple working trees sharing one repository object store; each linked worktree has
separate per-worktree state such as `HEAD` and `index` (Git docs: https://git-scm.com/docs/git-worktree). Git push
rejects non-fast-forward updates by default to prevent history loss and requires fetch/merge or rebase before retry
(Git docs: https://git-scm.com/docs/git-push/2.0.5). SQLite serializes writers and WAL allows readers and writers to
overlap, but only one writer can be active at a time (SQLite docs: https://sqlite.org/isolation.html and
https://sqlite.org/wal.html).

## Local Cost Measurement

On this machine, the current workspace's `.sase/sdd` clone is about 472 MiB.

`git count-objects -vH` reports about 213 MiB of packed Git objects in that clone. The checked-out `research/`
directory alone is about 211 MiB. The managed `sase` workspace root currently has 7 detected workspace-local SDD clones,
each roughly 466-473 MiB, so the duplicated disk footprint is real.

This argues for a cache/worktree optimization. It does not by itself justify a single shared mutable checkout.

## What Breaks With One Shared Writable Checkout

A single normal checkout has one working tree and one Git index. Concurrent uncoordinated agents would race in ways the
current per-workspace clone model avoids:

- Agent A and B can both write different files, then Agent A's `changed_sdd_files()` may see both files and commit B's
  work under A's commit marker. B may then see a clean tree and skip its own commit.
- Two agents writing the same path, such as the same plan slug or Q&A update, can overwrite each other before Git ever
  gets a chance to report a merge conflict.
- Concurrent `git add` / `git commit` / `git pull --rebase` operations share `.git/index` and repository state. Git's
  lock files prevent some corruption, but the user-visible result is failed commands, mixed commits, or a rebase state
  left behind for the next agent.
- Current async push mode can continue using the shared repo after the originating SDD operation returns. In a shared
  checkout, that background push can overlap the next agent's local writes or rebase.
- Bead creation can lose updates: two processes can load the same `next_counter` and issue list, generate the same next
  ID, and atomically rename their own `issues.jsonl` result. Atomic rename prevents a torn file, but not a lost update.
- Event stream writes use deterministic filenames and a path-level temporary file. Without a store-wide lock, two
  writers to the same stream can clobber each other's temporary file or final stream.

The important distinction: Git and atomic file writes protect storage integrity; they do not preserve logical ownership
of a SASE mutation.

## Can The Concurrent Writes Problem Be Solved?

Yes, but only by treating SDD writes as transactions. A correct shared-checkout design needs a single SDD mutation lock
or broker keyed by the companion repo identity, not by the current workspace path.

A minimal lock protocol would be:

1. Resolve a stable store identity from the SDD record, preferably provider + host + repo or remote URL.
2. Acquire an exclusive cross-process lock outside the checkout, for example under
   `$XDG_STATE_HOME/sase/sdd-locks/<store-id>.lock`.
3. Verify the shared checkout is clean and not in a merge/rebase/cherry-pick state. If dirty, either attribute the dirty
   state to a known unfinished transaction or stop.
4. Fetch and fast-forward/rebase before writing.
5. Run the actual SDD mutation callback: plan files, prompt snapshot, Q&A update, research file, bead JSONL/event
   mutation, config change, or generated guide refresh.
6. Stage only the mutation's declared pathspecs.
7. Commit immediately with the correct runtime tags and artifact commit marker.
8. Push while still holding the lock. On non-fast-forward rejection, fetch/rebase the just-created commit and retry.
9. On conflict, leave the commit reachable on a conflict branch or named stash and surface a clear recovery path.
10. Release the lock.

This solves same-machine writers only if every writer obeys it. It does not remove cross-machine conflicts; those still
require the fetch/rebase/push retry loop and manual conflict handling when two machines edit the same file.

A plain advisory lock is not enough if agents can directly edit `SASE_SDD_DIR` with shell tools and then run arbitrary
Git commands. For a single shared mutable checkout to be safe, SASE would need to make write APIs first-class and route
skills/CLI/planning workflows through them. Human or agent direct edits would either need to be unsupported, detected as
dirty shared state, or isolated in a per-agent scratch/worktree.

## Better Design Options

### Option A: Shared Bare Mirror + Per-Agent Worktree Or Reference Clone

This is the recommended direction.

Keep one machine-local mirror/cache of the SDD companion repo, then give each agent an isolated writable view:

- Use `git clone --reference-if-able` from the cache, or
- use `git worktree add --detach` from a local bare/non-bare cache, then commit in detached HEAD and push `HEAD` to the
  companion branch after rebase, or
- create per-agent short-lived branches/worktrees and rebase/push to `master`.

Benefits:

- Preserves per-agent working tree and index isolation.
- Keeps local commits and push failures scoped to the responsible agent.
- Avoids most repeated network clone cost.
- Can share Git object storage.
- Requires much less invasive mutation locking than a single shared checkout.

Limitations:

- Checked-out file contents are still duplicated unless sparse checkout or partial clone is added.
- Git worktree cannot casually check out the same branch in many worktrees, so the SDD flow should use detached HEAD or
  per-agent branches and push explicitly.
- Cross-machine push conflicts still exist and still need rebase/retry/manual conflict handling.

### Option B: Single Shared Mutable Checkout + SDD Transaction Lock

This is viable but heavier.

All SDD writes would need to move behind a transaction API. The lock would have to cover both file mutation and Git
mutation, including synchronous push. Async push would need to become a queued operation owned by the same broker.

Benefits:

- Maximum disk savings: one working tree per machine.
- One local checkout to keep fresh.

Costs:

- Serializes all SDD writes on the machine.
- Makes long network pushes block later local SDD writes unless push is handed to a queue that still owns repo state.
- Requires touching many write paths and probably Rust core bead mutation.
- Direct agent/human edits to `SASE_SDD_DIR` become unsafe unless prohibited or converted into brokered commands.
- More complicated failure recovery: stale locks, interrupted rebases, dirty shared checkout attribution, and commit
  marker correctness.

### Option C: Shared Read-Only Store + Transactional Overlay

Expose a shared read-only SDD store to agents and write proposed changes into per-agent overlays. A SASE broker applies
overlays in order.

This is conceptually clean but probably too much machinery for now. It would require changing how agents write research
notes and plans, and it would make ordinary shell-level editing less direct.

## Recommendation

Do not make a single shared mutable SDD checkout the default. The correctness risk is high because current SDD writes
are ordinary file writes plus local Git commands, and bead mutations are read/modify/write operations without a
store-wide lock.

Do pursue a machine-local SDD cache:

1. Add a machine-local cache directory derived from the SDD remote URL.
2. Refresh that cache under a short cache lock.
3. Materialize each agent's `.sase/sdd` from the cache as a reference clone or linked worktree.
4. Keep the current local-commit-first, push/rebase/retry behavior per agent.
5. Consider sparse checkout for heavy directories if most agents only need prompts/plans/beads metadata.

If maximum disk reduction becomes more important than write isolation, implement the transaction/broker design as a
separate explicit mode after all SDD writers are inventoried and routed through one API. That design should live in or
near the Rust core boundary for shared domain behavior, with Python/TUI code using a thin adapter.

## SASE Variables To Set

Suggested variables for this research run:

- `recommendation=shared_cache_per_agent_worktree`
- `shared_mutable_checkout_safe=no`
- `single_checkout_possible_with_lock=yes_but_not_recommended`
- `research_path=.sase/sdd/research/202607/shared_sdd_clone_research.md`
