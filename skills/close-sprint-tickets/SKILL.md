---
name: close-sprint-tickets
description: Close out your tickets in the current open sprint in your Jira Cloud instance. Lists all open issues (any type) assigned to you across selectable projects, lets you mark a subset as Done (auto-logs remaining estimate as Time Spent so Remaining = 0), and transitions the rest to user-chosen statuses. TRIGGER when the user says "close current sprint tickets", "close sprint tickets", "close my sprint", or "wrap up sprint tickets". No abbreviation trigger.
---

# Close Sprint Tickets

> **First use is fully automated** — Step 0 detects MCP connection, installs if needed, and auto-fills all configuration values. No manual setup required.

Linear 8-step workflow for closing out all your tickets (any issue type) in the active sprint in your Jira Cloud instance. For each "Done" ticket, auto-logs a worklog equal to the current `remainingEstimate` so `Time Remaining` becomes 0 while `Time Spent` reflects the full estimate.

**Invocation phrase**: "close current sprint tickets" or natural variants ("close sprint tickets", "close my sprint", "wrap up sprint tickets"). No abbreviation trigger.

---

## User Configuration

> Auto-filled by Step 0 on first run. Manual editing only needed for troubleshooting. See `../jira-shared/INSTALL.md` for details.

| Setting | Value | Auto-discovered via |
|---|---|---|
| `JIRA_CLOUD_ID` | `<YOUR_CLOUD_ID>` | `getAccessibleAtlassianResources` |
| `JIRA_SITE_URL` | `<YOUR_SITE_URL>` | `getAccessibleAtlassianResources` |
| `JIRA_ACCOUNT_ID` | `<YOUR_ACCOUNT_ID>` | `atlassianUserInfo` |

---

## Deferred MCP tools

Load up-front via `ToolSearch select:<names>`:
- `mcp__atlassian__searchJiraIssuesUsingJql`
- `mcp__atlassian__getJiraIssue`
- `mcp__atlassian__getTransitionsForJiraIssue`
- `mcp__atlassian__transitionJiraIssue`
- `mcp__atlassian__addWorklogToJiraIssue`

---

## Step 0 — First-run check (auto — skip if User Configuration is already filled in)

Run this step automatically on every invocation. Skip individual sub-steps that are already satisfied.

### 0a. Test MCP connection
Try calling `getVisibleJiraProjects` or `atlassianUserInfo`.
- **Success** → proceed to 0b
- **Failure (tool not found / connection error)** → proceed to 0a-install

### 0a-install. Install Atlassian MCP
Ask the user: "Atlassian MCP is not connected. Would you like me to install it?"
On confirmation, run:
```bash
claude mcp add atlassian -- npx -y @anthropic-ai/mcp-remote@latest https://mcp.atlassian.com/v1/mcp/authv2
```
Tell the user: **"MCP has been added. Please restart Claude Code and re-run this skill."** Then **stop**.

### 0b. Verify MCP endpoint version
Check the current MCP configuration URL (read from `.claude/settings.json` or equivalent).
- If URL is the deprecated `https://mcp.atlassian.com/v1/sse`:
  - Inform user: "Your Atlassian MCP uses the old v1/sse endpoint, which stops working after June 30, 2026. Updating to v2."
  - Run: `claude mcp remove atlassian` then `claude mcp add atlassian -- npx -y @anthropic-ai/mcp-remote@latest https://mcp.atlassian.com/v1/mcp/authv2`
  - Tell user to restart Claude Code. **Stop**.
- If already on `v1/mcp/authv2` → continue

### 0c. Auto-discover parameters (if any placeholder remains)
If any `<YOUR_...>` placeholder still appears in the User Configuration table:
1. Call `atlassianUserInfo` → get `accountId` → fill `JIRA_ACCOUNT_ID`
2. Call `getAccessibleAtlassianResources` → get `cloudId` and site URL → fill `JIRA_CLOUD_ID` and `JIRA_SITE_URL`
3. Present all discovered values to the user for confirmation
4. Write values into the User Configuration table above (replace placeholders)

### 0d. Verify end-to-end
Run a test query: `assignee = currentUser() ORDER BY created DESC` (limit 1).
- **Success** → "Setup verified. Proceeding."
- **Failure** → display error, suggest checking MCP connection or Jira permissions

---

## Step 1 — Discover which projects have my open sprint tickets

Run a one-shot scan across all projects to see where you currently have open work:

```jql
assignee = currentUser() AND sprint in openSprints() AND statusCategory != Done
```

Request fields: `summary, status, issuetype, timetracking, sprint, project`.

Extract unique project keys + names from the results. Cache the full result set in memory — Steps 3-6 reuse it.

**If zero results** → tell user "No open tickets in your current sprint across any project." Stop.

---

## Step 2 — Ask user which project to scope to

Use `AskUserQuestion`:

> "Which project's sprint tickets do you want to close?"

- One option per unique project found (max 3, since the 4th slot is reserved for "All projects")
- Always include `All projects` as the final option

**Shortcut**: if only one project found, skip the question. Just inform the user: "Found N open tickets in `<Project>` — proceeding."

---

## Step 3 — Filter & present numbered list

