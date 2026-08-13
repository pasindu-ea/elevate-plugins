---
name: release-reconcile
description: Reconcile documentation against all changes since the last release tag, or bootstrap a first README when the repo has no documentation surface yet. Opens one PR labelled for human approval. Usage: /release-reconcile [since-tag]
argument-hint: [since-tag]
---

You reconcile documentation after a release. The executor has checked out the
default branch. `gh` and `git` are authenticated via GITHUB-TOKEN.

## 0. Pick a mode
Run `ls README.md Docs 2>/dev/null` and `git tag --list`.

- **RECONCILE mode** — a README or Docs/ already exists. Go to §1.
- **BOOTSTRAP mode** — no README, no Docs/, or the doc surface is a stub
  with no real prose. There is nothing to reconcile against; go to §1b
  instead. Do not force RECONCILE mode's "fix only what changed" logic
  onto a repo that has nothing to begin with — that produces an empty,
  useless run.

## 1. Establish scope (RECONCILE mode)
- `git fetch --tags --force`
- SINCE = $1 if given, else `git describe --tags --abbrev=0`
- If no tag exists at all, fall back to the repository's root commit
  (`git rev-list --max-parents=0 HEAD`) and note in the PR body that this
  ran against the full history, not a release window.
- Diff range is `SINCE..HEAD`. If the range is empty, post nothing and stop.
- `git log --oneline SINCE..HEAD` and `git diff --stat SINCE..HEAD` to see the shape.

## 2. Extract changed symbols (RECONCILE mode)
From `git diff SINCE..HEAD` collect only names a human would type in a document:
config keys, environment variables, CLI flags, HTTP routes, public
class/method signatures, file paths referenced in docs, DB columns.
For each, record: name, kind, and change (added | renamed | removed | signature-changed).
IGNORE private members, local variables, test fixtures, and lockfiles.

## 3. Build the reverse index (RECONCILE mode)
For every symbol, search the doc surface (README.md, Docs/, .env.example, any
docs site directory) for mentions. Produce a table of
symbol -> change -> the exact file:line hits.
A hit means that doc makes a claim about something this release changed.
A symbol that is `added` with zero hits is an UNDOCUMENTED FEATURE — record it.

## 4. Fix what you can prove (RECONCILE mode)
Edit only the lines in your reverse index. Rules:
- Renames: update the name, keep the surrounding prose intact.
- Removals: delete the entry, and any now-dangling reference to it.
- Additions: add an entry that matches the file's existing format exactly.
- Never rewrite a section wholesale. Never "improve" prose you were not sent to fix.

## 1b. Establish scope (BOOTSTRAP mode)
- Read every source file to understand what this repo actually does — its
  entry point, its public classes/methods, and every environment variable
  it reads (e.g. `Environment.GetEnvironmentVariable("X")` in .NET).
- Read `.env.example` if present — its keys are the required config surface,
  even though listing a key there does not count as "documented" (see §5b).
- There is no release window here — the diff range is "the whole repo,
  as it stands today."

## 2b. Write the first README (BOOTSTRAP mode)
Write `README.md` covering, in this order: what the agent does in one
paragraph; how to configure it (every required env var, one line each,
in the same order `.env.example` lists them); how to run it; the public
surface a consumer would actually touch (class + method signatures, not
every private helper). Match the plainest style already present in the
repo's own code comments — do not invent a house style.

## 3b. What you could not document (BOOTSTRAP mode)
List anything you deliberately left out and why: behavior you could not
confirm without running the code, a design decision that looks arbitrary
and might not be, TODOs already present in the source. Same rule as
RECONCILE mode's doc debt — a "fully documented" claim you are not sure
of is a failed run.

## 5. Record doc debt / doc gaps — this is mandatory
RECONCILE mode: anything you could NOT fix goes in a "Doc debt" list with
a one-line reason. BOOTSTRAP mode: use §3b's list instead. Refuse rather
than guess when: it needs a diagram or screenshot; the intended behaviour
is genuinely ambiguous from the code; it needs a product decision. A run
that claims 100% fixed/documented is a failed run. Honesty here is the
deliverable.

## 5b. Metric (BOOTSTRAP mode only)
Run `Scripts/doc-coverage.sh` if present in this repo (before and after
your edit, i.e. against HEAD then against your working tree) to report
`Public surface: N`, `Documented: before -> after`, `COVERAGE: before% -> after%`.
If the script isn't present, report the same shape by hand: count the
public classes/methods/required env vars you found in §1b, and how many
your new README now mentions.

## 6. Open ONE gated pull request
- Branch: `docs/reconcile-SINCE-to-HEAD` (RECONCILE) or `docs/bootstrap-readme` (BOOTSTRAP)
- `gh pr create` against the default branch
- Apply the label `ai-dlc/docs/pending-review`
- PR body sections, in order:
  ### Fixed (n) / Documented (n)   — checklist, each with file:line or symbol name
  ### Doc debt (n) / Doc gaps (n)  — each with its reason
  ### Release notes draft          — RECONCILE mode only, grouped Breaking / Added / Fixed
  ### Metric                       — RECONCILE: Doc Debt before -> after (Scripts/doc-debt.sh).
                                       BOOTSTRAP: Coverage before -> after (Scripts/doc-coverage.sh, see §5b)

## Hard constraints
- You must NOT merge, and must NOT push to the default branch. A human approves.
- Exactly one PR per invocation. If the branch already exists, update it instead.
- If you fixed/documented nothing and found no debt/gaps, post no PR. Silence is a valid result.
