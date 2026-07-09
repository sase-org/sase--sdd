---
create_time: 2026-07-09
status: research
---

# Shared Per-Machine SDD Clone vs. Per-Workspace Clone

## The question

Today, preparing a `sase_<N>` agent workspace clones the SDD repo into that
workspace. Bryan asked:

1. Is per-workspace cloning necessary, or could a **single SDD clone per
   machine** be shared by all agents (sync on read, push on write)?
2. If we share one clone, can we solve the **concurrent-write problem** where
   several agents mutate SDD files at once?

## TL;DR / recommendation

- **The duplication is real and worth removing.** Each workspace holds its own
  ~472 MiB clone of `sase-org/sdd`, freshly pulled from GitHub over the network
  at prep time. This machine (`athena`) currently carries ~3.3 GiB of duplicated
  SDD clones across active workspaces, and every new workspace re-pays a ~213 MiB
  network transfer.
- **But "one shared clone" conflates two separable problems.** Split them:
  - **Problem A — clone duplication (disk + startup latency).** Easy, low-risk
    win. Solve by **sharing the object store**, not the working tree.
  - **Problem B — concurrent writes.** More nuanced, but *already partly solved
    today at the remote*, and the risky part lives in bead aggregate files, not
    in per-file plans/research/tales.
- **Recommended:** keep a **private working tree + index per workspace**, but
  back them with **one shared object store per machine** (a local mirror that
  workspaces `--reference`/local-hardlink clone from), plus an optional
  **sparse checkout** that omits the 208 MiB of research infographic PNGs. This
  captures nearly all the disk/latency savings *without changing today's
  concurrency model at all*.
- **Do NOT share a single working tree** unless you also add: a machine-wide
  advisory lock around the whole write→commit→push transaction, a push
  retry-with-backoff loop, and single-writer serialization for bead mutations in
  `sase-core`. That is a much larger change and it turns SDD writes into a
  serialized bottleneck. Reserve it for later, and only if the bead-merge pain
  below actually materializes.

## How SDD storage works today (verified)

This repo runs `sdd.storage: separate_repo` (see `sase.yml`). SDD lives in its
own GitHub repo `git@github.com:sase-org/sdd.git`, and it is **cloned per
workspace**:

- The workspace clone lives at `<workspace>/.sase/sdd` and is its own git repo
  (`remote.origin.url = git@github.com:sase-org/sdd.git`).
- It is a **distinct clone** from the primary checkout's `.sase/sdd` (verified by
  differing inodes), and from every other `sase_<N>` workspace's clone. There is
  no symlink, no git `alternates`, no hardlinking — the pack file has link
  count 1.
- The clone is created **fresh from GitHub over the network** (reflog:
  `clone: from github.com:sase-org/sdd.git`), not copied/hardlinked from a local
  mirror.

Code path (verified):

- Physical path selection: `src/sase/sdd/store.py` `_sdd_dir_for_storage`
  (L361-370) — `separate_repo` → `<workspace>/.sase/sdd`, `local`/auto →
  `<primary>/.sase/sdd`, `in_tree` → `<workspace>/sdd`. Reached via
  `resolve_sdd_store` / `resolve_sdd_dir`.
- Per-workspace clone: `ensure_workspace_sdd_clone` in
  `src/sase/sdd/_store_link.py:19-76`. It **prefers a network clone** from the
  companion remote (`_clone_sdd_store` → `git clone <remote_url>`), and only
  **falls back** to a **local clone from the primary**
  (`_clone_sdd_store_from_primary` → `git clone <primary/.sase/sdd>`, then rewrite
  origin). If a matching clone already exists it is reused and just
  pulled/fast-forwarded (`_is_matching_store_clone`, `_sync_workspace_sdd_clone`).
  Invoked from `src/sase/axe/run_agent_runner_setup.py:75`.
- Initial materialization into the primary: GitHub plugin
  `ws_materialize_sdd_store` (`sase-github` `workspace_plugin.py:205-270`),
  `_clone_gh_repo` doing plain `git clone` (SSH→HTTPS), **no
  `--reference`/`--shared`/`--depth`**.
- **No git alternates / `--reference` / `--shared` / mirror / cache exists
  anywhere.** Notably, the **main-repo** workspace clone (`_ensure_git_clone_at`,
  `src/sase/workspace_provider/utils.py:253`) already does a **local `git clone`
  from the primary**, which hardlinks `.git/objects` by default — so the main
  repo is already cheap. **The SDD clone is the lone outlier** doing a
  network-first clone with no object sharing.

### Cost, measured on this machine

