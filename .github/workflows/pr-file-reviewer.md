---
description: Automated PR reviewer that comments on every changed file and compares the PR description against actual changes
on:
  workflow_call:
    inputs:
      pr_number:
        description: "Pull request number to review"
        required: true
        type: number
      head_sha:
        description: "Head SHA to review (fetched from PR by default)"
        required: false
        type: string
      artifact_id:
        description: "Unique identifier to prevent artifact name collisions when multiple workflows run in the same parent workflow"
        required: false
        type: string
  workflow_dispatch:
    inputs:
      pr_number:
        description: "Pull request number to review"
        required: true
        type: number
  permissions:
    contents: read
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
        COPILOT_PAT_0: ${{ secrets.COPILOT_PAT_0 }}
        COPILOT_PAT_1: ${{ secrets.COPILOT_PAT_1 }}
        COPILOT_PAT_2: ${{ secrets.COPILOT_PAT_2 }}
        COPILOT_PAT_3: ${{ secrets.COPILOT_PAT_3 }}
        COPILOT_PAT_4: ${{ secrets.COPILOT_PAT_4 }}
        COPILOT_PAT_5: ${{ secrets.COPILOT_PAT_5 }}
        COPILOT_PAT_6: ${{ secrets.COPILOT_PAT_6 }}
        COPILOT_PAT_7: ${{ secrets.COPILOT_PAT_7 }}
        COPILOT_PAT_8: ${{ secrets.COPILOT_PAT_8 }}
        COPILOT_PAT_9: ${{ secrets.COPILOT_PAT_9 }}

    - name: Get PR details
      id: pr_check
      env:
        GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        PR_NUMBER: ${{ inputs.pr_number }}
        INPUT_HEAD_SHA: ${{ inputs.head_sha }}
        REPO: ${{ github.repository }}
      run: |
        # Fetch PR metadata
        PR_JSON=$(gh pr view "$PR_NUMBER" --repo "$REPO" --json number,headRefOid,headRefName,headRepository,state)
        echo "pr_number=$(echo "$PR_JSON" | jq -r '.number')" >> "$GITHUB_OUTPUT"
        echo "head_branch=$(echo "$PR_JSON" | jq -r '.headRefName')" >> "$GITHUB_OUTPUT"
        echo "head_repo=$(echo "$PR_JSON" | jq -r '.headRepository.owner.login + "/" + .headRepository.name')" >> "$GITHUB_OUTPUT"
        echo "pr_state=$(echo "$PR_JSON" | jq -r '.state')" >> "$GITHUB_OUTPUT"

        # Use the provided head SHA if available, otherwise use the one from the PR
        if [ -n "$INPUT_HEAD_SHA" ]; then
          HEAD_SHA="$INPUT_HEAD_SHA"
        else
          HEAD_SHA=$(echo "$PR_JSON" | jq -r '.headRefOid')
        fi
        echo "head_sha=$HEAD_SHA" >> "$GITHUB_OUTPUT"
        echo "head_short_sha=${HEAD_SHA:0:7}" >> "$GITHUB_OUTPUT"

permissions:
  contents: read
  issues: read
  pull-requests: read

concurrency:
  group: pr-file-review-${{ inputs.pr_number }}
  cancel-in-progress: true

tools:
  github:
    toolsets: [default]

safe-outputs:
  create-pull-request-review-comment:
    max: 10
    side: "RIGHT"
  submit-pull-request-review:
    max: 1
  add-comment:
    max: 1
    discussions: false
    issues: false

jobs:
  pre-activation:
    outputs:
      copilot_pat_number: ${{ steps.select-copilot-pat.outputs.copilot_pat_number }}
      pr_number: ${{ steps.pr_check.outputs.pr_number }}
      head_sha: ${{ steps.pr_check.outputs.head_sha }}
      head_short_sha: ${{ steps.pr_check.outputs.head_short_sha }}
      head_branch: ${{ steps.pr_check.outputs.head_branch }}
      head_repo: ${{ steps.pr_check.outputs.head_repo }}
      pr_state: ${{ steps.pr_check.outputs.pr_state }}

engine:
  id: copilot
  env:
    # We cannot use line breaks in this expression as it leads to a syntax error in the compiled workflow
    # If none of the `COPILOT_PAT_#` secrets were selected, then the default COPILOT_GITHUB_TOKEN is used
    COPILOT_GITHUB_TOKEN: ${{ case(needs.pre_activation.outputs.copilot_pat_number == '0', secrets.COPILOT_PAT_0, needs.pre_activation.outputs.copilot_pat_number == '1', secrets.COPILOT_PAT_1, needs.pre_activation.outputs.copilot_pat_number == '2', secrets.COPILOT_PAT_2, needs.pre_activation.outputs.copilot_pat_number == '3', secrets.COPILOT_PAT_3, needs.pre_activation.outputs.copilot_pat_number == '4', secrets.COPILOT_PAT_4, needs.pre_activation.outputs.copilot_pat_number == '5', secrets.COPILOT_PAT_5, needs.pre_activation.outputs.copilot_pat_number == '6', secrets.COPILOT_PAT_6, needs.pre_activation.outputs.copilot_pat_number == '7', secrets.COPILOT_PAT_7, needs.pre_activation.outputs.copilot_pat_number == '8', secrets.COPILOT_PAT_8, needs.pre_activation.outputs.copilot_pat_number == '9', secrets.COPILOT_PAT_9, secrets.COPILOT_GITHUB_TOKEN) }}

