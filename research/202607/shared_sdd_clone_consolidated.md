---
create_time: 2026-07-09
status: research
---

# Shared SDD Clone Consolidated Research

## Question

SASE currently prepares each agent workspace with its own `.sase/sdd` companion
repo clone. Could each machine instead keep one local SDD clone that all agents
share, syncing and pushing when needed? If so, can SASE solve concurrent writes
when multiple agents mutate SDD files at the same time?

## Short Answer

Use a shared **object cache**, not a shared mutable working tree.

The current per-workspace SDD clone is wasteful and worth optimizing. On this
machine, this workspace's SDD clone is 473 MiB; its `.git` directory is 221 MiB
with a 212.91 MiB pack; `research/**/*.png` accounts for about 204.9 MiB across
169 files; and 7 local SASE workspaces currently hold SDD clones totaling about
3.21 GiB.

But replacing those clones with one writable checkout is not a safe default.
Current SDD writes are ordinary file writes plus Git commands with no
machine-wide SDD lock, no `index.lock` wait/retry in the SDD commit path, and a
single-shot push fallback. A shared checkout would turn those gaps into local
write races.

Recommended direction:

1. Keep a private SDD working tree and Git index per workspace.
2. Back those private trees with a machine-local SDD mirror/cache, preferably by
   local hardlink clone from the mirror or primary checkout.
3. Add sparse checkout or an equivalent materialization rule that omits heavy
   rarely-read assets such as `research/**/*.png`.
4. Harden remote push/rebase retry and bead mutation serialization separately.
5. Consider a single shared working tree only as an explicit later mode with a
   transaction lock or writer broker.

## Verified Current Behavior

This repo configures `sdd.storage: separate_repo` in `sase.yml`; the default
config remains `auto` in `src/sase/default_config.yml`.

For separate-repo storage, `src/sase/sdd/store.py` resolves the effective SDD
directory to `<workspace>/.sase/sdd`:

- `resolve_sdd_dir()` documents the separate-repo working tree as
  workspace-local.
- `_sdd_dir_for_storage()` returns `<workspace>/.sase/sdd` for
  `SDD_STORAGE_SEPARATE_REPO`.
- `materialize_sdd_store()` calls `ensure_workspace_sdd_clone()` when a
  separate-repo record exists.

Per-workspace clone setup lives in `src/sase/sdd/_store_link.py`:

- `ensure_workspace_sdd_clone()` targets `<workspace>/.sase/sdd`.
- It reuses a matching existing clone and refreshes it with `git pull --rebase`.
- For a missing clone, it prefers `git clone <remote_url> <workspace_sdd>`.
- Only if that remote clone fails does it fall back to cloning from the
  primary checkout's `.sase/sdd`.
- Failures are logged and do not block launch.

Workspace checkout setup in `src/sase/workspace_provider/utils.py` already uses
`git clone <primary_workspace_dir> <target_checkout_dir>` for the main repo,
then resets `origin` and fetches. On a normal same-filesystem local clone, Git
can hardlink object files. The main repo therefore already gets the cheap local
clone path; the SDD clone is the outlier because it is network-first.

The GitHub provider confirms the separate-repo policy and primary materializer:
`sase-github` declares `sdd_storage_policy="separate_repo"` and materializes the
primary `.sase/sdd` with plain `git clone` via `_clone_gh_repo()`. Grepping the
SASE repo and linked GitHub plugin found no use of `--reference`, `--shared`,
`--dissociate`, `alternates`, `--mirror`, or `--depth` for SDD clone sharing.

## Verified Commit, Push, And Locking Behavior

`src/sase/sdd/_commit.py` commits local SDD changes by:

1. Listing changed files with `git ls-files`.
2. Running `git add -- <changed-files>`.
3. Running `git diff --cached --quiet`.
4. Running `git commit -m ... -- <changed-files>`.

That path has no fetch, no rebase, and no wait/retry for Git `index.lock`.
`commit_sdd_store_files()` then pushes only after a successful local commit.
Push failure is deliberately non-fatal so the local commit remains reachable.

