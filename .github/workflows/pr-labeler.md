---
description: Automated PR labeler that applies bug, enhancement, and documentation labels based on PR changes, title, and description
on:
  workflow_call:
    inputs:
      pr_number:
        description: "Pull request number to label"
        required: true
        type: number
      head_sha:
        description: "Head SHA to analyze (fetched from PR by default)"
        required: false
        type: string
  workflow_dispatch:
    inputs:
      pr_number:
        description: "Pull request number to label"
        required: true
        type: number
  permissions:
    pull-requests: read
  steps:
    - uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6.0.2
      name: Checkout the select-copilot-pat action folder
      with:
        persist-credentials: false
        sparse-checkout: .github/actions/select-copilot-pat
        sparse-checkout-cone-mode: true
        fetch-depth: 1

    - id: select-copilot-pat
      name: Select Copilot token from pool
      uses: ./.github/actions/select-copilot-pat
      env:
        COPILOT_PAT_0: ${{ secrets.COPILOT_PR_0 }}
        COPILOT_PAT_1: ${{ secrets.COPILOT_PR_1 }}
        COPILOT_PAT_2: ${{ secrets.COPILOT_PR_2 }}
        COPILOT_PAT_3: ${{ secrets.COPILOT_PR_3 }}
        COPILOT_PAT_4: ${{ secrets.COPILOT_PR_4 }}
        COPILOT_PAT_5: ${{ secrets.COPILOT_PR_5 }}
        COPILOT_PAT_6: ${{ secrets.COPILOT_PR_6 }}
        COPILOT_PAT_7: ${{ secrets.COPILOT_PR_7 }}
        COPILOT_PAT_8: ${{ secrets.COPILOT_PR_8 }}
        COPILOT_PAT_9: ${{ secrets.COPILOT_PR_9 }}

    - name: Get PR details
      id: pr_check
      env:
        GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        PR_NUMBER: ${{ inputs.pr_number }}
        INPUT_HEAD_SHA: ${{ inputs.head_sha }}
        REPO: ${{ github.repository }}
      run: |
        # Fetch PR metadata
        PR_JSON=$(gh pr view "$PR_NUMBER" --repo "$REPO" --json number,headRefOid,headRefName,state,title,body)
        echo "pr_number=$(echo "$PR_JSON" | jq -r '.number')" >> "$GITHUB_OUTPUT"
        echo "pr_state=$(echo "$PR_JSON" | jq -r '.state')" >> "$GITHUB_OUTPUT"
        echo "pr_title=$(echo "$PR_JSON" | jq -r '.title')" >> "$GITHUB_OUTPUT"

        # Use the provided head SHA if available, otherwise use the one from the PR
        if [ -n "$INPUT_HEAD_SHA" ]; then
          HEAD_SHA="$INPUT_HEAD_SHA"
        else
          HEAD_SHA=$(echo "$PR_JSON" | jq -r '.headRefOid')
        fi
        echo "head_sha=$HEAD_SHA" >> "$GITHUB_OUTPUT"

        # Get current labels on the PR
        LABELS=$(gh pr view "$PR_NUMBER" --repo "$REPO" --json labels --jq '[.labels[].name] | join(",")')
        echo "current_labels=$LABELS" >> "$GITHUB_OUTPUT"

permissions:
  contents: read
  issues: read
  pull-requests: read

concurrency:
  group: pr-labeler-${{ inputs.pr_number }}
  cancel-in-progress: true

tools:
  github:
    toolsets: [default]

safe-outputs:
  add-labels:
    allowed:
      - bug
      - enhancement
      - documentation
  noop:

jobs:
  pre-activation:
    outputs:
      copilot_pat_number: ${{ steps.select-copilot-pat.outputs.copilot_pat_number }}
      pr_number: ${{ steps.pr_check.outputs.pr_number }}
      head_sha: ${{ steps.pr_check.outputs.head_sha }}
      pr_state: ${{ steps.pr_check.outputs.pr_state }}
      pr_title: ${{ steps.pr_check.outputs.pr_title }}
      current_labels: ${{ steps.pr_check.outputs.current_labels }}

