---
name: create-jira-tickets
description: Create Jira tickets in any Jira Cloud instance following a 5-field quality rule (Description, Acceptance Criteria, Estimated hours, Due date, Job Code). TRIGGER when the user wants to "Create Jira Tickets", "create a Jira ticket/story/task", "log a Jira issue", or "raise a ticket". Handles project-key lookup (client name to Epic), AC drafting, time estimate confirmation, and field-ID resolution.
---

# Create Jira Tickets

> **First use is fully automated** — Step 0 detects MCP connection, installs if needed, and auto-fills all configuration values. No manual setup required.

Linear 7-step workflow for creating a Jira ticket in your Jira Cloud instance. Enforces the 5-field rule (Description, Acceptance Criteria, Estimated hours, Due date, Job Code).

**Invocation phrase**: "Create Jira Tickets" or "CJT" (or natural variants: "create a Jira ticket", "raise a ticket", "log this in Jira").

---

## User Configuration

> Auto-filled by Step 0 on first run. Manual editing only needed for troubleshooting. See `../jira-shared/INSTALL.md` for details.

| Setting | Value | Auto-discovered via |
|---|---|---|
| `JIRA_CLOUD_ID` | `<YOUR_CLOUD_ID>` | `getAccessibleAtlassianResources` |
| `JIRA_SITE_URL` | `<YOUR_SITE_URL>` | `getAccessibleAtlassianResources` |
| `JIRA_ACCOUNT_ID` | `<YOUR_ACCOUNT_ID>` | `atlassianUserInfo` |
| `AC_CUSTOM_FIELD` | `<YOUR_AC_FIELD>` | `getJiraIssueTypeMetaWithFields` |
| `EPIC_LINK_FIELD` | `<YOUR_EPIC_LINK_FIELD>` | `getJiraIssueTypeMetaWithFields` |
| `SPRINT_FIELD` | `<YOUR_SPRINT_FIELD>` | `getJiraIssueTypeMetaWithFields` |

---

## Deferred MCP tools

Load up-front via `ToolSearch select:<names>`:
- `mcp__atlassian__createJiraIssue`
- `mcp__atlassian__getJiraIssue`
- `mcp__atlassian__searchJiraIssuesUsingJql`
- `mcp__atlassian__getJiraIssueTypeMetaWithFields`
- `mcp__atlassian__getTransitionsForJiraIssue`, `mcp__atlassian__transitionJiraIssue` (only if Step 6 follow-ups are accepted)
- `mcp__atlassian__lookupJiraAccountId` (only if Reporter/Assignee is someone other than self)

---

## Mappings

Edit this table to match your projects and clients. The left side maps project names to JQL keys; the right side maps client shorthand names to their parent Epic (Job Code).

| Project Name | JQL Key |  | Client shorthand | Epic Key |
|---|---|---|---|---|
| _Example Project_ | `PROJ` | | _Apple_ | `PROJ-100` |

If a client/project is not in this table, search live:
```jql
project = <KEY> AND issuetype = Epic AND statusCategory != Done AND text ~ "<client>"
```

To populate this table, ask Claude to run:
```jql
project = <YOUR_PROJECT_KEY> AND issuetype = Epic AND statusCategory != Done
```

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
3. Ask the user which project key to use (e.g. `KAN`)
4. Call `getJiraProjectIssueTypesMetadata` to find Story issue type ID (do NOT assume defaults)
5. Call `getJiraIssueTypeMetaWithFields` → search for:
   - "Acceptance Criteria" → fill `AC_CUSTOM_FIELD`
   - "Epic Link" → fill `EPIC_LINK_FIELD`
   - "Sprint" → fill `SPRINT_FIELD`
6. Present all discovered values to the user for confirmation
7. Write values into the User Configuration table above (replace placeholders)
8. If a field is not found (e.g. no AC custom field), note it and skip in subsequent steps

### 0d. Verify end-to-end
Run a test query: `assignee = currentUser() ORDER BY created DESC` (limit 1).
- **Success** → "Setup verified. Proceeding to create ticket."
- **Failure** → display error, suggest checking MCP connection or Jira permissions