| Item | Size |
| --- | --- |
| One SDD clone (`<workspace>/.sase/sdd`) | **472 MiB** |
| — of which `.git` object store | 220 MiB (`size-pack` ~213 MiB, 5,280 objects, 103 commits) |
| — of which working tree | 252 MiB |
| — of which research `*.png` infographics | **208 MiB across 171 files** |
| — "live" SDD text (plans/tales/prompts/beads/epics minus PNGs) | ~44 MiB |
| Duplicated SDD clones across active workspaces (now) | **~3.3 GiB** |
| Network transfer per workspace prep | ~213 MiB |

So ~428 MiB of each 472 MiB clone is avoidable per workspace: 220 MiB is a
shareable object store, and 208 MiB is infographic PNGs an implementing agent
almost never reads.

### SDD content layout (matters for conflicts)

- **Per-file artifacts:** `plans`, `research/`, `tales/`, `prompts/`, `epics/`,
  `legends/`, `myths/`. One file per entity → two agents touching different
  entities never collide textually.
- **Beads are event-sourced:** each bead has its own append-only stream
  `beads/events/streams/sase-<id>.jsonl`, so different beads don't collide.
- **Bead aggregates are the real conflict surface:** `beads/issues.jsonl` (the
  canonical, Rust-owned store) and `beads/events/manifest.json` (a derived
  projection: `"generated_from": "issues.jsonl"`). `beads/beads.db` (SQLite) is
  **gitignored** and rebuilt from JSONL on every mutation, so it is not a git
  conflict surface — but see the shared-working-tree hazard below.

### Commit / push / locking today (verified in code)

- `commit_sdd_files()` in `src/sase/sdd/_commit.py` runs `git add -- <files>`
  then `git commit -m ... -- <files>` with `check=True`. **No fetch, no rebase,
  no `index.lock` wait/retry.** Concurrent index writers get a hard
  `CalledProcessError`/timeout.
- Separate-repo push goes through `src/sase/bead/sync.py`
  (`push_bead_work_launch[_async]`): `git push`, and on failure a **single**
  `git pull --rebase && git push` — no loop, no backoff. The async variant is a
  detached, fire-and-forget `sh -c "git push || (git pull --rebase && git
  push)"`.
- **No machine-wide or file-based lock guards SDD commits or bead git writes.**
  `fcntl.flock` is used elsewhere (memory proposals, ChangeSpec `.gp` files,
  agent-name allocation, TUI xprompt-save) but never around `_commit.py` /
  `bead/sync.py`.
- Bead mutations route through Rust (`sase_core_rs`) via
  `src/sase/core/bead_mutation_facade.py`; `beads.db` is deleted and rebuilt
  from `issues.jsonl` after each mutation (`src/sase/bead/project.py`
  `_refresh_db_from_jsonl`). There is **no bead daemon** — direct per-agent file
  access.
- SDD commits are **event-driven and small**: plan acceptance
  (`src/sase/axe/run_agent_exec_plan_accept.py`) and agent finalize
  (`src/sase/llm_provider/commit_finalizer.py`,
  `chore(sdd): sync uncommitted SDD store changes`). Commits carry
  `SASE_AGENT` / `SASE_MACHINE=athena` trailers.

## Reframing: concurrent writes already happen today

The per-workspace clone does **not** avoid concurrent writes. All ~27 workspace
clones push to the **same** `sase-org/sdd` master. Today's design mediates
concurrency **at the remote**: each agent has a private working tree, commits
locally, then reconciles at push time via `pull --rebase`. GitHub's ref update
is the serialization point.

So the choice is really *where* concurrency is mediated:

- **Today (per-workspace clone):** mediated at the **remote**. Isolation is free
  (each tree is private); the cost is N clones and N network fetches, and rebase
  conflicts surface on `issues.jsonl`/`manifest.json` at push.
- **Single shared clone:** mediated **locally**. One object store, one fetch —
  but now the local working tree, index, and (for beads) the single physical
  store must be serialized by *us*, because git won't do it politely.

This reframing is the key point: **"share a clone" and "solve concurrent
writes" are orthogonal.** You can get the disk/latency win without touching the
concurrency model, and you can harden concurrency independently.

## Why a single shared *working tree* is hazardous with today's code

If multiple agents share one `.sase/sdd` working directory:

1. **`index.lock` collisions.** `commit_sdd_files()` has no wait/retry, so two
   simultaneous commits → one hard-fails. Today's separate trees never contend
   on the same `.git/index.lock`.
2. **`pull --rebase` rewrites a shared tree.** The async push does
   `git pull --rebase` on failure, which rewrites HEAD and the working tree.
   That is safe only because each clone is **private** today. In a shared tree it
   can rewrite files another agent is mid-write on, or move HEAD out from under a
   pending commit.
