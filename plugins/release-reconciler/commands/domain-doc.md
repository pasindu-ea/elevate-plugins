---
name: domain-doc
description: Generate or refresh a business/product-facing domain knowledge document for new users and product owners — what the product does, who uses it, the business rules that govern it, and the user flows through it. Not a developer README.
argument-hint: (none)
---

You write `Docs/DOMAIN.md` — the one document a brand-new team member or a
product owner could read start to finish and come away actually
understanding the product: what it does, who uses it, what governs its
behavior, and how people move through it. No git-diff context here — this
represents the **whole current domain**, not what changed recently.

This is a different document from anything `/release-reconcile` writes.
That command explains *how to run the code*. This one explains *what the
product is and does*, for someone who will never open the source.

## 1. Read everything before writing anything

- Backend: service/business-logic layers, domain models/entities, validators,
  state machines, permission/authorization checks, any constants or enums
  that represent domain concepts (statuses, categories, limits).
- Frontend (if one exists): routes/pages, the components on each, and what
  API calls or state changes each user action triggers.
- Do not start writing until you've actually traced these — a plausible
  guess based on a class name is not a substitute for reading the logic.

## 2. Write the document in this structure

### What this is
One paragraph, plain language, zero jargon. A product owner's boss should
be able to read it and understand what the product does.

### Who uses it
Every distinct user role or permission level you can find **in the actual
authorization code** (a roles enum, `[Authorize(Roles=...)]`-style
attributes, permission checks, RBAC config) — not roles you infer from UI
text alone. For each role, state plainly what they can and cannot do.

### Core concepts
The main entities/objects in the domain (e.g. "Order", "Sprint", "Invoice")
and how they relate to each other — a short relationship description each,
not a full ERD. Use the domain's own vocabulary, not generic CS terms.

### Business rules
The actual logic that constrains or governs behavior: validation rules,
calculations, state transitions, eligibility checks, limits and
thresholds. **Every rule here must cite where you found it** —
`file:line` or the function/method name — so a reader can verify it and
so a future run can tell if the rule changed. Never state a rule because
it "sounds right" given the entity name; state it because you read it.

### User flows
If there's a frontend: for each major journey (e.g. "Creating an order"),
a numbered, step-by-step walkthrough derived from the actual
routes/pages/components and what they call. If a flow is too
dynamic/data-driven to trace statically, say so explicitly instead of
inventing plausible-sounding steps.

### Glossary
Any domain-specific term that appears in the code or UI that a newcomer
wouldn't already know, defined in one line each.

### Needs product input
Anything you found genuinely ambiguous: a rule with no explanation of
*why* it exists, a role whose permissions look inconsistent with its
name, a flow that branches in a way the code doesn't explain the reason
for. This section is mandatory even if short — a document that claims
full understanding everywhere is not credible. Same principle as
`/release-reconcile`'s doc-debt ledger: honest gaps beat confident guesses.

## 3. Regenerate whole, not patch

Unlike `/release-reconcile`, there is no "since last release" scope here —
rebuild the entire document fresh each run against the current codebase.

## 4. Skip the PR if nothing meaningfully changed

Before committing, diff your freshly generated content against the
existing `Docs/DOMAIN.md` (if one exists). If the only differences are
incidental rewording with no actual change in facts (roles, rules, flows,
concepts), don't open a PR — a no-op regeneration is noise, not a
contribution. If there's a real change, proceed.

## 5. Open ONE gated pull request
- Branch: `docs/domain-refresh`
- `gh pr create` against the default branch
- Apply the label `ai-dlc/docs/pending-review`
- PR body: a short summary of what changed since the last version (new
  role found, a rule that changed, a flow that no longer matches the
  code) — not the whole document restated.

## Hard constraints
- You must NOT merge, and must NOT push to the default branch. A human approves.
- Every business rule and every role must be traceable to actual code you read.
- If you cannot confirm something, put it in "Needs product input" — do not guess.
- Exactly one PR per invocation. If the branch already exists, update it instead.
