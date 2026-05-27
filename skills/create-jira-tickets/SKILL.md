---
name: create-jira-tickets
description: Create Jira tickets in any Jira Cloud instance following a 5-field quality rule (Description, Acceptance Criteria, Estimated hours, Due date, Job Code). TRIGGER when the user wants to "Create Jira Tickets", "create a Jira ticket/story/task", "log a Jira issue", or "raise a ticket". Handles project-key lookup (client name to Epic), AC drafting, time estimate confirmation, and field-ID resolution.
---

# Create Jira Tickets

> **First use is fully automated** — Step 0 detects MCP connection, installs if needed, and auto-fills all configuration values. No manual setup required.

Linear 7-step workflow for creating a Jira ticket in your Jira Cloud instance. Enforces the 5-field rule (Description, Acceptance Criteria, Estimated hours, Due date, Job Code).

**Invocation phrase**: "Create Jira Tickets" (or natural variants: "create a Jira ticket", "raise a ticket", "log this in Jira").

---

## User Configuration

> Auto-filled by Step 0 on first run. Manual editing only needed for troubleshooting. See `../../INSTALL.md` for details.

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

## Dynamic Lookup

All project, Epic, and Sprint selection is done via live Jira queries — no hardcoded mappings. This ensures the skill works for any user on any Jira instance without manual configuration.

### Projects

Call `getVisibleJiraProjects(cloudId)` → return the full list of projects the user can create issues in. Present as a numbered list for the user to pick from.

### Epics (Job Code)

After the user selects a project, query:

```jql
project = <SELECTED_KEY> AND issuetype = Epic AND statusCategory != Done ORDER BY updated DESC
```

Present all active Epics as a numbered list (Key + Summary) for the user to choose from. If no Epics exist in the selected project, inform the user and ask whether to proceed without a Job Code or pick a different project.

### Sprints

After the user selects a project, query:

```jql
project = <SELECTED_KEY> AND sprint in openSprints() ORDER BY created DESC
```

Extract the sprint name and ID from any returned issue's sprint field. Also always include `Backlog` as an option. Present the list for the user to pick from. If no open sprints exist, default to Backlog.

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
claude mcp add atlassian -- npx -y mcp-remote@latest https://mcp.atlassian.com/v1/mcp/authv2
```

Tell the user: **"MCP has been added. Please restart Claude Code and re-run this skill."** Then **stop**.

### 0b. Verify MCP endpoint version

Check the current MCP configuration URL (read from `.claude/settings.json` or equivalent).

- If URL is the deprecated `https://mcp.atlassian.com/v1/sse`:
  - Inform user: "Your Atlassian MCP uses the old v1/sse endpoint, which stops working after June 30, 2026. Updating to v2."
  - Run: `claude mcp remove atlassian` then `claude mcp add atlassian -- npx -y mcp-remote@latest https://mcp.atlassian.com/v1/mcp/authv2`
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

## Step 1 — Get context & dynamic selection

**If the user attached one or more images** (Teams chat screenshots, email screenshots, etc.) → read every image as visual input. Each image may represent a separate task request.

### Multi-image handling

- **Multiple images attached** → Read and parse ALL images, not just the first one. Each image that contains a distinct task request becomes its own ticket. Extract from each image: requester name, what's being asked, urgency cues ("when you have capacity" = no deadline; "by Friday" = explicit due date).
- **Single image with multiple requests** → If one screenshot contains multiple distinct task requests (e.g. a Teams chat with several asks), create a separate ticket for each request.
- **Single image, single request** → Standard single-ticket flow.

After parsing all images, present a numbered summary of all tickets to be created (one row per ticket) for user confirmation before proceeding to Step 1a. The user may remove, edit, or reorder tickets at this stage. Then run Step 1a–1d once (project, Epic, Sprint, Assignee apply to all tickets unless the user specifies otherwise per ticket).

### Language detection

Detect the language of the user's request (message text, image content, or inline context). All generated ticket content — Summary, Description (Background, Scope, Source Reference's "Original Request" paraphrase), Acceptance Criteria — must be written in the same language as the request. For example:

- User writes in English → all ticket fields in English
- User writes in Chinese → all ticket fields in Chinese
- Image contains English text → English ticket
- Mixed language → follow the dominant language of the request body

Fixed labels (section headings like "Source Reference", "Background", "Scope", "Acceptance Criteria", "Estimate", and field names like "Requester", "Date", "Source", "Original Request", "Original Estimate") always stay in English for consistency across Jira instances.

### 1a. Auto-query Project list

Call `getVisibleJiraProjects(cloudId)` immediately. Present all projects as a selection list:

```
Which project should this ticket go in?
1. PROJ — Project Hub
2. EPL — (Example) Product Launch
```

If only one project exists, auto-select it and inform the user.

### 1b. Auto-query Epic (Job Code) list

Once the project is selected, run:

```jql
project = <SELECTED_KEY> AND issuetype = Epic AND statusCategory != Done ORDER BY updated DESC
```

Present all active Epics:

```
Which Epic (Job Code) should this ticket be under?
1. PROJ-20 — Automation Toolkit
2. PROJ-18 — Mobile App
3. PROJ-4 — Admin
4. None — proceed without Epic
```