Separate-repo pushes are delegated to `src/sase/bead/sync.py`:

- Synchronous push is `git push`; on failure, one `git pull --rebase`; then one
  retrying `git push`.
- Async push starts a detached `sh -c "git push || (git pull --rebase && git
  push)"`.
- There is no retry loop, backoff, or SDD-wide serialization.

The repo has several `fcntl.flock` users, but they guard other subsystems
(memory proposal ledgers, ChangeSpec files, agent-name allocation, artifact
metadata, etc.). They do not wrap `commit_sdd_files()`,
`commit_sdd_store_files()`, or `push_bead_work_launch[_async]()`.

## Bead-Specific Findings

Bead mutations route through the Rust binding facade. In the linked
`sase-core` repo, `crates/sase_core/src/bead/mutation.rs` loads the store,
mutates an in-memory `MutableStore`, then saves event streams, config, and
`issues.jsonl`. `crates/sase_core/src/bead/jsonl.rs` writes files atomically by
creating a deterministic `*.tmp` sibling and renaming it into place.

That is storage-safe against torn files, but it is not a store-wide
read-modify-write lock. A search of `sase-core` found `fs2` locks in other
stores, not in the bead mutation path.

Important details:

- Bead events are stored as per-bead stream files under
  `beads/events/streams/*.jsonl`, which reduces conflicts when agents mutate
  different beads in private clones.
- `beads/events/manifest.json` is generated from the stream set, and
  `beads/issues.jsonl` is regenerated as a projection.
- `beads/beads.db` is a gitignored compatibility SQLite mirror in the Python
  layer; it is deleted and rebuilt from JSONL after export, so it is not a Git
  conflict surface but would race badly in one shared working directory.
- Atomic rename does not prevent lost updates when two processes load the same
  old state, compute different new states, and then each writes a complete new
  projection.

Therefore, bead aggregate/projection handling is the main content-level
concurrency surface. Any single-writer guarantee for `issues.jsonl` or event
stream read-modify-write belongs in `sase-core`, with Python and other
frontends calling through a thin adapter.

## Reframing The Concurrency Question

Per-workspace clones do not eliminate concurrency; they move it to the remote.
Today, agents commit in private local SDD clones and race only when pushing to
the shared `sase-org/sdd` branch. GitHub's ref update plus `pull --rebase`
mediates that race.

A single shared working tree moves the race onto the local machine. Then every
agent shares one working tree, one Git index, one HEAD/rebase state, one async
push process namespace, and one bead mirror directory. Git lock files prevent
some low-level corruption, but the current SDD code does not queue, retry, or
attribute those collisions.

This means clone duplication and concurrent-write correctness are separable:
SASE can get most of the disk/startup win while leaving the current remote-
mediated concurrency model intact.

## What Breaks With One Shared Writable Checkout

If all agents write into one `.sase/sdd` checkout today:

- Two `git add`/`git commit` operations contend on one `.git/index`; one may
  fail with `index.lock` or timeout.
- A background async push can run `git pull --rebase` and rewrite the shared
  working tree while another agent is writing.
- One agent's commit can stage another agent's partially written file if the
  pathspec is broad or the changed-file list is collected after both writes.
- Two agents writing the same research, plan, Q&A, or prompt snapshot path can
  overwrite before Git sees a merge conflict.
- Bead mutation can lose updates because mutation is load/modify/save without a
  store-wide lock.
- The `beads.db` delete-and-rebuild mirror can be removed by one process while
  another is using or rebuilding it.
- A stale Git lock or interrupted rebase affects every local agent instead of
  only one workspace.

These are solvable, but not by just changing `SASE_SDD_DIR` to a shared path.

## Options

### Option A: Shared Object Cache, Private Working Trees

Recommended.

Keep `<workspace>/.sase/sdd` as the agent's writable directory, but materialize
it from a machine-local SDD cache/mirror rather than from GitHub:

- Use a stable cache key derived from provider + host + repo or remote URL.
- Refresh the mirror/cache under a short cache lock.
- Create workspace clones by local hardlink clone from the mirror or primary
  SDD checkout, then set `origin` to the real remote.
