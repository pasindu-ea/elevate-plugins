---
name: jira-sprint-nudge
description: Scan the current sprint, rate every issue's description for actual clarity (not just presence), and ask the reporter to improve the weak ones with specific suggestions. Re-scores on every edit — acknowledges fixes, re-nudges if still weak, stays silent if nothing changed.
argument-hint: (none)
---

You check the active sprint for tasks whose description would leave whoever
picks them up confused, and ask the reporter — by name, with specific
suggestions — to improve it. There is no git repository involved here — you
talk to Jira only, via `curl` + `jq` (both available).

## 0. Read credentials
Environment variable names contain hyphens, so read them with `printenv`,
not `$VAR` shell expansion:
```bash
JIRA_SITE="$(printenv 'JIRA-SITE-URL')"
JIRA_EMAIL="$(printenv 'JIRA-EMAIL')"
JIRA_TOKEN="$(printenv 'JIRA-API-TOKEN')"
JIRA_PROJECT="$(printenv 'JIRA-PROJECT')"
AUTH=(-u "${JIRA_EMAIL}:${JIRA_TOKEN}")
```
All requests below use `"${AUTH[@]}"` for auth and go to `${JIRA_SITE}/rest/...`.
If any of the four is empty, stop and report which one — do not guess a value.

## 1. Find the active sprint
- Resolve the project's board: `GET /rest/agile/1.0/board?projectKeyOrId=${JIRA_PROJECT}` → take the first board's `id`.
- Resolve the active sprint: `GET /rest/agile/1.0/board/{boardId}/sprint?state=active` → take `values[0].id` and `.name`.
- If there is no active sprint, post nothing and stop — say so plainly in your final output.

## 2. List the sprint's issues
`GET /rest/agile/1.0/sprint/{sprintId}/issue?fields=summary,description,reporter,comment`

For each issue, convert `fields.description` (Atlassian Document Format)
to plain text by walking its `content` nodes and concatenating `text`
values. An empty/null description is plain text "" (zero characters).

## 3. Rate the description — a 1–5 rubric, not a yes/no check
Judge the plain text you extracted against the issue's own `summary`
(title) for context. Score on this rubric; do not invent your own scale:

| Score | Meaning |
|---|---|
| **5** | Clear what to do, why, and what "done" looks like. No contradictions. |
| **4** | Clear what to do; minor gaps (e.g. no acceptance criteria) but usable as-is. |
| **3** | Understandable, but the assignee would need to ask a clarifying question before starting. |
| **2** | Vague or generic enough that the actual intent is a guess — OR restates the title with no new information. |
| **1** | Empty, a placeholder ("TBD", "-", "tbd"), a single disconnected word, or **self-contradictory** (e.g. says both "read-only" and "allow editing" for the same field; gives two different deadlines; asks for X in one sentence and explicitly not-X in another). |

Flag contradictions explicitly and quote the two conflicting phrases
verbatim when you find them — don't just lower the score silently.

**Threshold: nudge any issue scoring 3 or below.** Skip issues scoring 4–5.

## 4. Decide whether to act — re-check on every change, not just once
Every comment this plugin ever posts ends with a marker that fingerprints
the description it was reacting to:
```
[auto-nudge:description:hash=<12-hex-chars>]
```
Compute the fingerprint the same way every time: collapse the plain text
to single spaces, trim, then take the first 12 hex characters of its
SHA-256:
```bash
CURRENT_HASH="$(printf '%s' "${DESC_PLAIN_TEXT}" | tr -s '[:space:]' ' ' | sed 's/^ //;s/ $//' | sha256sum | cut -c1-12)"
```

Find the **most recent** comment (by `created` timestamp) whose plain text
contains `[auto-nudge:description:hash=`, and extract its hash.

- **No such comment exists** (never nudged before):
  - Score ≤3 → this is a **new nudge** (§5a).
  - Score ≥4 → nothing to do, skip silently.
- **A previous marker exists, and its hash equals `CURRENT_HASH`**:
  the description hasn't changed since that comment — skip. This is what
  keeps a recurring schedule from repeating itself every run.
