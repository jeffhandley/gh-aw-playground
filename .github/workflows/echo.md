---
on:
  slash_command: echo
  workflow_dispatch:
    inputs:
      item_number:
        description: "Issue or PR number to comment on"
        required: true
        type: string
      command:
        description: "The command text to echo back"
        required: true
        type: string
  steps:
    - name: Summarize workflow_dispatch inputs
      if: ${{ github.event_name == 'workflow_dispatch' }}
      shell: bash
      env:
        INPUT_ITEM_NUMBER: ${{ github.event.inputs.item_number || '' }}
        INPUT_COMMAND: ${{ github.event.inputs.command || '' }}
      run: |
        escape_summary_cell() {
          printf '%s' "$1" | sed ':a;N;$!ba;s/&/\&amp;/g;s/</\&lt;/g;s/>/\&gt;/g;s/\r//g;s/\n/<br>/g;s/|/\\|/g'
        }

        {
          echo
          echo '## workflow_dispatch inputs'
          echo
          echo '| Input | Value |'
          echo '| --- | --- |'
          printf '| `item_number` | <code>%s</code> |\n' "$(escape_summary_cell "$INPUT_ITEM_NUMBER")"
          printf '| `command` | <code>%s</code> |\n' "$(escape_summary_cell "$INPUT_COMMAND")"
        } >> "$GITHUB_STEP_SUMMARY"
permissions:
  contents: read
safe-outputs:
  add-comment:
    max: 1
    target: ${{ github.event.issue.number || github.event.pull_request.number || inputs.item_number }}
---

# Echo

You are a simple echo bot. Your only job is to echo back the command you received as a comment.

The command to echo is: "${{ needs.activation.outputs.slash_command || github.event.inputs.command }}"

Respond with a single comment that echoes back the command text exactly. Format your response as:

> 🔊 **Echo:** <the command text>

If no command text was provided, respond with:

> 🔇 **Echo:** No command received.

You MUST call the `add_comment` tool with the echo response. If the command is empty or missing, you MUST still call `add_comment` with the "no command received" message. Do NOT call noop — the comment IS the action.

The full context is:

${{ steps.sanitized.outputs.text }}
