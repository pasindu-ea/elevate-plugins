# release-reconciler

Release-scoped documentation reconciler for [Xianix](https://xianix-team.github.io/documentation/agent-development/overview/).

Where `doc-writer` (official marketplace) syncs docs against a single pull
request, `release-reconciler` works backwards from **everything shipped
since the last release tag**: it extracts the symbols a release changed
(config keys, env vars, routes, public signatures), greps the doc surface
for every mention of them, fixes what it can prove, and opens one PR gated
behind a human-applied label.

Two modes, auto-detected:

- **Reconcile** — repo already has a README/Docs. Fixes only what the
  release actually invalidated; reports the rest as doc debt.
- **Bootstrap** — repo has no doc surface at all (no README, no Docs/, or
  a stub with no real prose). Writes the first README from the public
  surface it can find in the code, and reports coverage (documented
  symbols / total public surface) instead of a debt count.

## Usage

```
/release-reconcile [since-tag]
```

Defaults to the most recent tag reachable from `HEAD` if `since-tag` is
omitted; falls back to the repo's root commit if no tag exists at all.

## What it will not do

- Merge its own PR, or push to the default branch.
- Rewrite prose it wasn't sent to fix.
- Claim something is fixed when it isn't sure — those go in the PR's
  "Doc debt" section with a reason instead.

## Wiring

See `Docs/release-docs-reconciler.md` in the `the-agent` repo for the
two-rule cron + approval-gate setup this plugin is designed to run under:
one scheduled rule that opens the PR, and one webhook rule — matching the
`ai-dlc/docs/approved` label — that merges and publishes it.
