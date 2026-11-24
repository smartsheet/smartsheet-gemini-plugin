---
name: smartsheet-toolsets
description: Explains progressive tool loading in the Smartsheet MCP server — tool domains grouped into toolsets are discovered with search_tools and run through the execute_<toolset>_read and execute_<toolset>_write executors. Use when a needed Smartsheet tool is not in tools/list, or when the user asks to "list my portfolios", "show my reports", "manage automations", or another toolset operation. Don't use for core sheet tools listed directly in tools/list, such as get_sheet_summary or add_rows.
---

# Toolsets Intent

This intent provides guidance on using progressive tool loading via toolsets.

---

## TOOLSETS — Progressive Tool Loading

Some tool domains are organized into **toolsets** and are not listed directly in `tools/list`.
The executor tools (e.g. `execute_portfolio_read`, `execute_portfolio_write`) appear in `tools/list`
with descriptions of their capabilities. To use them, follow this two-step pattern:

1. **Discover tools:** Call `search_tools(toolset="<name>")` to see available tools and their schemas
2. **Execute:** Call the appropriate executor tool returned in the response (e.g. `execute_portfolio_read` or `execute_portfolio_write`)

**Rules:**
- Before calling `execute_{toolset}_read` or `execute_{toolset}_write` for the first time on any tool_name, you MUST call search_tools for that toolset first. Do not rely on prose descriptions of tool_name parameters from other guides as a substitute.
- Tools returned by `search_tools` MUST be invoked via their executor — never call them directly
- Use `execute_{toolset}_read` for read-only operations
- Use `execute_{toolset}_write` for mutations (create, delete)
- The executor's `tool_name` parameter accepts only the tools listed by `search_tools`

**Example — list portfolios:**
1. `search_tools(toolset="portfolio", access_type="read")` → returns tool definitions including `list_portfolios`
2. `execute_portfolio_read(tool_name="list_portfolios", arguments={})` → returns portfolio list
