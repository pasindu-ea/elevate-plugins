# jira-sprint-nudge

Checks every issue in the current sprint and rates its description for
actual clarity — not just "is there text here." A description can be
present and still fail: too short, too vague to act on, or internally
contradictory (e.g. says two different things about the same requirement).
Issues that score 3/5 or below get a Jira comment that `@`-mentions the
reporter, states the score, and lists specific, issue-specific suggestions
for what to add — which triggers Jira's own notification to them.

No code repository is involved — this plugin talks to Jira's REST API
directly via `curl` + `jq`, the same way the built-in Azure DevOps
work-item rules in this project talk to work items without cloning a repo.

## The rubric

| Score | Meaning |
|---|---|
| 5 | Clear what to do, why, and what "done" looks like |
| 4 | Clear, but missing something minor (e.g. acceptance criteria) |
| 3 | Understandable, but you'd need to ask a clarifying question first |
| 2 | Vague, generic, or just restates the title |
| 1 | Empty, a placeholder, or self-contradictory |

Issues scoring 1–3 get nudged. 4–5 are left alone.

## It re-checks on every edit, not just once

Every comment this plugin posts carries a fingerprint of the description
text it reacted to. On the next run, each issue falls into one of three
buckets:

| Description since last comment | Result |
|---|---|
| Unchanged | Silent — no repeat comments |
| Changed, still scores ≤3 | **Updated nudge** — new score, fresh suggestions against the current text |
| Changed, now scores ≥4 | **Improvement acknowledgment** — a short "thanks, this is clear now" |

So a reporter who ignores the nudge sees nothing new; one who edits and
still falls short gets updated feedback, not a repeat of the same
comment; one who fixes it gets a closing note instead of silence.

## Usage

```
/jira-sprint-nudge
```

No arguments — it always checks whichever sprint on the configured
project's board is currently active.

## What it needs

Four environment variables, read via `printenv` (hyphenated names, not
valid as bash `$VAR` expansion):

| Variable | What it is |
|---|---|
| `JIRA-SITE-URL` | e.g. `https://yourteam.atlassian.net` |
| `JIRA-EMAIL` | the Atlassian account email paired with the API token |
| `JIRA-API-TOKEN` | from https://id.atlassian.com/manage-profile/security/api-tokens |
| `JIRA-PROJECT` | the project key (e.g. `SCRUM`), not the project's display name |

## What it will not do

- Edit or delete the actual description field — it only comments.
- Repeat the same comment when nothing has changed since — it checks
  existing comments for the fingerprint marker `[auto-nudge:description:hash=...]`
  and compares it to the current description before acting.
- Give generic advice ("please add more detail") — every suggestion is
  tied to what's actually missing from that specific issue, recomputed
  fresh each time the text changes.
- Guess at a score it isn't confident in, or guess at anything it can't
  parse — it reports what it skipped and why instead.