---

## Step 1 — Get context

**If the user attached an image** (Teams chat screenshot, email screenshot, etc.) → read it as visual input. Extract: requester name, what's being asked, urgency cues ("when you have capacity" = no deadline; "by Friday" = explicit due date).

**Always ask these in Step 1 (regardless of whether an image was provided)** — these have no safe default:
- **Sprint** — `Current sprint` / `Backlog` / `<specific sprint name>`?
- **Assignee** — self or a specific person's name?
- **Reporter** — self or a specific person's name? (often the requester from the screenshot/email)

**If no image and no inline context** → also ask:
- Which client / project?
- What's the work?
- Any deadline?
- Any AC ideas you already have?

Batch all open questions into a single `AskUserQuestion` call. Don't proceed until you have enough to attempt a draft. One round of questions is usually enough.

**Resolving names to account IDs:** if Assignee/Reporter is anyone other than self, look up the account ID via `mcp__atlassian__lookupJiraAccountId` (load on demand) before Step 4. Do not invent IDs.

---

## Step 2 — Parse & auto-fill (single review table)

Resolve project key + Job Code from context (use the Mappings table above, or search live). Draft all fields and present in one table for user review:

| # | Field | Auto-filled value | Notes |
|---|---|---|---|
| 1 | **Description** | `<inferred from context, include requester quote if from image>` | rendered as ADF |
| 2 | **Acceptance Criteria** | `<4-6 bullet draft>` covering: deliverable, data correctness, styling/UX consistency, stakeholder sign-off | best-effort draft |
| 3 | **Estimated hours** | `<inferred from scope, or [TBD]>` | format: `2h`, `30m`, `1d`, `1w` |
| 4 | **Due date** | `<YYYY-MM-DD or "None">` | omit if user said "when you have capacity" |
| 5 | **Job Code (Epic)** | `<PROJ-NNN> <Client>` | from Mappings table or JQL search |
| 6 | **Sprint** | `<from Step 1 answer>` | Backlog / Current sprint / specific name — never auto-default |
| 7 | **Assignee** | `<from Step 1 answer>` | self or named person |
| 8 | **Reporter** | `<from Step 1 answer>` | self or named person (often requester) |