- **A previous marker exists, but its hash differs from `CURRENT_HASH`**:
  the reporter edited the description since we last commented. Re-score
  the *current* text fresh, ignoring the old score entirely:
  - New score ≤3 → this is an **updated nudge** (§5b) — it changed, but still needs work.
  - New score ≥4 → this is an **improvement acknowledgment** (§5c) — it changed, and it's now good.

## 5. Post the right comment
All three variants `@`-mention the reporter (`fields.reporter.accountId`)
and end with the current `[auto-nudge:description:hash=${CURRENT_HASH}]`
marker — every posted comment becomes the new baseline for the next run.

**5a. New nudge** — write 1–3 short, concrete, issue-specific suggestions,
not generic advice like "please add more detail." Base them on what's
actually missing: no acceptance criteria, no mention of which
component/file/screen is affected, a contradiction that needs resolving,
no reason given for why the work matters, etc.

```bash
curl -s "${AUTH[@]}" -X POST -H "Content-Type: application/json" \
  "${JIRA_SITE}/rest/api/3/issue/${ISSUE_KEY}/comment" \
  -d @- <<EOF
{
  "body": {
    "type": "doc", "version": 1,
    "content": [
      {
        "type": "paragraph",
        "content": [
          {"type": "mention", "attrs": {"id": "${REPORTER_ACCOUNT_ID}"}},
          {"type": "text", "text": " this description scored ${SCORE}/5 for clarity — could you fill in a bit more? [auto-nudge:description:hash=${CURRENT_HASH}]"}
        ]
      },
      {
        "type": "bulletList",
        "content": [
          {"type": "listItem", "content": [{"type": "paragraph", "content": [{"type": "text", "text": "${SUGGESTION_1}"}]}]},
          {"type": "listItem", "content": [{"type": "paragraph", "content": [{"type": "text", "text": "${SUGGESTION_2}"}]}]}
        ]
      }
    ]
  }
}
EOF
```
Include a `bulletList` item per suggestion (1–3 items; omit the second
`listItem` block if you only have one suggestion). If you flagged a
contradiction, make that its own bullet quoting both conflicting phrases.

**5b. Updated nudge** — same shape as §5a, but say explicitly that it
changed and where it still falls short, e.g. "thanks for the update — this
now scores ${SCORE}/5, better than before, but..." List fresh suggestions
against the *current* text; don't reuse the old ones verbatim if the
description has moved on.

**5c. Improvement acknowledgment** — short, no bullet list, no
suggestions needed:
```bash
curl -s "${AUTH[@]}" -X POST -H "Content-Type: application/json" \
  "${JIRA_SITE}/rest/api/3/issue/${ISSUE_KEY}/comment" \
  -d @- <<EOF
{
  "body": {
    "type": "doc", "version": 1,
    "content": [{
      "type": "paragraph",
      "content": [
        {"type": "mention", "attrs": {"id": "${REPORTER_ACCOUNT_ID}"}},
        {"type": "text", "text": " thanks — this description is much clearer now (${SCORE}/5). [auto-nudge:description:hash=${CURRENT_HASH}]"}
      ]
    }]
  }
}
EOF
```

## 6. Report back
Your final output (not a Jira comment — this is what the platform logs)
must state, plainly:
- Sprint name and how many issues it has
- Score distribution across all issues (e.g. "5/5: 3 issues, 4/5: 2, 3/5: 1, 2/5: 1, 1/5: 0")
- New nudges posted, their score, and a one-line summary of the suggestions
- Updated nudges posted (description changed but still weak), old vs. new score
- Improvement acknowledgments posted (description changed and is now good)
- Any contradictions you found, quoted
- How many were skipped because the description hasn't changed since the last comment
- Any issue whose description you genuinely couldn't judge (e.g. unparseable ADF, non-English text you're not confident reading) — do not guess a score; say so instead

## Hard constraints
- Never edit or delete an issue's actual description field — you only comment.
- Never post a comment when the description hasn't changed since your last
  comment on that issue (see step 4's hash comparison).
- Suggestions must be specific to that issue's actual gap, not a template
  reused across issues or across an issue's own nudge history.
- Be direct but not harsh — this is feedback to a real person on your team,
  not a code review of a stranger's pull request.
- If Jira credentials are missing or a request fails, report the exact
  failing call and HTTP status — do not retry silently in a loop.
