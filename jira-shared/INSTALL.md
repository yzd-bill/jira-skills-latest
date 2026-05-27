# Atlassian MCP — Pre-installation Guide

> **Run this guide BEFORE installing any Jira skills.** Setting up the Atlassian MCP connection first ensures the skills work immediately on first use — no restarts needed mid-workflow.

---

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) installed and working (CLI, Desktop App, or IDE extension)
- A Jira Cloud instance (e.g. `https://yoursite.atlassian.net`)
- An Atlassian account with permissions to create/edit issues

---

## Choose Your Installation Method

There are two ways to connect Claude Code to your Atlassian instance. Pick one — they are functionally equivalent.

| Method | Best for | Requires |
|--------|----------|----------|
| **Method 1: Connector (Recommended)** | All users — visual, no terminal commands needed | Claude Code Desktop App or IDE extension |
| **Method 2: CLI** | Developers who prefer terminal commands | Node.js 18+ |

---

## Method 1: Connector Installation (Recommended)

### 1.1. Open the Claude Code Directory

In the Claude Code Desktop App or IDE extension, open the **Directory** panel:

- **Desktop App**: Click the Directory icon in the sidebar (or use the menu)
- **IDE extension**: Open the Claude Code panel → click Directory

### 1.2. Search for the Atlassian connector

1. In the Directory, click the **Connectors** tab on the left sidebar
2. Type **`jira`** in the search bar
3. You will see **Atlassian Rovo** — "Access Jira & Confluence from Claude"

### 1.3. Install the connector

1. Click the **+** button on the **Atlassian Rovo** card
2. Follow the on-screen prompts to complete the installation
3. When prompted, authorize with your Atlassian account:
   - Browser opens automatically → Atlassian login page
   - Log in with your Atlassian account (if not already logged in)
   - Review the permissions requested → click **Accept**
   - Browser shows "Authorization successful" → return to Claude Code

### 1.4. Restart Claude Code

Close and reopen Claude Code (or restart the session). This is required for the new MCP server to be loaded.

### 1.5. Verify the connection

In Claude Code, ask:

```
"List my Jira projects"
```

Claude should return your project list with keys and names. If you see your projects, the connection is working. Proceed to the **Verification Checklist** below.

---

## Method 2: CLI Installation

### 2.1. Add the Atlassian MCP server

Open your terminal and run:

```bash
claude mcp add atlassian -- npx -y mcp-remote@latest https://mcp.atlassian.com/v1/mcp/authv2
```

This registers the Atlassian MCP server with Claude Code using the latest v2 authentication endpoint.

> **Note:** This method requires Node.js 18+. Run `node --version` to check. Install from [nodejs.org](https://nodejs.org/) if needed.

### 2.2. Restart Claude Code

Close and reopen Claude Code (or restart the CLI session). This is required for the new MCP server to be loaded.

```bash
# If using CLI, simply exit and re-launch:
exit
claude
```

### 2.3. Authorize with Atlassian

On the first MCP tool call after restart, your browser will open an Atlassian OAuth consent page. Complete the authorization flow:

1. Browser opens automatically → Atlassian login page
2. Log in with your Atlassian account (if not already logged in)
3. Review the permissions requested → click **Accept**
4. Browser shows "Authorization successful" → return to Claude Code

If the browser does not open automatically, check the terminal output for a URL and open it manually.

### 2.4. Verify the connection

In Claude Code, ask:

```
"List my Jira projects"
```

Claude should return your project list with keys and names. If you see your projects, the connection is working.

---

## Verification Checklist

After installation (either method), run through this checklist to confirm everything works. Ask Claude to perform each step:

### 1. MCP server is registered

```bash
claude mcp list
```

You should see `atlassian` in the output.

### 2. OAuth is authorized

Ask Claude:

```
"Show my Atlassian user info"
```

Claude calls `atlassianUserInfo` and returns your display name and account ID. If you get an auth error, re-run the OAuth flow (remove and re-add the MCP server).

### 3. Jira projects are accessible

Ask Claude:

```
"List my Jira projects"
```

Claude calls `getVisibleJiraProjects` and returns your project list with keys and names.

### 4. Issues are queryable

Ask Claude:

```
"Show my most recent Jira ticket"
```

Claude runs `assignee = currentUser() ORDER BY created DESC` (limit 1) and returns a ticket. This confirms read access to your Jira issues.

### 5. (Optional) Write access test

If you want to confirm write access before using the skills:

Ask Claude:

```
"Create a test ticket in <YOUR_PROJECT_KEY> with summary 'MCP connection test'"
```

This confirms `createJiraIssue` works. You can delete the test ticket manually in the Jira UI afterwards.

---

## Post-installation: Install Jira Skills

Once all verification checks pass, install the Jira skills. No further restarts are needed.

### Option 1: Install from GitHub

```bash
claude install-skill https://github.com/<owner>/<repo>/tree/main/skills/create-jira-tickets
claude install-skill https://github.com/<owner>/<repo>/tree/main/skills/close-sprint-tickets
```

### Option 2: Install from local directory

```bash
claude install-skill ./skills/create-jira-tickets
claude install-skill ./skills/close-sprint-tickets
```

The skills will auto-detect the existing MCP connection on first use and skip the MCP installation step entirely.

---

## Upgrading from v1 (Legacy Endpoint)

If you previously installed the Atlassian MCP with the old `v1/sse` endpoint:

```
https://mcp.atlassian.com/v1/sse    <- deprecated, stops working after June 30, 2026
```

Upgrade to v2:

```bash
claude mcp remove atlassian
claude mcp add atlassian -- npx -y mcp-remote@latest https://mcp.atlassian.com/v1/mcp/authv2
```

Then restart Claude Code. Your existing OAuth authorization carries over — no need to re-authorize.

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| `MCP tool not found` after install | Restart Claude Code — MCP servers are loaded at startup |
| Browser does not open for OAuth | Check terminal output for a URL and open it manually |
| OAuth consent page shows error | Ensure your Atlassian account has admin or project-level permissions |
| `getVisibleJiraProjects` returns empty | Your account may not have "Create Issues" permission — check Jira project settings |
| `401 Unauthorized` errors | OAuth token may have expired — run `claude mcp remove atlassian` then re-add and re-authorize |
| Multiple Atlassian sites | `getAccessibleAtlassianResources` returns all sites — the skills will ask you to pick the correct one on first use |
| Node.js version error (CLI only) | MCP remote proxy requires Node.js 18+. Run `node --version` to check |
| `npx` not found (CLI only) | Install Node.js from [nodejs.org](https://nodejs.org/) or via your package manager |
| Connector not visible in Directory | Ensure you are using Claude Code Desktop App or IDE extension — the Directory panel is not available in the CLI |