### 1c. Auto-query Sprint list

Run a JQL query to find open sprints in the selected project:

```jql
project = <SELECTED_KEY> AND sprint in openSprints() ORDER BY created DESC
```

Extract sprint info from returned issues. Present options:

```
Which Sprint?
1. <Sprint Name> (active)
2. Backlog
```

If no open sprints exist, default to Backlog and inform the user.

### 1d. Remaining context

**Always ask these (regardless of whether an image was provided)** — these have no safe default:

- **Assignee** — self or a specific person's name?
- **Reporter** — self or a specific person's name? (often the requester from the screenshot/email)

**If no image and no inline context** → also ask:

- What's the work?
- Any deadline?
- Any AC ideas you already have?

Batch all open questions into a single `AskUserQuestion` call. Don't proceed until you have enough to attempt a draft. One round of questions is usually enough.

**Resolving names to account IDs:** if Assignee/Reporter is anyone other than self, look up the account ID via `mcp__atlassian__lookupJiraAccountId` (load on demand) before Step 4. Do not invent IDs.

---

## Step 2 — Parse & auto-fill (single review table)

Use the project, Epic, and Sprint selected in Step 1 (from live Jira queries). Draft all fields and present in one table for user review:

| # | Field | Auto-filled value | Notes |
|---|---|---|---|
| 1 | **Description** | `<inferred from context, include requester quote if from image>` | rendered as ADF |
| 2 | **Acceptance Criteria** | `<4-6 bullet draft>` covering: deliverable, data correctness, styling/UX consistency, stakeholder sign-off | best-effort draft |
| 3 | **Estimated hours** | `<inferred from scope, or [TBD]>` | format: `2h`, `30m`, `1d`, `1w` |
| 4 | **Due date** | `<YYYY-MM-DD or "None">` | omit if user said "when you have capacity" |
| 5 | **Job Code (Epic)** | `<PROJ-NNN> <Epic name>` | from Dynamic Lookup (Step 1b) |
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
  "duedate": "<YYYY-MM-DD>",                // only if exists, else omit
  "reporter": { "accountId": "<reporter_account_id>" }  // only if Reporter != default; resolve via lookupJiraAccountId
  // "<SPRINT_FIELD>": <sprintId>           // only if user opted into a non-Backlog Sprint — pass as a NUMBER, not an array
}
```

1. **Time Tracking is mandatory** — Always include `"timetracking": {"originalEstimate": "<estimate>"}` in `additional_fields`. If `createJiraIssue` rejects the field (some project screens don't expose it at creation time), **immediately** follow up with `editJiraIssue` to set it. Never skip time tracking.

2. **Field fallback strategy** — If `createJiraIssue` rejects any field (AC, Epic Link, timetracking, etc.), do NOT abandon that field. Instead:
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

Every Description MUST follow this exact structure. The rendered result in Jira must look like this:

```
┌─────────────────────────────────────────────────────┐
│ ℹ️ (info panel — blue left border)                   │
│   Requester: <name>                                  │
│   Date: <YYYY-MM-DD>                                 │
│   Source: <where the request came from>              │
│   Original Request: <exact quote or paraphrase>     │
└─────────────────────────────────────────────────────┘
                                                       
  ───────────────── (horizontal rule) ─────────────────
                                                       
  Background                              ← H2 heading
  <1-2 sentences: WHY this work is needed.             
   Motivation/pain point only. NO deliverables.>       
                                                       
  ───────────────── (horizontal rule) ─────────────────
                                                       
  Scope                                   ← H2 heading
  1. **Feature A** — description of deliverable        
  2. **Feature B** — description of deliverable        
  3. **Feature C** — description of deliverable        
                                                       
  ───────────────── (horizontal rule) ─────────────────
                                                       
  Acceptance Criteria                     ← H2 heading
  ☐ First acceptance criterion                         
  ☐ Second acceptance criterion                        
  ☐ Third acceptance criterion                         
                                                       
  ───────────────── (horizontal rule) ─────────────────
                                                       
  Estimate                                ← H2 heading
  Original Estimate: <value>                           
