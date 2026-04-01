---
description: Automated PR greeter that confirms comment responses on pull requests, including copilot-swe-agent PRs

permissions:
  contents: read
  issues: read
  pull-requests: read

network:
  allowed:
    - defaults

safe-outputs:
  add-comment:
    max: 1
    target: "triggering"
    hide-older-comments: true
    discussions: false
    issues: false
  noop:
    report-as-issue: false

on:
  pull_request:
    types: [opened, reopened, synchronize]
  bots:
    - "Copilot"
    - "app/copilot-swe-agent"
    - "copilot-swe-agent"
    - "copilot-swe-agent[bot]"
  permissions:
    contents: read
    pull-requests: read

  # #####################################################################
  # Override the GITHUB_COPILOT_TOKEN secret usage for the workflow
  # with a randomly-selected token from a pool of secrets.
  #
  # As soon as organization-level billing is offered for Agentic
  # Workflows, this stop-gap approach will be removed.
  #
  # Different workflows can use different pools of secrets if desired,
  # by changing the COPILOT_PAT_0 through COPILOT_PAT_9 secrets to
  # any desired secret names.
  #
  # References:
  # - https://github.github.com/gh-aw/reference/frontmatter/#custom-steps-steps
  # - https://github.github.com/gh-aw/reference/frontmatter/#custom-jobs-jobs
  # - https://github.github.com/gh-aw/reference/frontmatter/#job-outputs
  # - https://github.github.com/gh-aw/reference/frontmatter/#ai-engine-engine
  # - https://github.github.com/gh-aw/reference/engines/#engine-environment-variables
  # - https://docs.github.com/en/actions/reference/workflows-and-actions/expressions#case
  # - [Update agentic engine token handling to use user-provided secrets (github/gh-aw#18017)](https://github.com/github/gh-aw/pull/18017)

  # Add the pre-activation step of selecting a random PAT from the supplied secrets
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

    - name: Gather PR details
      id: pr_check
      env:
        GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        PR_NUMBER: ${{ github.event.pull_request.number }}
        REPO: ${{ github.repository }}
      run: |
        PR_JSON=$(gh pr view "$PR_NUMBER" --repo "$REPO" --json number,title,state,isDraft,headRefName,headRefOid,author,headRepository)
        PR_AUTHOR=$(echo "$PR_JSON" | jq -r '.author.login // "unknown"')
        HEAD_SHA=$(echo "$PR_JSON" | jq -r '.headRefOid')

        echo "pr_number=$(echo "$PR_JSON" | jq -r '.number')" >> "$GITHUB_OUTPUT"
        echo "pr_title=$(echo "$PR_JSON" | jq -r '.title')" >> "$GITHUB_OUTPUT"
        echo "pr_state=$(echo "$PR_JSON" | jq -r '.state')" >> "$GITHUB_OUTPUT"
        echo "is_draft=$(echo "$PR_JSON" | jq -r '.isDraft')" >> "$GITHUB_OUTPUT"
        echo "pr_author=$PR_AUTHOR" >> "$GITHUB_OUTPUT"
        echo "head_branch=$(echo "$PR_JSON" | jq -r '.headRefName')" >> "$GITHUB_OUTPUT"
        echo "head_repo=$(echo "$PR_JSON" | jq -r '.headRepository.owner.login + "/" + .headRepository.name')" >> "$GITHUB_OUTPUT"
        echo "head_sha=$HEAD_SHA" >> "$GITHUB_OUTPUT"
        echo "head_short_sha=${HEAD_SHA:0:7}" >> "$GITHUB_OUTPUT"

        if [ "$PR_AUTHOR" = "Copilot" ] || [ "$PR_AUTHOR" = "app/copilot-swe-agent" ] || [ "$PR_AUTHOR" = "copilot-swe-agent" ] || [ "$PR_AUTHOR" = "copilot-swe-agent[bot]" ]; then
          echo "is_copilot_swe_agent=true" >> "$GITHUB_OUTPUT"
        else
          echo "is_copilot_swe_agent=false" >> "$GITHUB_OUTPUT"
        fi

