---
name: smartsheet-stakeholder-reporting
description: Drafts stakeholder communications from Smartsheet data — status reports, executive summaries, team updates, and VP-ready briefs grounded in sheet rows and discussion comments. Use when the user asks for a "status report", a "status update", something "for my VP", an "executive summary", or a "team update". Don't use for raw data questions or row-level analysis of a sheet, or for scheduling recurring emailed summaries.
---

# Skill: Stakeholder Reporting

Triggers: "status report", "status update", "for my VP", "executive summary", "team update"

## Tool Chain

1. Call `search_tools(toolset="workspaces", access_type="read")` and `search_tools(toolset="discussions", access_type="read")` first.
2. Locate the target sheet via `execute_workspaces_read` (`tool_name: "search"`).
3. Load current row and status data via `get_sheet_summary`.
4. Pull discussion comments via `execute_discussions_read` (`tool_name: "list_discussions"`, `arguments: {"sheet_id": ..., "target_type": "SHEET", "include_comments": true}`).

## Output Rules

After loading the data, go straight into the drafts without long preamble or raw row dumps:

- **For VP / executive asks with material risk:** Open with one line ("Here's your VP update — two versions depending on how you want to frame it.") and draft TWO variants:
  - **"Lead with risk"** — status `AT RISK` in subject, #1 blocker first, clear decision ask.
  - **"Lead with progress"** — accomplishments first, risks paired with mitigations.
- **For team updates or general status reports:** Open with one tailored line matching the audience (e.g. "Here's your team status update:") and produce a single clear draft.
- **Drafting output:** Write the drafts directly in chat (subject line and body). If the client has a message-drafting tool, you may use it as well.
- Weave in discussion context and keep each draft under 300 words.
- Close with one sentence offering to reformat for Slack or a different audience.
