---
# Trigger - when should this workflow run?
on: /echo

# Permissions - what can this workflow access?
# Write operations (creating issues, PRs, comments, etc.) are handled
# automatically by the safe-outputs job with its own scoped permissions.
permissions:
  contents: read
  issues: read
  pull-requests: read

# Network access
network: defaults

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

> You invoked the workflow using: `/${{ needs.activation.outputs.slash_command }}`
