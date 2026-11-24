# Smartsheet plugin

Connects Gemini Agent to Smartsheet through the Smartsheet remote MCP server, so
people can find, analyze, and update their sheets and work with the discussions,
attachments, reports, dashboards, and automations around them.

## What's included

| Skill | Use it to |
| :--- | :--- |
| `smartsheet-intelligence` | Core orchestration: tool usage rules, intent routing, workflow chains, and safety rules for reads and writes. Includes references for building charts and authoring sheet automations. |
| `smartsheet-toolsets` | Discover and run toolset tools through `search_tools` and the `execute_<toolset>_read`/`_write` executors. |
| `smartsheet-stakeholder-reporting` | Draft status reports, executive summaries, and team updates. |

MCP servers (Streamable HTTP, OAuth 2.0). Connect to the one for your Smartsheet region:

| Region | Server |
| :--- | :--- |
| US | `https://mcp.smartsheet.com` |
| EU | `https://mcp.smartsheet.eu` |
| AU | `https://mcp.smartsheet.au` |

## Prerequisites

- A Smartsheet account on a plan with access to Smartsheet AI and MCP.
- Your Smartsheet system admin must allow AI features and the Gemini connector for
  your organization.
- The plugin acts with your own Smartsheet permissions. You can only read or change
  items shared with you, at the access level you have.

## Sign-in and permissions

The first time you use Smartsheet in Gemini, it asks you to sign in to Smartsheet and approve
access. The plugin requests access to read and write sheets, reports, dashboards,
workspaces, discussions, attachments, and automations on your behalf. You can revoke
access at any time from your Smartsheet account settings.

## How the tools are organized

The server lists core sheet tools directly. Everything else is grouped into
toolsets, which the agent opens on demand with `search_tools` and runs through
`execute_<toolset>_read` or `execute_<toolset>_write`. Read tools carry
`readOnlyHint: true`, so searches and lookups run right away.

## Limitations

- The plugin can't share items or change anyone's permissions.
- Agentic workflows can only be scheduled on whole hours.
- New or renamed items can take a few minutes to appear in search.

## Support

- Documentation: https://developers.smartsheet.com/ai-mcp/smartsheet/mcp-server
- Support: https://help.smartsheet.com/contact
- Privacy policy: https://www.smartsheet.com/legal/privacy
- Terms of service: https://www.smartsheet.com/legal/user-agreement
