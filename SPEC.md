<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright 2026 hyperpolymath -->

# self-destructing-git-garbage — marker & reap specification

This is the contract. The reaper obeys it exactly; nothing else dies.

## 1. The governing principle (opt-in + provably-merged)

A branch is **reapable** only when it satisfies **at least one** of:

- **(M) Marked & expired** — its name carries a self-destruct marker whose
  expiry date is `<= today`.
- **(P) Provably merged** — its work is already safe: it is an ancestor of its
  repo's default branch, **or** its upstream tracking branch is `gone`
  (the remote branch was deleted, i.e. the PR merged).

> An **unmarked, unmerged** branch is **never** reaped — no matter how old it is.
> Age/access-time alone is never a deletion reason. (Estate rule:
> *no deletion by access-recency*.)

Two further hard guards, always:

- The repo's **default branch** (`main`/`master`) is never reaped.
- A **currently checked-out** branch is never reaped.
- Anything matching the repo's **keep-list** (see §4) is never reaped.

## 2. The marker grammar (branch-name convention)

Markers live in the branch name so they work on any forge, are visible in
`git branch`, and need zero per-repo infrastructure.

| Form | Meaning | Expires |
|---|---|---|
| `ttl/<YYYY-MM-DD>/<rest>` | time-boxed branch | on/after `<YYYY-MM-DD>` |
| `<anything>--expires-<YYYY-MM-DD>` | inline expiry suffix | on/after `<YYYY-MM-DD>` |
| `garbage/<rest>` | explicit throwaway | **immediately** (author discarded it) |
| `tmp/<rest>` , `scratch/<rest>` | ephemeral, reap **only once merged** | n/a (needs (P)) |

Rules:

- Dates are ISO `YYYY-MM-DD`, interpreted in UTC.
- `garbage/*` is the one marker that expires with no date — it is a deliberate
  "throw this away" signal. It still gets a backup before deletion.
- `tmp/*` and `scratch/*` are **not** time-markers: they only become reapable
  via the provably-merged path (P). They exist so the reaper can prioritise
  obvious scratch space without ever deleting unmerged scratch work.
- A branch may be both marked and merged; either path suffices.

## 3. What happens to a reaped branch (never just `delete`)

For every branch it reaps, the reaper first records recovery, then deletes:

1. Append `repo, branch, sha, restore_cmd` to a dated **manifest**.
2. Write a `git bundle` of the branch tip into `backups/`.
3. `git branch -D` (local) or `git push origin --delete` (remote).

Restore is therefore always one command. The local tip also survives in the
reflog for the gc window.

## 4. Per-repo keep-list (optional, belt-and-braces)

Drop a `.git-garbage.toml` at a repo root to pin patterns that must never be
touched even if they look reapable:

```toml
# .git-garbage.toml
keep = ["release/*", "local-rebase-*", "wip/*"]
# optional: extra kill patterns to treat as garbage/* (expire-on-merge only)
kill = []
```

The keep-list is honoured under **both** the (M) and (P) paths. There is no
config that can make an unmarked, unmerged branch reapable — that is a property
of the spec, not of configuration.

## 5. Default vs opt-out (explicitly not implemented)

This tool ships **opt-in + provably-merged** only. There is deliberately no
"expire every unmarked branch after N days" mode: that would delete by
staleness alone, which the estate forbids. If you ever want that, it must be a
separate, clearly-named tool — not a flag here — so the safe default can never
be flipped by accident.
