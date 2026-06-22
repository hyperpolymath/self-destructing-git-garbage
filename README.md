<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->
<!-- Copyright 2026 hyperpolymath -->

# self-destructing-git-garbage

Branches that clean themselves up — attachable to any repo, by convention, with
no per-repo setup. Built after an estate-wide sweep turned up ~2000 stale
agent/PR branches scattered across ~250 repos.

## The one rule

A branch is reaped **only** if it is either:

- **marked & expired** — its name carries a self-destruct marker that's past due, or
- **provably merged** — its work is already in the default branch (ancestor) or
  its remote branch was deleted on merge (`upstream: gone`).

An **unmarked, unmerged** branch is **never** touched — however old. Age alone is
never a reason. (This deliberately implements only *opt-in + provably-merged*;
there is no "expire everything stale" mode — see [SPEC.md](SPEC.md) §5.)

## Mark a branch to self-destruct

```sh
bin/git-garbage-mark my-branch --in 14          # ttl/<today+14>/my-branch
bin/git-garbage-mark my-branch --ttl 2026-07-01 # explicit expiry
bin/git-garbage-mark my-branch --garbage        # throwaway, expires now
```

Markers are just branch-name prefixes/suffixes (`ttl/<date>/…`,
`…--expires-<date>`, `garbage/…`), so they work on any forge and show up in
`git branch`. Full grammar in [SPEC.md](SPEC.md) §2.

## Reap (dry-run first, always)

```sh
bin/git-garbage-reap local  ~/developer/repos          # list what WOULD die
bin/git-garbage-reap local  --apply ~/developer/repos  # reap, with backups
bin/git-garbage-reap remote ~/developer/repos          # merged remote branches (owner namespace only)
bin/git-garbage-reap restore .git-garbage-state/reaped-local-<stamp>.tsv
```

Every reap writes a `git bundle` of the tip **and** a restore manifest before
deleting, so it's a one-command undo. Remote mode only acts on origins matching
`$OWNER_GLOB` (default `hyperpolymath`) and skips any branch backing an open PR —
third-party/forked remotes are never touched.

## Run it on a schedule

Owned-compute systemd timer (no metered GitHub Actions) — see
[cron/README.adoc](cron/README.adoc). Ships in dry-run; you flip it to `--apply`
once the manifests look right.

## Safety guarantees

- Default branch and the checked-out branch are never reaped.
- A per-repo `.git-garbage.toml` keep-list pins anything you want immortal
  (honoured under both reap paths) — see [examples/](examples/.git-garbage.toml).
- No deletion by access/modification time. No opt-out-by-staleness mode exists.
- Every deletion is bundled + manifested + reversible.

## Status / licensing

Seed implementation. Licence intent per estate policy: **MPL-2.0** (code) /
**CC-BY-SA-4.0** (docs) — SPDX tags are in place; the authoritative REUSE/licence
pass is owner-manual and not done here.
