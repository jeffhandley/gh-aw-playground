# gh-aw-playground

A playground repository for experimenting with [GitHub Agentic Workflows](https://githubnext.com/) (`gh-aw`). This repo demonstrates how to build AI-powered GitHub automation using natural language workflow definitions.

## What is gh-aw?

GitHub Agentic Workflows (`gh-aw`) is a tool that lets you define GitHub Actions workflows using natural language markdown files. The `gh aw compile` command transforms these `.md` workflow definitions into standard GitHub Actions `.lock.yml` files that run with AI (GitHub Copilot) at their core.

## Workflows

This repository includes several example agentic workflows:

| Workflow | Trigger | Description |
|---|---|---|
| [`hello-world.md`](.github/workflows/hello-world.md) | `workflow_dispatch` | Picks a random baby name and writes a rhyming poem |
| [`pr-hello.md`](.github/workflows/pr-hello.md) | Pull request events | Greets new PRs with a welcome comment |
| [`pr-labeler.md`](.github/workflows/pr-labeler.md) | Pull request events | Analyzes PR changes and applies `bug`, `enhancement`, or `documentation` labels |
| [`pr-file-reviewer.md`](.github/workflows/pr-file-reviewer.md) | Pull request events | Reviews changed files and leaves inline review comments |

The `pr-labeler` and `pr-file-reviewer` workflows are orchestrated by [`workflow-dispatcher-prs.yml`](.github/workflows/workflow-dispatcher-prs.yml), which polls for updated PRs every 15 minutes and dispatches the appropriate workflows.

## Repository Structure

```
.github/
├── actions/
│   └── select-copilot-pat/   # Composite action to select a PAT from the token pool
├── agents/
│   └── agentic-workflows.agent.md  # Copilot agent for creating/updating workflows
├── aw/
│   └── actions-lock.json     # Pinned versions of GitHub Actions used by compiled workflows
└── workflows/
    ├── *.md                  # Agentic workflow source files (human-authored)
    ├── *.lock.yml            # Compiled workflow files (auto-generated, do not edit)
    ├── copilot-setup-steps.yml       # Installs the gh-aw CLI extension
    └── workflow-dispatcher-prs.yml   # Orchestrates PR workflows on a schedule
```

## How It Works

1. **Author** a workflow in plain markdown (e.g., `pr-labeler.md`) describing what the workflow should do, its inputs, and its safe outputs.
2. **Compile** the workflow with `gh aw compile <workflow-name>`, which generates a `.lock.yml` file.
3. **Run** the compiled workflow — GitHub Actions executes it, invoking GitHub Copilot to carry out the instructions using the declared safe outputs.

### Safe Outputs

Agentic workflows use *safe outputs* to constrain what actions the AI can take. This repo uses:

- `add-comment` — Post a comment on a PR or issue
- `add-labels` — Apply labels to a PR or issue
- `create-pull-request-review-comment` — Leave an inline review comment
- `submit-pull-request-review` — Submit a pull request review
- `noop` — No GitHub API side effects (useful for testing)

## Setup

### Prerequisites

- The `gh` CLI with the `gh-aw` extension installed
- GitHub Copilot access
- Secrets configured for the Copilot PAT pool (see below)

### Installing the gh-aw Extension

```bash
gh extension install gh-aw-actions/setup
```

Or trigger the [`copilot-setup-steps`](.github/workflows/copilot-setup-steps.yml) workflow to install it in your environment.

### Copilot PAT Pool

The workflows use a pool of Personal Access Tokens (stored as `COPILOT_PR_0` through `COPILOT_PR_9` repository secrets) to distribute load across multiple Copilot-enabled accounts. The [`select-copilot-pat`](.github/actions/select-copilot-pat/action.yml) composite action randomly selects one of the available tokens at runtime.

### Compiling Workflows

```bash
# Compile a single workflow
gh aw compile hello-world

# Compile and validate all workflows
gh aw compile --validate
```

The compiled `.lock.yml` files are committed to the repository and marked as auto-generated in [`.gitattributes`](.gitattributes).

## VS Code Integration

The repository includes VS Code configuration for the GitHub Agentic Workflows MCP server (`.vscode/mcp.json`), which enables Copilot to assist with authoring and editing workflow files directly in the editor.