Also auto-fill these defaults (don't ask):
- Issue type: `Story` (most common; switch to `Task` / `Bug` only if context demands)
- Summary: `<Client> - <action> <noun>` (e.g. "Acme - Add Dashboard Filter to Monthly Report")
- Priority: `Undefined` (default)

Mark uncertain values with `[TBD]`. Then ask user to confirm or edit. **Do not invent** missing values — surface them as `[TBD]`.

**IMPORTANT — AC must be shown in full:** The review table only summarizes AC as a count (e.g. "10 items"). You MUST also list every AC bullet below the table so the user can review the exact wording before creation. Never hide AC behind a count — always show the complete list.

---

## Step 3 — Plan-mode preview

If currently in plan mode → write the draft (Step 2 table + summary + issue type) to the plan file, then `ExitPlanMode` for approval.

If NOT in plan mode → just print the table, the full AC list, and wait for the user to say "go" or send edits.

---

## Step 4 — Create

Once user confirms:

1. **Always run field discovery first** — call `getJiraIssueTypeMetaWithFields(cloudId=JIRA_CLOUD_ID, projectIdOrKey, issueTypeId)` to detect which fields are available on this project's screen. Different projects (Classic vs Team-managed) expose different fields. Also use `getJiraProjectIssueTypesMetadata` to find the correct issue type IDs — do NOT assume defaults (e.g. Story may be `10001` in one instance but `10009` in another).

2. **Call `createJiraIssue`** with ADF-formatted description (see Description Format Rules below):

```
cloudId: "<JIRA_CLOUD_ID>"
projectKey: "<project key>"
issueTypeName: "Story"
summary: "<concise summary>"
description: <ADF document — see Description Format Rules>
contentFormat: "adf"
assignee_account_id: "<JIRA_ACCOUNT_ID>"
additional_fields: {
  "<EPIC_LINK_FIELD>": "<PROJ-NNN>",       // Epic Link (Job Code) — for Classic projects
  // OR use top-level "parent": "<PROJ-NNN>" for Team-managed projects
  "<AC_CUSTOM_FIELD>": {                    // Acceptance Criteria (ADF doc) — try taskList first, fallback to bulletList
    "type": "doc", "version": 1,
    "content": [{
      "type": "taskList", "attrs": {"localId": "<uuid>"},
      "content": [
        { "type": "taskItem", "attrs": {"localId": "<uuid>", "state": "TODO"}, "content": [{ "type": "text", "text": "<AC item>" }]}
      ]
    }]
  },
  "timetracking": { "originalEstimate": "2h" },
  "duedate": "2026-05-30",                 // only if exists, else omit
  "reporter": { "accountId": "<reporter_account_id>" }  // only if Reporter != default; resolve via lookupJiraAccountId
  // "<SPRINT_FIELD>": <sprintId>           // only if user opted into a non-Backlog Sprint — pass as a NUMBER, not an array
}
```

3. **Time Tracking is mandatory** — Always include `"timetracking": {"originalEstimate": "<estimate>"}` in `additional_fields`. If `createJiraIssue` rejects the field (some project screens don't expose it at creation time), **immediately** follow up with `editJiraIssue` to set it. Never skip time tracking.

4. **Field fallback strategy** — If `createJiraIssue` rejects any field (AC, Epic Link, timetracking, etc.), do NOT abandon that field. Instead:
   - Try setting it via `editJiraIssue` after creation
   - For AC: if no custom field exists, include AC content in the Description body
   - For Epic Link: use `parent` field for Team-managed projects instead of `EPIC_LINK_FIELD`

**If user chose a Sprint** in Step 2 (not Backlog), look up the active sprint ID first:
```jql
project = <KEY> AND sprint in openSprints()
```
Take any returned issue's sprint field `[0].id` → use as the new sprint ID. (Alternatively read board state.)

---

### Description Format Rules

**CRITICAL: Always use `contentFormat: "adf"` (Atlassian Document Format), NOT `"markdown"`.** Markdown mode with `\n` literals renders as plain text in Jira, destroying readability.

Every Description MUST follow this structure using ADF nodes:

1. **Source Reference** (info panel) — Who requested it, when, where, and original request quote:
   - `Requester:` name and role/relationship
   - `Date:` YYYY-MM-DD
   - `Source:` where the request came from (Teams chat, email, screenshot, meeting, etc.)
   - `Original Request:` exact quote or paraphrase in quotes

2. **Background** (paragraph) — 1-2 sentence explaining WHY this work is needed (motivation, pain point). Do NOT describe deliverables here — that belongs in Scope.

3. **Scope** (ordered list) — Each deliverable as a numbered list item: `**Feature name** — description`. Use `orderedList`/`listItem` ADF nodes with bold text for the feature name followed by an em dash and description.

4. **Acceptance Criteria** (task list) — Use `taskList`/`taskItem` ADF nodes to render as checkable checkboxes. If the MCP tool rejects `taskList`/`taskItem` with `INVALID_INPUT`, automatically fallback to `bulletList`/`listItem` and inform the user they can convert to checkboxes in Jira UI (select list → click ☑ toolbar button).

5. **Estimate** (paragraph) — `Original Estimate: <value>`

ADF skeleton:
```json
{
  "type": "doc", "version": 1,
  "content": [
    { "type": "panel", "attrs": {"panelType": "info"}, "content": [/* Source Reference */] },
    { "type": "rule" },
    { "type": "heading", "attrs": {"level": 2}, "content": [{"type": "text", "text": "Background"}] },
    { "type": "paragraph", "content": [/* ... */] },
    { "type": "rule" },
    { "type": "heading", "attrs": {"level": 2}, "content": [{"type": "text", "text": "Scope"}] },
    { "type": "orderedList", "content": [/* listItem: bold feature — description */] },
    { "type": "rule" },
    { "type": "heading", "attrs": {"level": 2}, "content": [{"type": "text", "text": "Acceptance Criteria"}] },
    { "type": "taskList", "attrs": {"localId": "<uuid>"}, "content": [/* taskItem nodes with state="TODO" */] },
    { "type": "rule" },
    { "type": "heading", "attrs": {"level": 2}, "content": [{"type": "text", "text": "Estimate"}] },
    { "type": "paragraph", "content": [/* Original Estimate: Xh/Xd/Xw */] }
  ]
}
```

---

## Step 5 — Verify (8-row table)

Read back via `getJiraIssue` with explicit `fields` list. Present:

| Field | Value | Status |
|---|---|---|
| Description | `<truncated>` | OK |
| Acceptance Criteria | N bullets | OK |
| Estimated hours | `2h` | OK |
| Due date | `<date or None>` | OK |
| Job Code | `<PROJ-NNN> (<Client>)` | OK |
| Sprint | `<Sprint Name>` | OK |
| Assignee | `<displayName>` | OK |
| Reporter | `<displayName>` | OK |

Return: ticket key + browse URL `https://<JIRA_SITE_URL>/browse/<KEY>`.

---

## Step 6 — Follow-up offers

Proactively offer (don't execute without "yes"):

- **Pull into current sprint** — set sprint field via edit
- **Transition to In Progress** — look up available transitions via `getTransitionsForJiraIssue` and walk forward through the workflow (some instances require multiple hops, e.g. Backlog → Next → In Progress). Cap at 2 hops.
- **Add comment** — e.g. `@<requester>` confirming receipt

---

## Pitfalls

- Do not confuse project name prefixes with project keys — always verify the actual JQL key (e.g. a project called "Data Analytics" might have key `DA`, not `DATA`).
- For Classic Jira projects, use the Epic Link custom field (`EPIC_LINK_FIELD`) for Job Code, NOT the top-level `parent` field. The `parent` field auto-populates server-side.
- For Team-managed (next-gen) projects, use the `parent` field instead of `EPIC_LINK_FIELD` — the custom field may not exist on the screen.
- **The `parent` field must be passed via `additional_fields`, not as a top-level parameter.** The top-level `parent` parameter in `createJiraIssue` only works for Subtask types. To link a Story/Task to an Epic, pass it as an object in `additional_fields`: `{"parent": {"key": "PROJ-100"}}`, NOT as top-level `parent: "PROJ-100"`. Using the wrong approach causes `INVALID_INPUT`.
- Pass the Sprint field as a single number, NOT an array. Use `8564` not `[8564]`.
- Never invent values for missing fields — if you can't infer, mark `[TBD]` and ask the user.
- **Never use `contentFormat: "markdown"` with escaped `\n` strings.** Jira renders them as literal `\n` text, destroying readability. Always use `contentFormat: "adf"` with a proper ADF JSON document.
- **Issue type IDs are not universal.** Story could be `10001` in one instance and `10009` in another. Always discover via `getJiraProjectIssueTypesMetadata` before calling `getJiraIssueTypeMetaWithFields`.
- **Custom field IDs vary across projects** even within the same Jira instance. The User Configuration field IDs (AC, Epic Link, Sprint) may only work for specific projects. Always run field discovery (`getJiraIssueTypeMetaWithFields`) for new/unfamiliar projects.
- **Time tracking must not be skipped.** If `createJiraIssue` rejects `timetracking`, immediately use `editJiraIssue` to set it. The estimate is one of the 5 required fields — it must exist in Jira's time tracking, not just in the Description text.
- **taskList/taskItem ADF nodes may be rejected** by the Atlassian MCP tool with `INVALID_INPUT`. Always try `taskList`/`taskItem` first for checkbox-style AC. If rejected, automatically fallback to `bulletList`/`listItem` and inform the user to convert in Jira UI (select list → click ☑ toolbar button).
