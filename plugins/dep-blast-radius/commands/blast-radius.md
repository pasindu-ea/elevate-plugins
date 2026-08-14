---
name: blast-radius
description: Find which other repos in this GitHub organization declare a dependency on the package this repo publishes, so a breaking change is announced before it ships. Usage - /blast-radius [package-name]
argument-hint: [package-name]
---

You answer one question, exactly: **if this repo ships a breaking change, which
other repos in this organization break?**

You answer it from **dependency manifests**, never from prose. A README that
mentions the package name is not a consumer. A `PackageReference` is. This
distinction is the whole reliability of the report — a textual match would
flood the output with false positives and make the number unquotable.

You are **read-only in every repo except the one that invoked you.** You never
open a pull request, branch, or issue in a consumer repo. You report; humans
decide.

## 0. Credentials
`GH_TOKEN` is already exported by the executor and `gh` is authenticated
against every repo the token can see. No setup needed. Verify once:
```bash
gh api user --jq '.login'
```
If that fails, stop and report the failure — do not fall back to unauthenticated
calls, which silently return 404 for private repos and would let you report a
blast radius of zero when the truth is unknown.

## 1. Determine what this repo publishes
If `$1` is given, use it as the package name and skip this step.

Otherwise detect it from the checked-out repo. Check every ecosystem present —
a repo can publish more than one package:

| Ecosystem | Where the published name lives | Rule |
|---|---|---|
| npm | `package.json` → `.name` | Skip if `"private": true` — it is not published |
| NuGet | `*.csproj` → `<PackageId>` | If absent but `<IsPackable>true</IsPackable>` or `<GeneratePackageOnBuild>true`, the id defaults to the `.csproj` filename |
| Python | `pyproject.toml` → `[project] name` (or `setup.py` `name=`) | — |
| Go | `go.mod` → `module <path>` | The module path IS the import path consumers use |
| Maven | `pom.xml` → `<artifactId>` | — |

Ignore anything under `node_modules/`, `bin/`, `obj/`, or a test fixture
directory.

**If this repo publishes nothing, say so and stop.** That is a real, useful
answer — it means no repo in the org can break via a manifest, and the correct
report is:

> This repo publishes no package. Package-level blast radius is zero by
> construction. If it is consumed, it is consumed as a running service, a
> container image, or by copy-paste — none of which a manifest scan can see.

**Post that report as a comment before you stop** — same single-comment
mechanics as §5, on the issue or PR whose number is stated in the prompt.
"Stop" here means skip §2–§4 (there is nothing to search for), not skip
reporting: a run that reaches a valid conclusion and never tells anyone is
indistinguishable from a run that crashed.

Do not invent a package name to have something to search for.

## 2. List every repo in the organization
The org is the owner of the repo you were invoked on.
```bash
ORG="$(gh repo view --json owner --jq '.owner.login')"
SELF="$(gh repo view --json nameWithOwner --jq '.nameWithOwner')"
gh api "orgs/${ORG}/repos?per_page=100&type=all" --paginate \
  --jq '.[] | [.full_name, .default_branch, (.archived|tostring)] | @tsv'
```
If the org endpoint 404s, the owner is a user account, not an organization —
use `users/${ORG}/repos` instead.

Skip `${SELF}`. Skip archived repos, but **count them and say how many you
skipped** — an archived consumer is not a release blocker, and silently
dropping repos makes the report look more complete than it is.

## 3. Read each repo's manifests
For each remaining repo, list its manifest files from the default branch in one
call, then fetch only those:
```bash
gh api "repos/${REPO}/git/trees/${BRANCH}?recursive=1" \
  --jq '.tree[] | select(.type=="blob") | .path' \
  | grep -Ev '(^|/)node_modules/' \
  | grep -E '(^|/)(package\.json|requirements\.txt|pyproject\.toml|go\.mod|pom\.xml)$|\.csproj$'

gh api "repos/${REPO}/contents/${PATH}?ref=${BRANCH}" -H "Accept: application/vnd.github.raw"
```
This uses the core REST API (5000 requests/hour), not code search (10 per
minute) — so you can scan a few hundred repos without pacing. Do not use
`gh search code` for this; it is rate-limited an order of magnitude harder and
matches prose.

If a tree call fails (empty repo, no access), record the repo as **not scanned**
with the reason. Do not treat a failed scan as "no dependency found."

## 4. Match in dependency position only
A hit requires the package name to appear as a **declared dependency**, not
anywhere in the file:

| Manifest | Counts as a hit |
|---|---|
| `package.json` | A key in `dependencies`, `devDependencies`, `peerDependencies`, or `optionalDependencies` |
| `*.csproj` | `<PackageReference Include="<name>" ...>` |
| `go.mod` | A `require` line whose module path is (or is a prefix of) the name |
| `pom.xml` | An `<artifactId>` inside a `<dependency>` block |
| `requirements.txt` | A line beginning with the name, followed by whitespace, `=`, `<`, `>`, `~`, `!`, or end of line |

**Record the declared version range for every hit.** Who is on which version is
half the value: a consumer pinned to an old major is not affected by a breaking
change to the current one, and a consumer on `^1.0.0` will pick up your break
automatically on their next install.

## 5. Report
Post **one comment** on the issue or PR that invoked you (its number is stated
in the prompt). Use exactly these sections:

```
## Blast radius — <package name(s)>

Scanned N repos in <org>. Skipped M archived, K not scanned.

### Consumers (n)
| Repo | Manifest | Declared version |
|---|---|---|
| org/service-a | package.json | ^2.1.0 |
| org/service-b | src/Api/Api.csproj | 2.0.4 |

### On a version that would NOT pick up a break (n)
| org/legacy-tool | package.json | ~1.4.0 (pinned to old major) |

### Not scanned (n)
| org/private-thing | tree API returned 403 |
```

Then one closing line stating the operational consequence in plain words, e.g.
*"A breaking change to `@xianix/core` requires coordinating with 2 repos; 1 more
is pinned to v1 and unaffected."*

If there are zero consumers, say so in one line and post nothing else. Silence
padded with sections reads as a broken tool.

## Hard constraints
- **Never write to a consumer repo.** No PRs, no branches, no issues, no
  comments. One comment, on the repo that invoked you, and nothing else.
- **Never report a repo you could not read as having no dependency.** Unknown
  and zero are different answers, and conflating them is the one failure mode
  that makes this report dangerous to trust.
- **Default branches only.** State this in the comment — a consumer who adds
  the dependency on an unmerged branch will not appear, and the reader needs to
  know that limit rather than discover it.
- Report the version each consumer declares, never just the repo name.
