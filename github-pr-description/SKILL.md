---
name: github-pr-create
description: Creates or updates a GitHub Pull Request for the current branch, generating a description from committed changes and honoring any PR template.
---

# github-pr-create

Automatically creates or updates a GitHub Pull Request for the current branch using the `gh` CLI, generating a description that reflects only committed changes relative to the default branch.

## When to use

- When the user asks to open, create, or update a Pull Request
- When the user asks to push their work as a PR or submit changes for review

## Instructions

### 1. Guard: detect current and default branch

```bash
CURRENT_BRANCH=$(git rev-parse --abbrev-ref HEAD)
DEFAULT_BRANCH=$(gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name')
```

If `CURRENT_BRANCH` equals `DEFAULT_BRANCH`, **stop immediately** and tell the user:
> "You are on the default branch (`<branch>`). Please switch to a feature branch before creating a PR."

### 2. Gather committed changes (ignore local uncommitted work)

Fetch the remote default branch to ensure an accurate diff:

```bash
git fetch origin "$DEFAULT_BRANCH" --quiet
```

Collect the list of commits and the full diff of committed changes relative to the remote default branch:

```bash
COMMITS=$(git log --oneline "origin/$DEFAULT_BRANCH..HEAD")
DIFF=$(git diff "origin/$DEFAULT_BRANCH..HEAD")
```

If `COMMITS` is empty (no committed changes ahead of the default branch), **stop** and tell the user there is nothing to open a PR for.

### 3. Load the PR template (if present)

Check for a PR template and read it if found:

```bash
TEMPLATE_PATH=".github/PULL_REQUEST_TEMPLATE.md"
if [ -f "$TEMPLATE_PATH" ]; then
  TEMPLATE=$(cat "$TEMPLATE_PATH")
fi
```

### 4. Check whether a PR already exists for the current branch

```bash
PR_URL=$(gh pr list --head "$CURRENT_BRANCH" --json url --jq '.[0].url')
```

- If `PR_URL` is non-empty → a PR already exists → go to step **6 (update)**.
- If `PR_URL` is empty → no PR yet → go to step **5 (create)**.

### 5. Generate the PR description and create the PR

Using the commits, diff, and template (if present), generate a high-quality PR description that:
- **Opens with a high-level summary (TL;DR)** — see [TL;DR (required)](#tldr-required) for its shape and placement
- Fills in every section of the PR template (if one exists), replacing placeholder text with real content derived from the changes
- Does **not** invent changes that are not present in the diff

When generating the title, inspect the commits for [Conventional Commits](https://www.conventionalcommits.org/) prefixes (`feat`, `fix`, `chore`, `refactor`, `docs`, `test`, `perf`, `ci`, `build`, `style`, `revert`). Derive the title prefix using this logic:
- If all commits share the same type (and optionally the same scope), use `<type>(<scope>): <summary>` — e.g. `feat(auth): add OAuth2 login flow`.
- If commits share the same type but different scopes, omit the scope: `<type>: <summary>`.
- If commits span multiple types, use the **highest-priority** type present. Priority order (highest first): `fix` > `feat` > `perf` > `refactor` > `chore` > `docs` > `test` > `ci` > `build` > `style`.
- If no commit follows the Conventional Commits format, omit the prefix entirely and write a plain imperative title.

Create the PR:

```bash
gh pr create \
  --title "<generated title from commits>" \
  --body "<generated description>" \
  --base "$DEFAULT_BRANCH"
```

Report the URL returned by `gh pr create` to the user.

### 6. Update the description of an existing PR

Generate a fresh description using the same approach as step 5 (commits, diff, template), including the required TL;DR and re-deriving the semantic title prefix from the current commit list.

Update the PR:

```bash
gh pr edit "$PR_URL" \
  --title "<generated title>" \
  --body "<generated description>"
```

Tell the user the PR was updated and show the URL.

## TL;DR (required)

Every PR description MUST open with a high-level summary of what the PR is mainly about, so a
reviewer can grasp it at a glance before reading any detail.

### Shape

- 1–3 sentences, or at most 3 bullets.
- Answers two things: **what** changes, and **why** it matters.
- No file-by-file breakdown, no implementation detail, no restating the detailed sections verbatim.
  (This limit applies to the summary itself — not to the section that hosts it, which still carries its full detail.)
- Written for someone who has not read the ticket or the diff.

### Placement

The heading label is not fixed — `TL;DR`, `Summary`, or whatever vocabulary the template already
uses. What matters is that the summary comes **first**. Decide where it goes using the first rule
that matches:

1. **The template already has a summary-like section** (`Summary`, `Overview`, `Description`,
   `What`, `What & Why`, `Context`, `Motivation`, or similar) → the TL;DR becomes the first content
   of that section, and the section's detailed content follows it as subsequent paragraphs under the
   same heading. Do not add a second heading, and do not drop the detail in favour of the summary.
2. **The template exists but every section is a specific** (ticket link, checklist, testing notes,
   screenshots, risk, rollout) → place the TL;DR **above** the template body, as a short un-headed
   lead paragraph or under an explicit `## TL;DR` heading. Leave every template section in place.
3. **No template** → the body starts with the summary; a heading is optional.

## Description generation rules

- Always base the description on the **committed** diff only (step 2). Never guess or fabricate changes.
- If a PR template exists, treat each section header as a required field and fill it from the diff. Never remove, rename, or repurpose template sections — but you **may add** a leading summary section when the template offers no home for the TL;DR.
- Keep the title short (≤72 chars), imperative mood, no trailing period.
- Preserve Conventional Commits semantics in the title: derive the `<type>(<scope>):` prefix from the commits as described in step 5. Never invent a type that is not present in at least one commit.
- The body should be written in clear, plain English suitable for a code review audience.
- Never expose secrets, credentials, or PII found in the diff.

## Key constraints

- All GitHub operations MUST use the `gh` CLI — no direct API calls.
- The description MUST always lead with a high-level summary (see [TL;DR (required)](#tldr-required)), whatever the template looks like.
- Never run on the default branch (enforced in step 1).
- Only analyse **committed** changes (`git diff origin/<default>..HEAD`), not the working tree or staging area.
- If `gh` is not installed or not authenticated, stop and tell the user to install/authenticate it first.
