---
name: github-pr-assess-comments
description: Fetches all unresolved review comments on the PR for the current branch, assesses their validity, and prints a summary table. Comments are never fixed — only assessed.
---

# github-pr-assess-comments

Analyses every unresolved review thread on the Pull Request associated with the current branch. For each comment it evaluates whether the feedback is valid and renders a summary table.

> **Important:** This skill **only assesses** comments. It never applies fixes, edits files, or resolves threads — not even for short or trivial changes.

## When to use

- When the user asks to review, triage, or assess PR comments / review feedback
- When the user wants a summary of outstanding review threads
- When the user wants to know what reviewers said

## Instructions

### 1. Guard: verify a PR exists for the current branch

```bash
CURRENT_BRANCH=$(git rev-parse --abbrev-ref HEAD)
PR_JSON=$(gh pr view --json number,url,headRefName 2>/dev/null)
```

If the command fails or returns nothing, **stop immediately** and tell the user:
> "No open Pull Request found for the current branch (`<branch>`). Please open a PR first."

Extract the PR number and URL:

```bash
PR_NUMBER=$(echo "$PR_JSON" | jq '.number')
PR_URL=$(echo "$PR_JSON" | jq -r '.url')
```

### 2. Resolve the repository owner and name

```bash
REPO=$(gh repo view --json nameWithOwner --jq '.nameWithOwner')
OWNER=$(echo "$REPO" | cut -d'/' -f1)
REPO_NAME=$(echo "$REPO" | cut -d'/' -f2)
```

### 3. Fetch all unresolved review threads via GraphQL

Use the GitHub GraphQL API to retrieve review threads with their resolved state and associated comments:

```bash
gh api graphql -f query='
query($owner: String!, $repo: String!, $pr: Int!) {
  repository(owner: $owner, name: $repo) {
    pullRequest(number: $pr) {
      reviewThreads(first: 100) {
        nodes {
          isResolved
          path
          line
          comments(first: 10) {
            nodes {
              databaseId
              author { login }
              body
              createdAt
            }
          }
        }
      }
    }
  }
}' -f owner="$OWNER" -f repo="$REPO_NAME" -F pr="$PR_NUMBER"
```

Filter to keep only threads where `isResolved` is `false`. If no unresolved threads are found, tell the user:
> "No unresolved review comments found on PR #`<number>`. All threads are resolved."
Then stop.

### 4. Gather code context for each unresolved thread

For each unresolved thread, retrieve the relevant section of the file at the commented line to understand the surrounding code:

```bash
# Get the file content at the PR head
gh api "repos/$OWNER/$REPO_NAME/contents/<path>?ref=$CURRENT_BRANCH" \
  --jq '.content' | base64 --decode
```

Extract approximately 10 lines around the commented line to give enough context for assessment.

Also fetch the PR diff to understand what changed:

```bash
gh pr diff "$PR_NUMBER"
```

### 5. Assess each comment

> **Never apply any fix.** Do not edit files, do not resolve threads, and do not make code changes — regardless of how short or trivial the fix might seem.

For every unresolved comment, perform the following analysis using the comment body, the file context, and the PR diff:

**Validity assessment** — classify the comment as one of:
- `✅ Valid` — the feedback identifies a real issue (bug, security flaw, style violation, missing logic, etc.)
- `⚠️ Debatable` — the feedback is a matter of preference or opinion; a reasonable engineer could disagree
- `❌ Invalid` — the feedback is factually incorrect, already addressed, or not applicable to the changed code

**Suggested action** — for each comment, describe what *should* be done without doing it. Examples:
- For `✅ Valid`: describe the specific code change, refactor, or addition that would address the feedback
- For `⚠️ Debatable`: explain the trade-off and suggest a response the author could leave
- For `❌ Invalid`: suggest a reply explaining why the comment does not apply

### 6. Render the summary table

Print the results as a Markdown table with the following columns:

| # | File & Line | Author | Comment (excerpt) | Validity | Suggested Action |
|---|-------------|--------|-------------------|----------|------------------|

Rules for the table:
- **#** — sequential index (1-based)
- **File & Line** — relative file path and line number from the thread (e.g. `src/foo.ts:42`)
- **Author** — GitHub login of the comment author
- **Comment (excerpt)** — first 80 characters of the comment body, truncated with `…` if longer
- **Validity** — one of `✅ Valid`, `⚠️ Debatable`, or `❌ Invalid`
- **Suggested Action** — concise description of what to do (≤120 chars); longer descriptions go in a numbered list below the table, referenced by index

After the table, include a **Detail** section that expands every entry whose suggested action exceeded 120 characters, numbered to match the table index.

## Key constraints

- **Never apply fixes.** Do not edit any file, resolve any thread, or make any code change — regardless of how trivial the fix appears.
- All GitHub operations MUST use the `gh` CLI — no direct API calls with `curl` unless `gh api` is used.
- Never fabricate comments or code that are not retrieved from the API.
- Keep assessment objective: base validity on correctness, clarity, and relevance to the diff — not on personal style preferences unless a linter/style guide is referenced.
- Never expose secrets, credentials, or PII found in comment bodies or code.
- If `gh` is not installed or not authenticated, stop and tell the user to install/authenticate it first.