3. **`beads.db` delete-and-rebuild race.** Each bead mutation deletes
   `beads.db{,-wal,-shm}` and rebuilds from JSONL. Two agents in one directory
   delete each other's DB mid-rebuild.
4. **Partial-write commits.** A blanket `git add .` from agent A can stage agent
   B's half-written file.
5. **Stale locks / blast radius.** A crashed agent can leave `index.lock`
   behind; one bad local state now breaks every agent on the machine instead of
   one workspace.

None of these are unsolvable, but **all require new machine-level
serialization** that does not exist today.

## Options considered

### Option A — Shared object store, private working trees  ★ recommended

Keep per-workspace `.sase/sdd`, but create it as a **local reference/hardlink
clone** from one per-machine SDD mirror instead of a network clone:

- Machine mirror at a stable path, e.g.
  `~/.local/state/sase/sdd-cache/<owner>/<repo>` (bare or normal), keyed by
  remote URL.
- Workspace prep: `git clone --reference <mirror> --dissociate=false` (or a
  same-filesystem `git clone --local` from the mirror, which hardlinks packs),
  then `git fetch` the mirror periodically / on demand.
- **Concurrency model is unchanged** — every workspace still has a private tree,
  index, and refs, and still reconciles at the remote via push-rebase. Nothing
  in `_commit.py` / `bead/sync.py` needs to change.
- **Savings:** eliminates the ~213 MiB network fetch per prep and nearly all of
  the 220 MiB `.git` duplication (objects shared/hardlinked).
- **Caveat:** `--reference`/`--shared` create the classic alternates footgun —
  never `git gc --prune` the mirror while dependent clones reference it. A
  same-FS `--local` hardlink clone is safer under repack (each clone keeps its
  own pack dir; hardlinks break cleanly on repack) and needs no alternates
  discipline. Prefer the hardlink-clone variant.
- **Most of this already exists.** The main-repo clone already does exactly this
  kind of local hardlink clone (`utils.py:253`), and SDD already has a
  local-clone-from-primary path (`_clone_sdd_store_from_primary` in
  `_store_link.py`) — today it is only a *fallback*. The smallest viable change
  is to **prefer the local (hardlinked) clone**, sourced from a per-machine
  mirror (or the primary), and demote the network clone to the
  no-local-source case in `ensure_workspace_sdd_clone` (`_store_link.py:19-76`).

### Option A+ — …plus sparse checkout

Layer a sparse checkout that omits `research/**/*.png` (and other rarely-read
heavy assets). Combined with Option A, per-workspace SDD footprint drops from
**472 MiB → well under ~50 MiB** (shared objects + no infographics), with no
concurrency change. Agents that genuinely need an infographic can materialize it
on demand.

### Option B — Single shared working tree + machine lock  (heavier)

One `.sase/sdd` per machine, all agents write into it. To be safe:

- A machine-wide advisory `flock` (extend the existing `fcntl.flock` helpers)
  around the **entire** transaction: write files → `git add` → `git commit` →
  `git pull --rebase` → `git push` → release. Add lock timeout + stale-lock
  recovery.
- Move bead mutation serialization into `sase-core` (single writer; see boundary
  note) so `issues.jsonl` and the `beads.db` rebuild are never concurrent.
- Harden push into a **retry-with-backoff loop** (today it is single-shot).

Upside: **one** `issues.jsonl`, so bead aggregate merges disappear entirely (the
one place where separate clones actually hurt). Max disk savings. Downside: SDD
writes become serialized machine-wide (a bottleneck under fan-out), larger blast
radius, and real recovery/robustness work. Not recommended for v1.

### Option C — SDD writer daemon  (cleanest single-writer, most infra)

A per-machine service owns the one clone; agents enqueue "apply these files +
commit message" / bead mutations over a socket; the daemon serializes, batches,
and debounces pushes. This is the theoretically cleanest concurrency story
(single writer, WAL-safe SQLite, natural retry/backoff home) but the most new
infrastructure and a new failure mode (daemon down). Consider only if Option
A(+) proves insufficient and write volume grows a lot.

### Option D — `git worktree` off a shared repo  (poor fit)

Shares objects, but git forbids the same branch (`master`) being checked out in
two worktrees, and SDD's flow is "everyone commits master." You'd need
detached-HEAD or per-agent branches plus a merge step — more complexity than
Option A for the same object-sharing benefit. Rejected.

### Tradeoff summary

