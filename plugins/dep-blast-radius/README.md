# dep-blast-radius

*"If I ship a breaking change, who breaks?"*

Today that question is answered by asking in Slack and waiting a day, or by
shipping and finding out. Neither scales past a handful of repos, and both put
the delay on someone else's release.

## Command

### `/blast-radius [package-name]`

1. Works out what the current repo **publishes** (npm `name`, NuGet
   `PackageId`, `pyproject` name, Go module path, Maven `artifactId`).
2. Lists every repo in the organization.
3. Reads each one's dependency manifests from its default branch.
4. Reports the repos that declare a dependency on it — **and the version each
   one is pinned to**.

Pass a package name explicitly to check one that this repo does not publish.

## Why manifests, not code search

A textual search for a package name matches READMEs, changelogs, comments and
lockfile noise. A manifest match in *dependency position* is exact, so the
number is quotable without an allowlist. It is also ~500× cheaper in rate
limit: the core REST API allows 5000 requests/hour against code search's 10 per
minute.

## What it will not do

- **It never writes to a consumer repo** — no PRs, no issues, no comments. It
  posts one comment on the repo that invoked it.
- **It never reports "no dependency" for a repo it could not read.** Repos that
  fail to scan are listed separately, because unknown and zero are different
  answers.
- **It reads default branches only**, and says so in every report.
- **It does not detect symbol-level breakage.** It answers *who depends on me*,
  not *which of my changes breaks them*. That is a separate, much noisier
  problem — this one is the reliable half.

## Requires

`GITHUB-TOKEN` with `repo` scope, already exported as `GH_TOKEN` by the
executor. Read-only scope is sufficient and preferable.
