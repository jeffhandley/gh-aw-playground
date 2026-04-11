---
# Trigger - when should this workflow run?
on:
  slash_command: echo
  reaction: "rocket"
  status-comment: false
  workflow_dispatch:
    inputs:
      command:
        type: string
        description: "Command to invoke, excluding the leading `/`"
        required: true

# Outputs - what APIs and tools can the AI use?
safe-outputs:
  noop:
    report-as-issue: false
  add-comment:
    max: 1
  report-failure-as-issue: false

---

# Echo

Add a comment to echo the command that was used to invoke this workflow:

{{ #if github.event_name == 'workflow_dispatch' }}
> You manually dispatched the workflow with the following command:
> `/${{ github.event.inputs.command }}`
{{ /if }}

{{ #if needs.activation.outputs.slash_command }}
> You invoked the workflow using the following slash_command:
> `/${{ needs.activation.outputs.slash_command }}`
{{ /if }}
