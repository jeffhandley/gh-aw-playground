---
on:
  workflow_dispatch:
  permissions:
    contents: read

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

jobs:
  pre-activation:
    outputs:
      copilot_pat_number: ${{ steps.select-copilot-pat.outputs.copilot_pat_number }}

engine:
  id: copilot
  env:
    HELLO_NUMBER: ${{ needs.pre_activation.outputs.copilot_pat_number }}
    HELLO_LOOKUP: ${{ case(needs.pre_activation.outputs.copilot_pat_number == '0', 'Secret 0', needs.pre-activation.outputs.copilot_pat_number == '1', 'Secret 1', needs.pre-activation.outputs.copilot_pat_number == '2', 'Secret 2', needs.pre-activation.outputs.copilot_pat_number == 2, 'Numeric 2!', 'Huh; no match') }}
    COPILOT_GITHUB_TOKEN: ${{ case(needs.pre_activation.outputs.copilot_pat_number == '0', secrets.COPILOT_HELLO_0, needs.pre-activation.outputs.copilot_pat_number == '1', secrets.COPILOT_HELLO_1, needs.pre-activation.outputs.copilot_pat_number == '2', secrets.COPILOT_HELLO_2, needs.pre-activation.outputs.copilot_pat_number == 2, secrets.COPILOT_HELLO_2, secrets.COPILOT_GITHUB_TOKEN) }}

permissions:
  contents: read

network:
  allowed:
    - defaults

safe-outputs:
  noop:
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