---

# PR File Reviewer

You are an automated PR review assistant working in repository `${{ github.repository }}`.

## PR Context

The following PR details were gathered during pre-activation:

- **PR Number**: #${{ needs.pre_activation.outputs.pr_number }}
- **Head SHA**: `${{ needs.pre_activation.outputs.head_sha }}`
- **Short SHA**: `${{ needs.pre_activation.outputs.head_short_sha }}`
- **Head Branch**: `${{ needs.pre_activation.outputs.head_branch }}`
- **Head Repository**: `${{ needs.pre_activation.outputs.head_repo }}`
- **PR State**: `${{ needs.pre_activation.outputs.pr_state }}`
- **Server URL**: `${{ github.server_url }}`

## Step 1: Get PR Description

Use GitHub tools to fetch the PR description (body text) for PR #${{ needs.pre_activation.outputs.pr_number }} in `${{ github.repository }}`. You will need this for comparing against actual changes later.

## Step 2: Collect Changed Files

Use GitHub tools to get the list of files changed in PR #${{ needs.pre_activation.outputs.pr_number }}. For each file, note:

- File path
- Change status (added, modified, removed, renamed)
- The patch/diff content — you will need this to determine valid line numbers for review comments

## Step 3: Leave Review Comments (Max 10 Files)

For up to the **first 10** non-deleted files changed in the PR (in the order returned by the API):

1. **Skip deleted files entirely** — they have no lines to comment on.
2. For each non-deleted file:
   a. From the diff patch for the file, identify the **last line number on the RIGHT side** that appears in a diff hunk. This is the line you will comment on.
   b. Call `create_pull_request_review_comment` with:
      - `path`: the file path
      - `line`: the last RIGHT-side line number from the diff
      - `body`: a message in this format:

        ```
        ✅ I reviewed this file

        Reviewed at commit [`<short-sha>`](<commit-url>) on branch `<head-branch>` in [`<head-repo>`](<repo-url>)
        ```

        Where:
        - `<short-sha>` = the **Short SHA** from the PR Context
        - `<commit-url>` = `<server-url>/<owner>/<repo>/pull/<pr-number>/commits/<head-sha>` using values from the PR Context
        - `<head-branch>` = the **Head Branch** from the PR Context
        - `<head-repo>` = the **Head Repository** from the PR Context
        - `<repo-url>` = `<server-url>/<head-repo>`

> ⚠️ If the PR State is **CLOSED** or **MERGED**, skip this step entirely — review comments cannot be reliably applied to closed PRs.

## Step 4: Submit the PR Review

After creating all individual file comments, submit a `submit_pull_request_review` with:

- **event**: `COMMENT`
- **body** containing three sections:

**Section 1 — Review Reference:**

> **Review conducted at:**
> - **Repository**: [`<head-repo>`](<repo-url>)
> - **Branch**: `<head-branch>`
> - **Commit**: [`<head-sha>`](<commit-url>)

**Section 2 — PR Description vs Actual Changes:**

Compare the PR description (from Step 1) against the actual file changes (from Step 2). Assess:

- Are the described changes **consistent** with the actual diff?
- Does the description **omit** significant changes visible in the diff?
- Is the description **misleading** about any changes?
- If the PR description is **empty or missing**, explicitly note this and recommend adding a description.

Provide a brief, objective analysis. Do not speculate beyond what the diff and description show.

**Section 3 — Files Reviewed:**

List all files where you left review comments, and note any files that were skipped:
- Files beyond the first 10 (skipped due to limit)
- Deleted files (skipped because they have no lines to comment on)

> ⚠️ If the PR State is **CLOSED** or **MERGED**, skip this step entirely.

## Step 5: Post a PR Comment

Use `add_comment` to post a summary comment on the PR with:

**1. Review Reference:**

> **Reviewed at:**
> - **Repository**: [`<head-repo>`](<repo-url>)
> - **Branch**: `<head-branch>`
> - **Commit**: [`<head-sha>`](<commit-url>)

**2. PR Description vs Actual Changes (closed/merged PRs only):**

If the PR State is **CLOSED** or **MERGED** (Steps 3 & 4 were skipped), include the PR description consistency analysis here instead. Compare the PR description against the actual file changes as described in Step 4, Section 2.

**3. File Summary Table:**

List every file changed in the PR with its review status:

**For open PRs:**

| File | Status |
|------|--------|
| `path/to/file.js` | ✅ Review comment applied |
| `path/to/other.js` | ✅ Review comment applied |
| `path/to/deleted.js` | 🗑️ Deleted — no comment possible |
| `path/to/eleventh.js` | ⏭️ Skipped — beyond first 10 files |

**For closed/merged PRs:**

| File | Status |
|------|--------|
| `path/to/file.js` | 📋 Changed (closed PR, no inline comment) |
| `path/to/deleted.js` | 🗑️ Deleted |

**4. Totals:**

- Files changed: X
- Files with review comments applied: Y (0 for closed/merged)
- Files skipped (beyond limit): Z
- Files deleted: W
