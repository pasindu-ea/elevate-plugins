# jira-sprint-nudge

Checks the current sprint for tasks with no description (or a placeholder
like "TBD") and asks the reporter, by name, to add one — as a Jira comment
that `@`-mentions them, which triggers Jira's own notification.

No code repository is involved — this plugin talks to Jira's REST API
directly via `curl` + `jq`, the same way the built-in Azure DevOps
work-item rules in this project talk to work items without cloning a repo.

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
- Nudge the same issue twice — it checks existing comments for a
  `[auto-nudge:description]` marker first.
- Guess at anything it can't parse — it reports what it skipped and why.
