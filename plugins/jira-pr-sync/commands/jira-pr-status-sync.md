---
name: jira-pr-status-sync
description: Move every Jira issue referenced in this PR's commit messages from one status to another, but only if each issue is currently in the expected starting status. Usage - /jira-pr-status-sync "<from-status>" "<to-status>"
argument-hint: "<from-status>" "<to-status>"
---

You move Jira issues along the board based on what actually happened to a
GitHub pull request. There is no ambiguity to resolve here — the commit
message convention tells you exactly which issue(s) this PR is about:

```
SCRUM-1: adding team member's work
```

The Jira ID is the token before the first colon on the first line of the
commit message.

## 0. Read arguments and credentials
$1 = the expected **current** status (e.g. `"In Progress"`)
$2 = the status to move matching issues **to** (e.g. `"In Review"`)
If either is missing, stop and say so — do not guess a default.

```bash
JIRA_SITE="$(printenv 'JIRA-SITE-URL')"
JIRA_EMAIL="$(printenv 'JIRA-EMAIL')"
JIRA_TOKEN="$(printenv 'JIRA-API-TOKEN')"
AUTH=(-u "${JIRA_EMAIL}:${JIRA_TOKEN}")
```
All Jira requests use `"${AUTH[@]}"` against `${JIRA_SITE}/rest/...`.

## 1. Find every Jira ID this PR actually touches
The PR number is stated in the prompt you were invoked with (e.g. "for PR
#7") — use that number, don't search for one. Get every commit on that PR
(not just the PR title — the team's convention is a commit-message prefix,
and a PR can carry several commits):
```bash
gh pr view <pr-number> --json commits --jq '.commits[].messageHeadline'
```
For each commit's first line, extract a Jira ID **only if it's the exact
prefix before a colon**: pattern `^[A-Z][A-Z0-9]*-[0-9]+:`. A ticket
mentioned mid-sentence, or without the trailing colon, does not count —
this convention is intentionally strict so you never act on a false match.

Collect the **unique** set of IDs found. If none, report that plainly and
stop — do not fall back to scanning the PR title or body.

## 2. For each Jira ID, check before you act
```bash
curl -s "${AUTH[@]}" "${JIRA_SITE}/rest/api/3/issue/${ID}?fields=status,summary"
```
- If the issue doesn't exist / 404s → report it and move on; don't guess a fix.
- If `fields.status.name` does **not** exactly match `$1` (the expected
  "from" status) → **skip it**. This is the most important rule in this
  command: never move an issue whose current status you weren't told to
  expect. If someone already moved SCRUM-1 to "In Testing" by hand, a late
  PR-opened event must not drag it backward to "In Review".
- If it matches → proceed to step 3.

## 3. Find the right transition and apply it
Jira transition IDs are workflow-specific, not fixed strings — you must
look them up per issue, per run:
```bash
curl -s "${AUTH[@]}" "${JIRA_SITE}/rest/api/3/issue/${ID}/transitions"
```
Find the entry whose `to.name` exactly matches `$2`. If none exists (the
workflow doesn't allow that move from the current status), report that
specifically — don't force it via a field edit instead of a transition.

```bash
curl -s "${AUTH[@]}" -X POST -H "Content-Type: application/json" \
  "${JIRA_SITE}/rest/api/3/issue/${ID}/transitions" \
  -d "{\"transition\": {\"id\": \"${TRANSITION_ID}\"}}"
```

## 4. Report back
Your final output (not a Jira comment) must state, plainly, for every
unique Jira ID found in this PR's commits:
- Moved: `SCRUM-1: "In Progress" -> "In Review"`
- Skipped, wrong starting status: `SCRUM-2: expected "In Progress", was "Done"`
- Skipped, no such transition available: `SCRUM-3: no transition to "In Review" from "In Progress" in this workflow`
- Skipped, issue not found: `SCRUM-4: 404`

## Hard constraints
- Never transition an issue whose current status isn't exactly the `$1`
  you were given — this is the one rule that must never be relaxed, even
  if the "obvious" intent seems clear.
- Only read Jira IDs from commit messages, never from the PR title/body.
- One transition attempt per issue per run — if it fails, report why;
  do not retry with a different transition guessed to "probably work."
- If Jira credentials are missing or a request fails, report the exact
  failing call and HTTP status.
