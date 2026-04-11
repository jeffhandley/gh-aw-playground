---
steps:
  - name: Log dispatch inputs
    id: dump_inputs
    run: |
      echo "## Workflow Dispatch Inputs" >> "$GITHUB_STEP_SUMMARY"
      echo "" >> "$GITHUB_STEP_SUMMARY"
      echo '```json' >> "$GITHUB_STEP_SUMMARY"
      echo '${{ toJSON(github.event.inputs) }}' >> "$GITHUB_STEP_SUMMARY"
      echo '```' >> "$GITHUB_STEP_SUMMARY"---
---

# Log Dispatch Inputs

Workflows that support the `workflow_dispatch` event and accept inputs can import this shared workflow file to log the inputs to the `pre_activation` summary.

Include this top-level frontmatter:

```yml
imports:
  - shared/log-dispatch-inputs.md
```