- Demote network clone to the fallback path.
- Keep per-workspace commits and push/rebase behavior.

This preserves local working-tree and index isolation, avoids the repeated
network fetch, shares or hardlinks Git objects, and requires much less behavior
change than a shared writable checkout.

Prefer same-filesystem local/hardlink clones over Git alternates if practical.
Alternates/reference clones save space but require disciplined mirror garbage
collection; local hardlinks avoid the classic "mirror pruned an object my clone
needs" failure mode.

The implementation is naturally close to existing code:

- The main workspace clone already uses local clone from the primary checkout.
- `src/sase/sdd/_store_link.py` already has a local clone from primary helper;
  it is just behind the network clone today.

### Option A+: Option A Plus Sparse Checkout

Also recommended.

Omit `research/**/*.png` and other rarely-needed heavy assets from ordinary
agent SDD checkouts, with on-demand materialization for agents that need those
files. On this machine, that alone avoids about 205 MiB of checked-out PNGs per
workspace, in addition to sharing the 213 MiB packed object payload.

### Option B: Single Shared Working Tree With A Transaction Lock

Viable but not recommended for v1.

A correct shared-checkout design needs one SASE-owned transaction around the
entire mutation:

1. Acquire an exclusive cross-process lock keyed by SDD remote/store identity,
   outside the checkout.
2. Verify the checkout is clean and not mid-merge/rebase/cherry-pick.
3. Fetch and rebase or fast-forward.
4. Run the declared SDD mutation.
5. Stage only the mutation's declared paths.
6. Commit with the correct runtime tags and artifact marker.
7. Push while still holding the lock, with bounded retry/backoff.
8. On conflict, leave a reachable recovery branch or stash and report a clear
   recovery path.
9. Release the lock.

Async push must either become synchronous inside the transaction or be owned by
the same serialized broker. Direct human/agent shell edits to the shared
checkout would need to be prohibited, detected as dirty state, or converted
into brokered operations.

### Option C: SDD Writer Broker/Daemon

This is the cleanest single-writer model but the most infrastructure. A
per-machine service owns one checkout; agents submit "apply these paths with
this commit message" or bead mutations; the broker serializes writes, pushes,
and retries. Consider it only if Option A+ still leaves unacceptable conflicts
or write volume grows enough to justify a new local service.

### Option D: Git Worktrees

Git worktrees share object storage, but Git does not allow the same branch to
be checked out in many worktrees. SASE would need detached HEADs or per-agent
branches plus explicit push/merge logic. That is workable, but it is not
clearly better than local hardlink/reference clones for the same object-sharing
win.

## Recommendation

Do not make a single shared mutable SDD checkout the default.

Implement this phased plan instead:

1. Add a per-machine SDD mirror/cache keyed by the companion remote URL.
2. Change workspace prep so `ensure_workspace_sdd_clone()` prefers a local
   hardlink clone from that cache or from the primary `.sase/sdd`; keep network
   clone as fallback.
3. Add sparse checkout/materialization rules for heavy research PNGs.
4. Improve `bead/sync.py` push behavior from one-shot push/rebase/push to a
   bounded fetch/rebase/push retry loop with jittered backoff.
5. Put bead read-modify-write serialization or merge-aware projection handling
   in `sase-core`, where the bead mutation logic already lives.
6. Revisit a single shared working tree only if disk pressure or bead aggregate
   conflict pain remains after those changes.

This keeps the high-value performance and disk savings while preserving the
most important property of the current design: each agent owns its working tree
and its local Git state.

## Final SASE Variables

Set these variables for downstream consumers:

- `recommendation=shared_object_cache_private_worktrees`
- `shared_working_tree_recommended=no`
- `concurrency_solvable=yes_with_transaction_lock_or_broker`
- `recommended_phase=A_plus_sparse`
- `final_research_path=.sase/sdd/research/202607/shared_sdd_clone_consolidated.md`
- `confidence=high`
