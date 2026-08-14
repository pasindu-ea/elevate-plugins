# jira-pr-sync

Moves Jira issues along the board automatically as a PR moves through its
lifecycle, using the commit message convention your team already follows:

```
SCRUM-1: adding team member's work
```

The Jira ID is the token before the first colon on the first line of the
commit message — the plugin never reads the PR title or body for this,
only actual commits, and never acts on an ID mentioned without that exact
`ID: ` prefix.

## The one command, used twice

```
/jira-pr-status-sync "<from-status>" "<to-status>"
```

Both arguments are required. The command finds every unique Jira ID
referenced across the PR's commits, and for each one: checks its *current*
status, and only transitions it if that current status **exactly matches**
`<from-status>`. This single safety rule is why the same command is reused
for both lifecycle stages instead of two separate always-move scripts —
an issue someone already moved by hand is never dragged backward.

| Trigger | Call |
|---|---|
| PR opened | `/jira-pr-status-sync "In Progress" "In Review"` |
| PR merged | `/jira-pr-status-sync "In Review" "In Testing"` |

## What it needs

Three environment variables, read via `printenv` (hyphenated names):

| Variable | What it is |
|---|---|
| `JIRA-SITE-URL` | e.g. `https://yourteam.atlassian.net` |
| `JIRA-EMAIL` | the Atlassian account email paired with the API token |
| `JIRA-API-TOKEN` | from https://id.atlassian.com/manage-profile/security/api-tokens |

No `JIRA-PROJECT` needed — the Jira ID in each commit already encodes its
own project key, so this plugin works across projects without configuration.

## What it will not do

- Move an issue whose current status isn't exactly the `<from-status>` it
  was given — even if the "obvious" intent seems clear. This is the one
  rule that never bends.
- Read Jira IDs from the PR title or description, only commit messages.
- Force a transition via a field edit if the workflow has no direct
  transition to the target status — it reports that instead.
- Guess which transition ID to use — it always looks up the issue's
  available transitions fresh, since transition IDs are workflow-specific.

## Wiring

Two webhook rules, both keyed on GitHub `pull_request` events:

```jsonc
// Fires on PR opened
{
  "name": "jira-move-to-in-review",
  "platform": "github",
  "repository": { "url": "repository.clone_url" },
  "match-any": [{ "rule": "action==opened" }],
  "use-inputs": [{ "name": "pr-number", "value": "pull_request.number", "mandatory": true }],
  "use-plugins": [{ "plugin-name": "jira-pr-sync@elevate-plugins", "marketplace": "pasindu-ea/elevate-plugins", "slash-command": "/jira-pr-status-sync" }],
  "with-envs": [{ "name": "GITHUB-TOKEN", "value": "secrets.GITHUB-TOKEN", "mandatory": true }],
  "model": "claude-sonnet-5",
  "max-budget-usd": 2,
  "execute-prompt": "Run /jira-pr-status-sync \"In Progress\" \"In Review\" for PR #{{pr-number}}."
}
```

```jsonc
// Fires on PR merged (not just closed — merged specifically)
{
  "name": "jira-move-to-in-testing",
  "platform": "github",
  "repository": { "url": "repository.clone_url" },
  "match-any": [{ "rule": "action==closed&&pull_request.merged==true" }],
  "use-inputs": [{ "name": "pr-number", "value": "pull_request.number", "mandatory": true }],
  "use-plugins": [{ "plugin-name": "jira-pr-sync@elevate-plugins", "marketplace": "pasindu-ea/elevate-plugins", "slash-command": "/jira-pr-status-sync" }],
  "with-envs": [{ "name": "GITHUB-TOKEN", "value": "secrets.GITHUB-TOKEN", "mandatory": true }],
  "model": "claude-sonnet-5",
  "max-budget-usd": 2,
  "execute-prompt": "Run /jira-pr-status-sync \"In Review\" \"In Testing\" for PR #{{pr-number}}."
}
```

Note `action==closed&&pull_request.merged==true` specifically — a PR that's
closed *without* merging must not move anything to "In Testing".
