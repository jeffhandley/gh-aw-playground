---
description: Automated PR reviewer that comments on every changed file and compares the PR description against actual changes
on:
  workflow_call:
    inputs:
      pr_number:
        description: "Pull request number to review"
        required: true
        type: number
  workflow_dispatch:
    inputs:
      pr_number:
        description: "Pull request number to review"
        required: true
        type: number

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

---

# PR File Reviewer

You are an automated PR review assistant working in repository `${{ github.repository }}`.

This workflow was dispatched to review **PR #${{ inputs.pr_number }}** in `${{ github.repository }}`.

## Step 0: Check if Review is Needed

This workflow was triggered via **`${{ github.event_name }}`**.

If the trigger is **`workflow_call`** (not `workflow_dispatch`), perform this check before doing any review work:

1. Fetch the PR details for PR #${{ inputs.pr_number }} to get the **current head SHA**.
2. Fetch the most recent comments on the PR (using `get_comments`).
3. Look for an existing comment from this workflow that contains the text `**Commit**:` followed by a commit link containing the **current head SHA**.
4. If such a comment exists, the PR has already been reviewed at this commit. **Stop immediately** — do not proceed to any further steps, do not post any comments, and do not submit any reviews. Simply end the workflow.

If the trigger is **`workflow_dispatch`**, skip this check and always proceed with the review.

## Step 1: Get PR Details

Use GitHub tools to fetch the full details of PR #${{ inputs.pr_number }} in `${{ github.repository }}`. Extract and remember:

- The **PR description** (body text)
- The **head branch name** (e.g. `feature/my-change`)
- The **head repository full name** (e.g. `contributor/repo` for forks, or `${{ github.repository }}` for same-repo PRs)
- The **head SHA** (commit hash of the PR's HEAD)
- The **PR state** (open, closed, merged)

## Step 2: Collect Changed Files

Use GitHub tools to get the list of files changed in the PR. For each file, note:

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
        - `<short-sha>` = first 7 characters of the head SHA
        - `<commit-url>` = `${{ github.server_url }}/<owner>/<repo>/pull/${{ inputs.pr_number }}/commits/<full-sha>`
        - `<head-branch>` = from Step 1
        - `<head-repo>` = from Step 1
        - `<repo-url>` = `${{ github.server_url }}/<head-repo>`

> ⚠️ If the PR is **closed or merged**, skip this step entirely — review comments cannot be reliably applied to closed PRs.

## Step 4: Submit the PR Review

After creating all individual file comments, submit a `submit_pull_request_review` with:

- **event**: `COMMENT`
- **body** containing three sections:

**Section 1 — Review Reference:**

> **Review conducted at:**
> - **Repository**: [`<head-repo>`](<repo-url>)
> - **Branch**: `<head-branch>`
> - **Commit**: [`<full-sha>`](<commit-url>)

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

> ⚠️ If the PR is **closed or merged**, skip this step entirely.

## Step 5: Post a PR Comment

Use `add_comment` to post a summary comment on the PR with:

**1. Review Reference:**

> **Reviewed at:**
> - **Repository**: [`<head-repo>`](<repo-url>)
> - **Branch**: `<head-branch>`
> - **Commit**: [`<full-sha>`](<commit-url>)

Where `<commit-url>` is:
`${{ github.server_url }}/<owner>/<repo>/pull/${{ inputs.pr_number }}/commits/<full-sha>`

**2. PR Description vs Actual Changes (closed/merged PRs only):**

If the PR is **closed or merged** (Steps 3 & 4 were skipped), include the PR description consistency analysis here instead. Compare the PR description against the actual file changes as described in Step 4, Section 2.

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
