# Jira Skills — Installation & Setup

Both `create-jira-tickets` and `close-sprint-tickets` share the same Atlassian MCP dependency. This guide documents the automated setup flow that runs on first use of either skill.

---

## Step 0 — Test MCP Connection

Try calling any Atlassian MCP tool (e.g. `getVisibleJiraProjects` or `atlassianUserInfo`).

- **Success** → proceed to Step 1b (version check)
- **Failure** → proceed to Step 1 (install)

---

## Step 1 — Install Atlassian MCP (only if Step 0 failed)

Ask the user: "Atlassian MCP is not connected. Would you like me to install it?"

On confirmation, run:

```bash
claude mcp add atlassian -- npx -y @anthropic-ai/mcp-remote@latest https://mcp.atlassian.com/v1/mcp/authv2
```

Then tell the user: **"MCP has been added. Please restart Claude Code and re-run this skill."**

Stop here — the skill will re-enter Step 0 on next invocation after restart.

---

## Step 1b — Verify MCP Endpoint Version (runs even if Step 0 passed)

Check the current MCP configuration URL. The correct endpoint is:

```
https://mcp.atlassian.com/v1/mcp/authv2
```

If the URL is the deprecated `https://mcp.atlassian.com/v1/sse`:

1. Inform the user: "Your Atlassian MCP uses the old v1/sse endpoint, which will stop working after June 30, 2026. Updating to v2."
2. Run:
   ```bash
   claude mcp remove atlassian
   claude mcp add atlassian -- npx -y @anthropic-ai/mcp-remote@latest https://mcp.atlassian.com/v1/mcp/authv2
   ```
3. Remind user to restart Claude Code.

If already on v2 → continue.

---

## Step 2 — Auto-discover Parameters & Fill Configuration

All values are discovered automatically via MCP API calls:

| Parameter | How to discover | Used by |
|---|---|---|
| `JIRA_CLOUD_ID` | `getAccessibleAtlassianResources` → `id` | Both skills |
| `JIRA_SITE_URL` | `getAccessibleAtlassianResources` → `url` | Both skills |
| `JIRA_ACCOUNT_ID` | `atlassianUserInfo` → `accountId` | Both skills |
| `AC_CUSTOM_FIELD` | `getJiraIssueTypeMetaWithFields` → search for "Acceptance Criteria" | create-jira-tickets only |
| `EPIC_LINK_FIELD` | `getJiraIssueTypeMetaWithFields` → search for "Epic Link" | create-jira-tickets only |
| `SPRINT_FIELD` | `getJiraIssueTypeMetaWithFields` → search for "Sprint" | create-jira-tickets only |

After discovery, the skill automatically writes these values into its own SKILL.md User Configuration table, replacing any `<YOUR_...>` placeholders.

Present all discovered values to the user for confirmation before writing.

---

## Step 3 — Verify End-to-End

Run a test query to confirm everything works:

```jql
assignee = currentUser() ORDER BY created DESC
```

- **Success** → "Setup complete. You can now use the skill."
- **Failure** → display error and suggest checking MCP connection or Jira permissions.

---

## Manual Troubleshooting

If the automated flow fails:

1. **MCP not found after install**: Make sure you restarted Claude Code after `claude mcp add`.
2. **Auth errors**: The first time you connect, Atlassian will prompt for OAuth consent in your browser. Make sure to complete the authorization flow.
3. **Wrong Cloud ID**: If you have multiple Atlassian sites, `getAccessibleAtlassianResources` returns all of them. Pick the correct one.
4. **Custom fields not found**: Field IDs vary by project and Jira instance. The auto-discovery searches the Story issue type. If your project uses different issue types, run discovery manually with the correct `issueTypeId`.
