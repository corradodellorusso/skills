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

### 🧶 [`yarn-catalog-update`](./yarn-catalog-update/SKILL.md)

> Checks all dependencies in the Yarn Catalog for available updates, lets the user pick the target version interactively (or auto-selects the latest minor in non-interactive mode), rewrites the catalog, and prints a change summary.

**When to use:** Ask your agent to *"update the yarn catalog"*, *"bump my dependencies"*, or *"check for outdated packages"*.

**What it does:**
- Guards against non-Yarn repositories and repos without a `catalog:` or `catalogs:` block in `.yarnrc.yml`
- Queries the npm registry for the latest patch, minor, and major versions of each catalog entry
- In **interactive mode**: prompts per-package for the desired update type (patch / minor / major / skip)
- In **non-interactive mode**: auto-selects the latest minor and reports available majors at the end
- Rewrites `.yarnrc.yml` atomically, preserving the original range prefix (`^`, `~`, exact)
- Prints a Markdown change table and reminds the user to run `yarn install`

---

### 🌳 [`graphql-ast-visitors`](./graphql-ast-visitors/SKILL.md)

> Step-by-step guide for mastering GraphQL AST Visitors with plain graphql-js — visitor patterns, schema traversal, and operation document traversal.

**When to use:** Ask your agent to *"write a GraphQL AST visitor"*, *"review this visitor"*, *"traverse a GraphQL schema"*, or *"extract fields from an operation"*.

**What it does:**
- Shows how to get an AST from both a schema (SDL or compiled `GraphQLSchema`) and an operation document
- Covers all visitor patterns (KindVisitor, enter/leave, generic, reducer, `visitInParallel`)
- Provides Kind → Node type reference tables for schema and operation nodes
- Includes practical examples (collect fields, rewrite nodes, early exit, skip subtrees)
- Includes a review checklist for validating existing visitors

---

## 🚀 Adding a new skill

1. Create a folder with a short, kebab-case name (e.g. `my-skill/`)
2. Add a `SKILL.md` file following the structure:
   - Front-matter with `name` and `description`
   - A **When to use** section
   - Step-by-step **Instructions**
   - **Constraints** and guardrails
3. Register the skill in this README under **Available skills**
