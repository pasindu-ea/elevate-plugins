---
name: jira-sprint-nudge
description: Scan the current sprint for issues with no (or near-empty) description and ask the reporter to add one. Skips issues already nudged.
argument-hint: (none)
---

You check the active sprint for tasks missing a description, and ask the
person who created each one to add it. There is no git repository involved
here — you talk to Jira only, via `curl` + `jq` (both available).

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

For each issue, look at `fields.description`:
- `null`, or an empty ADF doc (no real text content when you walk its `content` nodes) → **missing**
- Real ADF content whose combined plain text is under ~15 characters (e.g. just "TBD" or "-") → **also missing** — a placeholder isn't a description
- Anything else with substantive text → **fine, skip it**

## 3. Skip issues you've already nudged
Before nudging, check `fields.comment.comments` (already included in the
fetch above) for any existing comment whose plain text contains the marker
`[auto-nudge:description]`. If found, skip this issue — don't nudge twice.
This is what keeps a recurring schedule from spamming the same task every run.

## 4. Post the nudge
For each genuinely missing-description issue, get the reporter's
`accountId` from `fields.reporter.accountId`. POST a comment in Atlassian
Document Format (ADF) that `@`-mentions them:

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
        {"type": "text", "text": " this task doesn't have a description yet — could you add one? A short summary of what's needed helps whoever picks it up. [auto-nudge:description]"}
      ]
    }]
  }
}
EOF
```
The `[auto-nudge:description]` marker at the end is required — it's how
step 3 recognizes this issue was already handled on a future run.

## 5. Report back
Your final output (not a Jira comment — this is what the platform logs)
must state, plainly:
- Sprint name and how many issues it has
- How many had a real description already (skipped)
- How many were nudged just now, with their issue keys
- How many were skipped because they were already nudged before
- Any issue you were unsure about and why (e.g. ADF you couldn't parse) — do not guess; say so

## Hard constraints
- Never edit or delete an issue's actual description field — you only comment.
- Never comment on an issue more than once for the same reason (see step 3).
- If Jira credentials are missing or a request fails, report the exact
  failing call and HTTP status — do not retry silently in a loop.
