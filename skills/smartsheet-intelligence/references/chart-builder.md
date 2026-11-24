# Skill: Chart & Metric Builder (Slise)

Triggers: "chart", "graph", "visualize", "plot", "show me", "how are X doing", "what's going on with X",
"add a chart to my dashboard", "add these to my dashboard", "add them all to the dashboard" — and, more
importantly, **any time the user is working on a sheet**, even before they ask for a visualization.

Build charts and metrics from sheet data using the Slise tools. This skill is **proactive and composing**:
don't wait to be asked for each step, and don't hand back a single chart when the data supports a richer answer.

**Tool access:** The dashboard and chart tools named below (`get_chart_suggestions`, `create_chart_from_prompt`,
`create_chart_from_data`, `refine_chart`, `explain_chart`, `query_chart`, `get_chart_definition`,
`add_charts_to_dashboard`, `list_dashboards`, `get_dashboard`, `create_dashboard_with_ai`) are NOT listed in `tools/list`.
You MUST first call `search_tools(toolset="dashboards")` to load them, then invoke them via
`execute_dashboards_read` (read tools) or `execute_dashboards_write` (`create_chart_from_prompt`,
`add_charts_to_dashboard`, `create_dashboard_with_ai`). Wherever this skill says to call one of them, go through that route.

## Core stance: be proactive, produce more than one thing

1. **On a sheet, volunteer the capability.** When the user is on a sheet (you have a `sheet_id`), proactively
   offer that you can chart it, and lead with concrete options from `get_chart_suggestions` — don't make the
   user discover the feature or name a tool. Show the suggested charts/metrics you can actually create.