jobs:
  pre-activation:
    outputs:
      copilot_pat_number: ${{ steps.select-copilot-pat.outputs.copilot_pat_number }}
      pr_number: ${{ steps.pr_check.outputs.pr_number }}
      pr_title: ${{ steps.pr_check.outputs.pr_title }}
      pr_state: ${{ steps.pr_check.outputs.pr_state }}
      is_draft: ${{ steps.pr_check.outputs.is_draft }}
      pr_author: ${{ steps.pr_check.outputs.pr_author }}
      head_branch: ${{ steps.pr_check.outputs.head_branch }}
      head_repo: ${{ steps.pr_check.outputs.head_repo }}
      head_sha: ${{ steps.pr_check.outputs.head_sha }}
      head_short_sha: ${{ steps.pr_check.outputs.head_short_sha }}
      is_copilot_swe_agent: ${{ steps.pr_check.outputs.is_copilot_swe_agent }}

engine:
  id: copilot
  env:
    # We cannot use line breaks in this expression as it leads to a syntax error in the compiled workflow
    # If none of the `COPILOT_PAT_#` secrets were selected, then the default COPILOT_GITHUB_TOKEN is used
    COPILOT_GITHUB_TOKEN: ${{ case(needs.pre_activation.outputs.copilot_pat_number == '0', secrets.COPILOT_PAT_0, needs.pre_activation.outputs.copilot_pat_number == '1', secrets.COPILOT_PAT_1, needs.pre_activation.outputs.copilot_pat_number == '2', secrets.COPILOT_PAT_2, needs.pre_activation.outputs.copilot_pat_number == '3', secrets.COPILOT_PAT_3, needs.pre_activation.outputs.copilot_pat_number == '4', secrets.COPILOT_PAT_4, needs.pre_activation.outputs.copilot_pat_number == '5', secrets.COPILOT_PAT_5, needs.pre_activation.outputs.copilot_pat_number == '6', secrets.COPILOT_PAT_6, needs.pre_activation.outputs.copilot_pat_number == '7', secrets.COPILOT_PAT_7, needs.pre_activation.outputs.copilot_pat_number == '8', secrets.COPILOT_PAT_8, needs.pre_activation.outputs.copilot_pat_number == '9', secrets.COPILOT_PAT_9, secrets.COPILOT_GITHUB_TOKEN) }}
---

# PR Hello

You are a friendly automated greeter for pull request activity in `${{ github.repository }}`.

## PR Context

- **PR Number**: #${{ needs.pre_activation.outputs.pr_number }}
- **PR Title**: `${{ needs.pre_activation.outputs.pr_title }}`
- **Author**: `${{ needs.pre_activation.outputs.pr_author }}`
- **PR State**: `${{ needs.pre_activation.outputs.pr_state }}`
- **Draft**: `${{ needs.pre_activation.outputs.is_draft }}`
- **Head Branch**: `${{ needs.pre_activation.outputs.head_branch }}`
- **Head Repository**: `${{ needs.pre_activation.outputs.head_repo }}`
- **Head SHA**: `${{ needs.pre_activation.outputs.head_sha }}`
- **Short SHA**: `${{ needs.pre_activation.outputs.head_short_sha }}`
- **copilot-swe-agent PR**: `${{ needs.pre_activation.outputs.is_copilot_swe_agent }}`

## Your Task

Use `add_comment` to post exactly one brief markdown comment on the triggering pull request.

The comment must:

1. Start with `## PR Hello`.
2. Thank `${{ needs.pre_activation.outputs.pr_author }}` for opening or updating the PR, and mention the PR title.
3. Mention the head branch `${{ needs.pre_activation.outputs.head_branch }}` and short SHA `${{ needs.pre_activation.outputs.head_short_sha }}`.
4. If `copilot-swe-agent PR` is `true`, explicitly say this confirms the workflow can respond to PRs from `copilot-swe-agent`.
5. If the PR is a draft, briefly note that.
6. Stay upbeat, concise, and under 120 words.
7. Do not review the code or suggest changes; this is only a PR response test.

If the PR state `${{ needs.pre_activation.outputs.pr_state }}` is not `OPEN`, do not post a comment. Call `noop` with a short explanation instead.
