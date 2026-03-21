---
permissions:
  contents: read

network:
  allowed:
    - defaults

safe-outputs:
  noop:

on:
  workflow_dispatch:
  permissions:
    contents: read

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
        COPILOT_PAT_0: ${{ secrets.COPILOT_HELLO_0 }}
        COPILOT_PAT_1: ${{ secrets.COPILOT_HELLO_1 }}
        COPILOT_PAT_2: ${{ secrets.COPILOT_HELLO_2 }}
        COPILOT_PAT_3: ${{ secrets.COPILOT_HELLO_3 }}
        COPILOT_PAT_4: ${{ secrets.COPILOT_HELLO_4 }}
        COPILOT_PAT_5: ${{ secrets.COPILOT_HELLO_5 }}
        COPILOT_PAT_6: ${{ secrets.COPILOT_HELLO_6 }}
        COPILOT_PAT_7: ${{ secrets.COPILOT_HELLO_7 }}
        COPILOT_PAT_8: ${{ secrets.COPILOT_HELLO_8 }}
        COPILOT_PAT_9: ${{ secrets.COPILOT_HELLO_9 }}

# Add the pre-activation output of the randomly selected PAT
jobs:
  pre-activation:
    outputs:
      copilot_pat_number: ${{ steps.select-copilot-pat.outputs.copilot_pat_number }}

# Override the COPILOT_GITHUB_TOKEN expression used in the activation job
# Consume the PAT number from the pre-activation step and select the corresponding secret
engine:
  id: copilot
  env:
    # We cannot use line breaks in this expression as it leads to a syntax error in the compiled workflow
    # If none of the `COPILOT_PAT_#` secrets were selected, then the default COPILOT_GITHUB_TOKEN is used
    COPILOT_GITHUB_TOKEN: ${{ case(needs.pre_activation.outputs.copilot_pat_number == '0', secrets.COPILOT_HELLO_0, needs.pre_activation.outputs.copilot_pat_number == '1', secrets.COPILOT_HELLO_1, needs.pre_activation.outputs.copilot_pat_number == '2', secrets.COPILOT_HELLO_2, needs.pre_activation.outputs.copilot_pat_number == '3', secrets.COPILOT_HELLO_3, needs.pre_activation.outputs.copilot_pat_number == '4', secrets.COPILOT_HELLO_4, needs.pre_activation.outputs.copilot_pat_number == '5', secrets.COPILOT_HELLO_5, needs.pre_activation.outputs.copilot_pat_number == '6', secrets.COPILOT_HELLO_6, needs.pre_activation.outputs.copilot_pat_number == '7', secrets.COPILOT_HELLO_7, needs.pre_activation.outputs.copilot_pat_number == '8', secrets.COPILOT_HELLO_8, needs.pre_activation.outputs.copilot_pat_number == '9', secrets.COPILOT_HELLO_9, secrets.COPILOT_GITHUB_TOKEN) }}

---

## Hello World - Baby Name Rhyme Generator

You are a fun, lighthearted comedian AI assistant.

### Your Task

1. **Pick a random name**: Choose a random popular baby name from the years 2000-2020 in the United States. Pick from a wide variety — don't always pick the same name. Use both boy and girl names. Be creative and surprising with your selection.

2. **Say hello**: Greet the person by name in an enthusiastic, friendly way.

3. **Make a funny rhyme**: Write a short, funny poem or limerick that rhymes with the chosen name. Keep it clean, playful, and family-friendly. Aim for 4-6 lines.

4. **Report your output**: Call the `noop` tool with a message containing:
   - The name you chose
   - Your greeting
   - The funny rhyme

Make sure the noop message is well-formatted with markdown so it renders nicely in the GitHub Actions step summary.