```

### Section-by-section rules

1. **Source Reference** — `panel` node with `panelType: "info"`. Contains ONE paragraph with 4 lines separated by `hardBreak` nodes. The 4 lines are always:
   - `Requester: <name>`
   - `Date: <YYYY-MM-DD>`
   - `Source: <origin — e.g. Teams chat, email, task list, Slack message>`
   - `Original Request: <exact quote or close paraphrase>`

2. **Background** — `heading` (level 2) + single `paragraph`. Write only WHY (motivation, pain point). Do NOT describe deliverables — that belongs in Scope.

3. **Scope** — `heading` (level 2) + `orderedList` with `listItem` nodes. Each item: **bold feature name** followed by ` — ` (space-em-dash-space) then description. Use `strong` mark on the feature name text node only.

4. **Acceptance Criteria** — `heading` (level 2) + `taskList` with `taskItem` nodes (renders as checkboxes ☐). Each `taskItem` has `attrs: {"localId": "<uuid>", "state": "TODO"}`. If the MCP tool rejects `taskList`/`taskItem` with `INVALID_INPUT`, fallback to `bulletList`/`listItem` and inform the user to convert in Jira UI.

5. **Estimate** — `heading` (level 2) + single `paragraph`: `Original Estimate: <value>`.

6. **Horizontal rules** — Insert a `rule` node between every section (after Source Reference, after Background, after Scope, after Acceptance Criteria). Total of 4 `rule` nodes.

### Complete ADF example

This is a full, copy-paste-ready ADF document. Replace placeholder values but preserve the exact node structure:

```json
{
  "type": "doc",
  "version": 1,
  "content": [
    {
      "type": "panel",
      "attrs": { "panelType": "info" },
      "content": [
        {
          "type": "paragraph",
          "content": [
            { "type": "text", "text": "Requester: Sarah Chen" },
            { "type": "hardBreak" },
            { "type": "text", "text": "Date: 2026-05-27" },
            { "type": "hardBreak" },
            { "type": "text", "text": "Source: Teams chat" },
            { "type": "hardBreak" },
            { "type": "text", "text": "Original Request: Auto-delete promotional emails that haven't been opened for 7 days" }
          ]
        }
      ]
    },
    { "type": "rule" },
    {
      "type": "heading",
      "attrs": { "level": 2 },
      "content": [{ "type": "text", "text": "Background" }]
    },
    {
      "type": "paragraph",
      "content": [
        {
          "type": "text",
          "text": "The inbox accumulates a large volume of advertising and promotional emails. Manual cleanup is time-consuming. An automated rule is needed to identify long-unopened promotional emails and clean them up automatically, keeping the inbox organized."
        }
      ]
    },
    { "type": "rule" },
    {
      "type": "heading",
      "attrs": { "level": 2 },
      "content": [{ "type": "text", "text": "Scope" }]
    },
    {
      "type": "orderedList",
      "attrs": { "order": 1 },
      "content": [
        {
          "type": "listItem",
          "content": [
            {
              "type": "paragraph",
              "content": [
                { "type": "text", "text": "Promotional email detection", "marks": [{ "type": "strong" }] },
                { "type": "text", "text": " — Identify advertising/promotional emails based on labels, sender, and subject keywords" }
              ]
            }
          ]
        },
        {
          "type": "listItem",
          "content": [
            {
              "type": "paragraph",
              "content": [
                { "type": "text", "text": "Open status tracking", "marks": [{ "type": "strong" }] },
                { "type": "text", "text": " — Track email open status and flag promotional emails unopened for more than 7 days" }
              ]
            }
          ]
        },
        {
          "type": "listItem",
          "content": [
            {
              "type": "paragraph",
              "content": [
                { "type": "text", "text": "Auto-cleanup execution", "marks": [{ "type": "strong" }] },
                { "type": "text", "text": " — Automatically move qualifying emails to trash or delete them directly" }
              ]
            }
          ]
        },
        {
          "type": "listItem",
          "content": [
            {
              "type": "paragraph",
              "content": [
                { "type": "text", "text": "Whitelist protection", "marks": [{ "type": "strong" }] },
                { "type": "text", "text": " — Provide a whitelist mechanism to prevent accidental deletion of important emails" }
              ]
            }
          ]
        }
      ]
    },
    { "type": "rule" },
    {
      "type": "heading",
      "attrs": { "level": 2 },
      "content": [{ "type": "text", "text": "Acceptance Criteria" }]
    },
    {
      "type": "taskList",
      "attrs": { "localId": "ac-list-001" },
      "content": [
        {
          "type": "taskItem",
          "attrs": { "localId": "ac-001", "state": "TODO" },
          "content": [{ "type": "text", "text": "Identify advertising/promotional emails based on labels, sender, and subject keywords" }]
        },
        {
          "type": "taskItem",
          "attrs": { "localId": "ac-002", "state": "TODO" },
          "content": [{ "type": "text", "text": "Track email open status and flag emails unopened for more than 7 days" }]
        },
        {
          "type": "taskItem",
          "attrs": { "localId": "ac-003", "state": "TODO" },
          "content": [{ "type": "text", "text": "Automatically move qualifying emails to trash or delete them directly" }]
        },
        {
          "type": "taskItem",
          "attrs": { "localId": "ac-004", "state": "TODO" },
          "content": [{ "type": "text", "text": "Provide a whitelist mechanism to prevent accidental deletion of important emails" }]
        },
        {
          "type": "taskItem",
          "attrs": { "localId": "ac-005", "state": "TODO" },
          "content": [{ "type": "text", "text": "Generate a cleanup report summary" }]
        }
      ]
    },
    { "type": "rule" },
    {
      "type": "heading",
      "attrs": { "level": 2 },
      "content": [{ "type": "text", "text": "Estimate" }]
    },
    {
      "type": "paragraph",
      "content": [{ "type": "text", "text": "Original Estimate: 1d" }]
    }
  ]
}
```

**When generating a new ticket, copy this exact ADF structure. Only replace the text values — never change the node types, nesting, or order.**

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