| Option | Disk/machine | Prep latency | Concurrency change | Risk | Effort |
| --- | --- | --- | --- | --- | --- |
| Today (per-workspace network clone) | ~N × 472 MiB | high (net fetch) | none needed | low | — |
| **A: shared objects, private trees** | ~1 × 220 MiB + N × tree | **low (local)** | **none** | low | small |
| **A+: A + sparse checkout** | ~1 × 220 MiB + N × ~small | low | none | low | small–med |
| B: single shared tree + lock | ~1 × 472 MiB | lowest | **large** (new locks) | med–high | large |
| C: writer daemon | ~1 × 472 MiB | lowest | large (new service) | med | large |

## Solving the concurrent-write problem (independent of the clone decision)

Whichever option you pick, these harden concurrency and are worth doing anyway:

1. **Keep the layout conflict-free.** Per-file plans/research/tales and per-bead
   event streams already mean most concurrent writes touch disjoint files.
   Preserve that; avoid new single aggregate files.
2. **Make bead aggregates regenerable and merge-tolerant.** `manifest.json` is
   already derived (`generated_from: issues.jsonl`) and `beads.db` is rebuilt —
   so on a rebase conflict they can be **regenerated** rather than hand-merged.
   The one file that still needs a real merge/single-writer strategy is
   `issues.jsonl`. Because it is Rust-owned, that logic belongs in `sase-core`
   (append-ordered writes or an in-process/file lock at the mutation seam), not
   in per-frontend Python.
3. **Replace the single-shot push with a bounded retry loop** (`fetch` →
   `rebase` → `push`, a few attempts with jittered backoff) in
   `bead/sync.py`. This helps *today* too, since 27 clones already race the
   remote.
4. **If you ever share a working tree, wrap the whole transaction in one
   machine-level `flock`** — never just the commit or just the push.

## Rust core boundary

Per this repo's boundary rule, SDD storage policy, concurrency serialization,
and bead-store mutation are **backend/domain** behavior — a CLI, editor, or web
frontend would need identical semantics. So:

- The per-machine **mirror/cache resolution + local clone/fetch** and any
  **write serialization** should live in / be driven by `../sase-core`
  (`sase_core_rs`), with Python as a thin adapter.
- `issues.jsonl` mutation is already Rust-owned — the right home for a
  single-writer or file-lock guarantee is there.

## Recommended plan (phased)

1. **Phase 1 (high value, low risk):** Add a per-machine SDD **mirror/cache** and
   make `ensure_workspace_sdd_clone` (`src/sase/sdd/_store_link.py:19-76`)
   **prefer a local hardlink clone from it** (reusing the existing
   `_clone_sdd_store_from_primary` machinery) instead of the network-first
   `_clone_sdd_store`; refresh the mirror via `fetch`. No concurrency change.
   Big disk + startup win. This mirrors what the main-repo clone already does at
   `src/sase/workspace_provider/utils.py:253`.
2. **Phase 2:** Add **sparse checkout** to omit `research/**/*.png` (and similar
   heavy assets) from workspace SDD trees, with on-demand materialization.
3. **Phase 3 (hardening, useful regardless):** Turn the single-shot push into a
   **fetch→rebase→push retry-with-backoff** loop in `bead/sync.py`, and move the
   `issues.jsonl` single-writer/lock guarantee into `sase-core`.
4. **Phase 4 (only if needed):** Evaluate a truly single shared working tree
   (Option B) or a writer daemon (Option C) *only* if bead-aggregate rebase pain
   or disk pressure remains after Phases 1–3.

## Open questions

- Mirror location + key (per remote URL vs per project), and its `gc`/`fetch`
  refresh policy without breaking alternates.
- Whether to keep `--reference` (needs gc discipline) or same-FS `--local`
  hardlink clone (safer). Recommend hardlink.
- Whether infographic PNGs should live in the SDD repo at all, or move to
  Git-LFS / an out-of-band asset store — that alone would shrink `.git` and every
  clone dramatically.
- Concrete `issues.jsonl` concurrency strategy in `sase-core`: append-ordered
  writes, per-store advisory lock, or a small write queue.

## Answer, as SASE variables

Set on this research agent (see "Answer variables" at run end):

- `shared_clone_recommended = yes_objects_only` — share the object store per
  machine, keep private working trees; do **not** share a single working tree in
  v1.
- `concurrency_solvable = yes` — but keep working trees private (remote-mediated,
  as today) or, if a shared tree is later required, add a machine `flock`
  transaction + push retry loop + `sase-core` single-writer for `issues.jsonl`.
- `recommended_option = A_plus_sparse` — shared object store + sparse checkout
  omitting research PNGs.