2. **Bias toward MULTIPLE outputs.** For open-ended asks ("how are sales doing", "what's going on with this
   project") the right answer is usually several visuals, not one. Prefer generating **multiple charts and/or
   metrics** whenever the data supports it. A couple of headline metrics plus one or two charts is a great
   default shape for an open-ended question.

3. **Charts AND metrics — both are first-class outputs, but only charts go on dashboards.** A single-number
   metric (total, average, count, % complete) is a valid, valuable output. Use metrics for headline numbers and
   charts for distributions/trends/breakdowns. Dashboard placement accepts **charts only** — a metric answer
   cannot be placed and attempting it is rejected. Still build a mix; just don't offer to place the metrics.

4. **Don't stop at the chart — every chart or metric response carries ALL next actions.** The user must NEVER
   have to ask "what can I do next". Every time you return a chart or metric, include the full set of chart skills
   in that same response:
   - `refine_chart` — fetch and present the grouped refinement options (Try a different view / Refine the chart).
   - dashboard placement — offer it for chart answers only, via `add_charts_to_dashboard` in a single call.
   - `explain_chart` — offer (or proactively give) the explanation of how the result was produced.

   These are not optional follow-ups the user requests — they are baked into every chart AND metric response.
   Metrics get `refine_chart` and `explain_chart`, but not dashboard placement.

## The prompt endpoint can return multiple charts and metrics

`create_chart_from_prompt` starts (or continues) a conversation and returns a conversation whose **answer can
contain multiple messages** — several charts and/or metrics together. Render **every** chart/metric in the
answer, not just the first. The answer also carries the chart/metric definition AND its data, so render
directly — you do **not** need a separate query step on this flow. Keep the `conversation_id` and each answer
`message_id` for refinements and dashboard placement.

When the ask is open-ended, phrase the prompt to invite breadth, e.g. "Give me an overview of how sales are
doing — a few key metrics and a couple of charts" rather than forcing a single narrow chart.

## Presentation contract: title + description + options (do NOT flatten)

The suggestion and refinement responses are **nested**, and you must preserve that nesting when you
surface them. Do **not** collapse them into one flat list of actions. Each item the user sees has three
parts — a **title**, a **description**, and a set of **options** (a dropdown-style choice), e.g.:

```
Title:       Analyze
Description: Revenue by product to see which items drive the most dollars
Options:     [Bar, Pie, Stacked Bar]      ← pick one to create the chart

Title:       Refine the chart
Description: Tighten what the chart shows
Options:     [Group smaller items into "Other", Show only the top 5]
```

### From `get_chart_suggestions` (`categories[]`)

The response is `{ categories: [ { title, icon, suggestions: [ { title, description, presentations: [ { type, chartType, prompt } ] } ] } ] }`.

- **title** ← `suggestions[].title` (the category `categories[].title`, e.g. "Analyze" / "Compare", is the group header above a set of cards).
- **description** ← `suggestions[].description`.
- **options** ← that suggestion's `presentations[]`. Each presentation is one option: label it by `chartType` (Bar, Pie, Stacked Bar…). Selecting an option runs `create_chart_from_prompt` with that presentation's `prompt`.

So one suggestion = one card (title + description + an options dropdown of its chart types). One category = a labeled group of those cards. Keep both levels.

### From `refine_chart` (`refinements[]`)

The response is `{ refinements: [ { title, prompt, chartType, type } ] }`.

- **Group the flat `refinements[]` by `type`** (`CHART_TYPE`, `FILTER`, `CHANGE_AGGREGATION`, `SORT`, …) — that grouping is what turns a flat list into titled sections.
- A natural mapping: `CHART_TYPE` refinements → a "Try a different view" card whose **options** are the `chartType`s; all other types → a "Refine the chart" card whose **options** are the refinement `title`s.
- **title** ← the group label; **description** ← a short summary of what that group does; **options** ← each refinement's `title` (selecting one runs `create_chart_from_prompt` with its `prompt`).

### Rule

Always emit **title + description + options** per card, with categories/types as the grouping. Never hand back
a single flat list of prompts — that loses the title, the description, and the dropdown grouping Slise provides.

## Composed flows

### A. User is on a sheet (proactive entry point)

Don't wait for an explicit chart request. As soon as you have a `sheet_id`:

```
get_chart_suggestions   (sheet_id → suggested charts AND metrics you can create)
        ↓   present them as ready-to-go options ("I can build any of these — or all of them")
create_chart_from_prompt   (fire it for the option(s) the user wants, or the obvious headline ones)
```

### B. Open-ended question ("how are sales doing?")  → multiple charts/metrics

```
create_chart_from_prompt   (sheet_id + a breadth-inviting prompt asking for several metrics and charts)
        ↓   the answer contains MULTIPLE messages — render every chart and metric
for EACH chart, in the same response, include ALL of:
   refine_chart            (grouped refinement options)
   explain_chart           (offer/give the explanation)
then, for ALL charts together:
   add_charts_to_dashboard   (ONE call populating the dashboard with every chart)
```

This flow produces several charts, so pass them all to one `add_charts_to_dashboard` call — never a
sequence of calls.

### C. Specific request ("a chart of X by Y")

```
create_chart_from_prompt   (sheet_id + the specific prompt)
        ↓   render the chart(s) from the answer
include ALL next actions in the SAME response: refine_chart + dashboard placement + explain_chart
```

Even for a specific single-chart ask, the response still carries the full set of next actions (refine, place on
dashboard, explain) — the user never has to ask what's next. Placement always uses `add_charts_to_dashboard`,
whether the answer carried one chart or several.

### D. Applying a refinement

`refine_chart` returns suggestion objects only (`title` + `prompt`); it does **not** change the chart. To apply
one, call `create_chart_from_prompt` again with the **same** `conversation_id`, the previous answer's
`message_id` as `previous_message_id`, and the refinement's `prompt`. Render the refined result from the new answer.

## Dashboard

Charts can be placed on a dashboard. Offer this proactively once a chart exists — don't wait to be asked. Metric
answers cannot be placed and are rejected, so don't offer it for them.

### Placing charts

**Put every chart for a dashboard in one `add_charts_to_dashboard` call, and never run placement calls
concurrently.** Each call picks slots based on the layout it sees when it starts, so concurrent calls land widgets
on top of each other and can damage a dashboard the user already relies on. For "add them all" or "add these
two", send them together. For a later follow-up, send only the new charts; charts already placed would be added
again as duplicates.

Takes a `dashboard_id` and `charts`: a list of `conversation_id`, `message_id`, and `messageType` from the chart
answer, **max 10 per call**.

- More than 10 charts: split into consecutive calls of at most 10 and **wait for each to finish** before starting
  the next. Never run them concurrently.
- The response returns `dashboardPermalink` plus one `results` entry per chart, in the order you sent them.
  Surface the permalink as a "View on dashboard" link.

**Check `results` — a successful call does not mean every chart landed.** Each entry has a `type` of
`SUCCEEDED` or `FAILED`. Report honestly and specifically:

- All `SUCCEEDED` — confirm and give the "View on dashboard" link.
- Some `FAILED` — say which charts landed and which didn't. The successful ones are already on the dashboard;
  don't imply the whole operation failed, and don't silently drop the failures.
- A `FAILED` entry's `reason` tells you what to do next. Only `NOT_ATTEMPTED` is safe to retry: those charts were
  never sent, so re-send just those in a new call. For any other reason, do **not** retry automatically — the
  chart may already be on the dashboard and retrying can create a duplicate. Explain and let the user decide.
- `UNAUTHORIZED` means no edit access to that dashboard; `NOT_FOUND` means the chart or conversation is gone.
  Both are worth stating plainly rather than retrying.

Actively encourage assembling a dashboard: when a user is exploring a sheet, building a few metrics (headline
numbers) plus a couple of charts (trends/breakdowns) is the ideal outcome — then place the charts together in a
single `add_charts_to_dashboard` call. Bias toward offering to place the charts you create.

The `query_chart` and `get_chart_definition` tools operate on a chart already saved to a dashboard and are not
part of the inline conversation flow above.

## Tool reference

| Tool | Use it for |
|---|---|
| `get_chart_suggestions` | Proactively surface charts AND metrics you can create from a sheet, before anything exists |
| `create_chart_from_prompt` | Generate chart(s) and/or metric(s) — new conversation, or continue one to apply a refinement. Answer can carry MULTIPLE charts/metrics, each with its data to render |
| `refine_chart` | Fetch follow-up refinement options for an existing chart (returns options only); include in every chart response |
| `explain_chart` | Explain how a chart/metric answer was produced (summary + steps); include in every chart response |
| `add_charts_to_dashboard` | Place **1–10** charts onto a dashboard in a single call (returns `dashboardPermalink` + per-chart `results`). The tool for every dashboard placement, whether one chart or several |
| `query_chart` | Only for a chart already saved on a dashboard |
| `get_chart_definition` | Only for a chart already saved on a dashboard |

## Rules

- Be proactive: on a sheet, lead with suggestions; EVERY chart response carries all next actions
  (refine_chart + dashboard placement + explain_chart) — the user never asks "what can I do next".
- Prefer MULTIPLE charts/metrics for open-ended asks; render every message the answer returns, not just the first.
- Treat metrics as first-class outputs alongside charts — both can be refined and explained, but only charts can
  be placed on a dashboard.
- **Place charts with a single `add_charts_to_dashboard` call.** Never run placement calls concurrently — that
  overlaps widgets and can damage the dashboard. Never re-send a chart that is already placed.
- After a bulk placement, always read `results`: report which charts landed and which didn't, and only retry
  entries whose `reason` is `NOT_ATTEMPTED`.
- Keep each answer's `conversation_id` and `message_id` — refinements and dashboard placement both need them.
- Render directly from the `create_chart_from_prompt` answer — no separate query step on this flow.
