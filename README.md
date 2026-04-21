# 🤖 Agent Skills

A collection of reusable skills for AI coding agents. Each skill provides specialized knowledge, step-by-step instructions, and guardrails that an agent loads on demand to perform specific tasks reliably and consistently.

## 📖 What is a skill?

A **skill** is a structured Markdown file (`SKILL.md`) that an agent reads before tackling a specific task. It captures proven workflows, constraints, and generation rules so the agent doesn't have to guess — it just follows the playbook.

Skills are designed to be:
- 🔁 **Reusable** — drop them into any project or agent configuration
- 🧩 **Composable** — combine multiple skills for tasks that span domains
- ✅ **Opinionated** — encode best practices and guardrails directly

---

## 🗂️ Available skills

### 🔀 [`github-pr-create`](./github-pr-description/SKILL.md)

> Creates or updates a GitHub Pull Request for the current branch, generating a description from committed changes and honoring any PR template.

**When to use:** Ask your agent to *"open a PR"*, *"create a pull request"*, or *"submit my changes for review"*.

**What it does:**
- Guards against running on the default branch
- Analyses only **committed** changes (never the working tree)
- Detects and fills in an existing `.github/PULL_REQUEST_TEMPLATE.md`
- Creates a new PR or updates an existing one via the `gh` CLI
- Generates a concise, accurate title and body — no hallucinated changes

---

### 💬 [`github-pr-assess-comments`](./github-pr-assess-comments/SKILL.md)

> Fetches all unresolved review comments on the PR for the current branch, assesses their validity, proposes a fix for each, and prints a summary table.

**When to use:** Ask your agent to *"assess PR comments"*, *"triage review feedback"*, or *"summarise outstanding review threads"*.

**What it does:**
- Guards against running when no open PR exists for the current branch
- Fetches all unresolved review threads via the GitHub GraphQL API
- Gathers file and diff context for each commented line
- Classifies every comment as `✅ Valid`, `⚠️ Debatable`, or `❌ Invalid`
- Produces a concrete fix proposal for each comment
- Renders a Markdown summary table with author, file/line, validity, and proposed fix

---

## 🚀 Adding a new skill

1. Create a folder with a short, kebab-case name (e.g. `my-skill/`)
2. Add a `SKILL.md` file following the structure:
   - Front-matter with `name` and `description`
   - A **When to use** section
   - Step-by-step **Instructions**
   - **Constraints** and guardrails
3. Register the skill in this README under **Available skills**