engine:
  id: copilot
  env:
    # We cannot use line breaks in this expression as it leads to a syntax error in the compiled workflow
    # If none of the `COPILOT_PAT_#` secrets were selected, then the default COPILOT_GITHUB_TOKEN is used
    COPILOT_GITHUB_TOKEN: ${{ case(needs.pre_activation.outputs.copilot_pat_number == '0', secrets.COPILOT_PR_0, needs.pre_activation.outputs.copilot_pat_number == '1', secrets.COPILOT_PR_1, needs.pre_activation.outputs.copilot_pat_number == '2', secrets.COPILOT_PR_2, needs.pre_activation.outputs.copilot_pat_number == '3', secrets.COPILOT_PR_3, needs.pre_activation.outputs.copilot_pat_number == '4', secrets.COPILOT_PR_4, needs.pre_activation.outputs.copilot_pat_number == '5', secrets.COPILOT_PR_5, needs.pre_activation.outputs.copilot_pat_number == '6', secrets.COPILOT_PR_6, needs.pre_activation.outputs.copilot_pat_number == '7', secrets.COPILOT_PR_7, needs.pre_activation.outputs.copilot_pat_number == '8', secrets.COPILOT_PR_8, needs.pre_activation.outputs.copilot_pat_number == '9', secrets.COPILOT_PR_9, secrets.COPILOT_GITHUB_TOKEN) }}

---

# PR Labeler

You are an automated PR labeling assistant working in repository `${{ github.repository }}`.

## PR Context

The following PR details were gathered during pre-activation:

- **PR Number**: #${{ needs.pre_activation.outputs.pr_number }}
- **Head SHA**: `${{ needs.pre_activation.outputs.head_sha }}`
- **PR State**: `${{ needs.pre_activation.outputs.pr_state }}`
- **PR Title**: `${{ needs.pre_activation.outputs.pr_title }}`
- **Current Labels**: `${{ needs.pre_activation.outputs.current_labels }}`

## Your Task

You must analyze the PR and determine which of the following labels should be applied:

- **`bug`** — The PR fixes a bug, corrects incorrect behavior, addresses an error, or resolves a defect
- **`enhancement`** — The PR adds new functionality, improves existing features, refactors code, or makes performance improvements
- **`documentation`** — The PR adds or updates documentation files (README, docs/, guides), comments, docstrings, or any other form of documentation

### Rules

1. **Never remove labels** — only add labels that are not already present.
2. **Multiple labels are OK** — a PR can be both a bug fix and an enhancement, or include documentation changes alongside code changes.
3. **Be inclusive** — if a PR touches documentation even as part of a larger change, add the `documentation` label.
4. **Only add labels you are confident about** — do not guess. Base your decision on evidence from the title, description, and actual file changes.

## Step 1: Get PR Description

Use GitHub tools to fetch the full PR details for PR #${{ needs.pre_activation.outputs.pr_number }} in `${{ github.repository }}`. Read the title and body (description) carefully.

## Step 2: Collect Changed Files

Use GitHub tools to get the list of files changed in PR #${{ needs.pre_activation.outputs.pr_number }}. For each file, note:

- File path
- Change status (added, modified, removed, renamed)
- The patch/diff content

## Step 3: Analyze and Determine Labels

Based on the PR title, description, and actual file changes, determine which labels to apply:

### Bug indicators
- Title or description mentions "fix", "bug", "issue", "error", "crash", "broken", "patch", "resolve", "correct"
- Changes fix incorrect logic, error handling, or broken behavior
- Changes address a reported issue

### Enhancement indicators
- Title or description mentions "add", "feature", "improve", "enhance", "refactor", "update", "support", "implement", "optimize", "new"
- Changes add new files, new functions, new capabilities
- Changes improve performance or restructure code

### Documentation indicators
- Changes to `.md` files, `docs/` directories, `README` files
- Changes to code comments, docstrings, JSDoc, or similar inline documentation
- Title or description mentions "docs", "documentation", "readme", "guide", "instructions"

## Step 4: Apply Labels

Check which labels from `[bug, enhancement, documentation]` are already present in the current labels: `${{ needs.pre_activation.outputs.current_labels }}`.

For each label you determined should be applied in Step 3:
- If the label is **already present**, skip it
- If the label is **not yet present**, add it using the `add_labels` tool

If no new labels need to be added, call the `noop` tool to report that no labeling changes were needed.

## Step 5: Report Summary

Call the `noop` tool with a markdown-formatted summary containing:

1. **PR Analysis**: Brief description of what the PR does
2. **Labels Applied**: List which labels were added (if any)
3. **Labels Already Present**: List which labels were already on the PR
4. **Reasoning**: Brief explanation of why each label was or was not applied