Filter the Step 1 results to the chosen project (or keep all if `All projects`). Sort by `status` (Backlog → In Progress → In Review → others) then by `key` ascending. Present as numbered list:

```
#  | Key       | Type    | Summary                                  | Status      | Est / Rem
1  | PROJ-100  | Story   | Acme - Fix dashboard alignment           | Backlog     | 30m / 30m
2  | PROJ-101  | Task    | Acme - Update quarterly report template  | In Progress | 2h  / 1h
3  | PROJ-102  | Bug     | Acme - Review data pipeline output       | In Review   | 4h  / 4h
```

Format duration from `originalEstimateSeconds` / `remainingEstimateSeconds` as Jira strings: `30m`, `1h`, `1h 30m`, `1d`. If estimate is null, show `-`.

---

## Step 4 — Ask which to mark Done

Single free-text prompt to the user:

> "Which item numbers should be marked **Done**? (e.g. `1,3,5` or `all` or `none`)"

Parse:
- `all` → `done_set` = all items, `remaining_set` = []
- `none` → `done_set` = [], `remaining_set` = all items
- comma list (e.g. `1,3,5`) → split, validate each is within the displayed range, error out and re-ask on invalid input

---

## Step 5 — For each remaining ticket, ask target status

Skip this step entirely if `remaining_set` is empty.

Loop through `remaining_set`. For each ticket:

1. Call `getTransitionsForJiraIssue` to fetch valid transitions from the current status.
2. Build options from the response (max 4). Always include `Keep current status` as the first option (records `null` transition ID).
3. Ask via `AskUserQuestion` (use the ticket key as the `header`):

   > "What should happen to `<PROJ-NNN>` (currently `<status>`)?"

4. Record the chosen transition ID (or `null` for "Keep current status") against the ticket key.

**Optimization**: if the user explicitly said "keep all remaining as-is" in Step 4 follow-up text, skip the loop and treat every remaining ticket as `null` transition.

---

## Step 6 — Execute (Done batch first, then transitions batch)

Maintain a `failures` list. Tolerate per-ticket errors — log and continue.

### 6a — Done batch (per ticket in `done_set`)

1. Read `remainingEstimateSeconds` from the cached Step 1 data.
2. **If `remainingEstimateSeconds > 0`**:
   - Convert to Jira duration string (e.g. `5400` → `"1h 30m"`).
   - Call `addWorklogToJiraIssue` with `timeSpent = "<duration>"`. Jira auto-decrements `Remaining Estimate` to 0.
3. **If `remainingEstimateSeconds == 0`** → skip worklog (already fully logged).
4. **If `originalEstimate is null`** → skip worklog silently (no estimate was ever set; nothing to log to zero).
5. **Walk to Done** — many Jira workflows do not allow a direct transition to Done from every status. The skill auto-chains through intermediate states:
   - Call `getTransitionsForJiraIssue`. Find a transition whose target status's `statusCategory.key == "done"`.
   - If found → apply via `transitionJiraIssue`. Done.
   - If not found → look for any forward transition (a transition whose target status category differs from the current one and moves toward completion). Apply it, then re-call `getTransitionsForJiraIssue` and look for a `done` transition again.
   - Cap the auto-chain at **2 hops** to avoid infinite loops. If still not at Done after 2 hops, capture as a failure with a clear message ("<PROJ-NNN> needs manual transition: no Done path within 2 hops from `<status>`") and continue.

### 6b — Transitions batch (per ticket in `remaining_set`)

- If recorded transition ID is `null` → skip.
- Otherwise call `transitionJiraIssue` with the recorded ID.

---

## Step 7 — Verify & wrap up

Re-query to confirm the new state:

```jql
key in (<comma-separated list of all touched keys>)
```

Fields: `summary, status, timetracking`.

Present:

| # | Key | New Status | Time Spent | Remaining | Status |
|---|---|---|---|---|---|
| 1 | PROJ-100 | Done | 30m | 0m | OK |
| 2 | PROJ-101 | In Review | 1h | 1h | OK |

For any entry in `failures`, mark the Status column as FAILED and include the error message in a footnote.

**Final wrap-up**: one line, e.g. `Closed 3 tickets, transitioned 2 others. 0 failures.`

No follow-up offers — the skill ends here.

---

## Pitfalls

- Setting `remainingEstimate: 0` via `editJiraIssue` directly without a worklog wipes Remaining but does NOT increment `Time Spent`. Wrong outcome. **Always use `addWorklogToJiraIssue` instead.**
- Do not hardcode transition IDs to "Done" — workflows vary by project and instance. The path to Done may require multiple hops (e.g. Backlog → Next → In Progress → Review → Done). Always call `getTransitionsForJiraIssue` per ticket.
- Do not confuse `remainingEstimate == 0` (already fully logged) with `originalEstimate is null` (no estimate ever set). Both result in "skip worklog" but they're different states — log them differently in any debug output.
- Jira duration format: worklog `timeSpent` must be a string like `30m`, `1h`, `1h 30m`, `1d`. Don't pass raw seconds.
- Re-querying full ticket data between steps wastes API calls — cache the Step 1 result and reuse `remainingEstimateSeconds` from it for Step 6a.
- `All projects` is not a JQL filter value — just don't apply a `project = ...` clause when the user selects it.
