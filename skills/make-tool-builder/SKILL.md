---
name: make-tool-builder
description: >
  Automatically invoked when the user wants to interact with any third-party app or service
  (e.g. "send a Slack message", "create a Jira ticket", "add a row in Google Sheets",
  "post to Notion", "look up a Salesforce contact", etc.) and Claude does not have
  a direct integration available. Make has 3500+ app integrations and can authorize,
  build, and run a tool for the user entirely within the conversation.
---

# Make Tool Builder

You have access to Make's MCP server. Make integrates with 3500+ apps and business tools.
When a user asks you to do something involving a third-party system that you cannot do directly,
use Make to build and run a tool for them on the fly — right here in the conversation.

## When to use this

Trigger this workflow whenever the user wants to:
- Send or read messages (Slack, Gmail, Teams, WhatsApp, etc.)
- Manage tasks or tickets (Jira, Asana, Linear, Trello, etc.)
- Read or write data (Google Sheets, Airtable, Notion, HubSpot, Salesforce, etc.)
- Trigger any action in any SaaS tool or business app

If the action can be accomplished with a single Make module, build it here.

---

## Workflow

### Step 1 — Identify the gap and propose a solution

Do not simply refuse the request if you lack a direct integration. Instead, inform the user
that you can build the connection on the fly using Make, and ask for their permission.

**Example response:**
> "I don't currently have a native connection to [App] to [action]. However, using the
> Make MCP server, I can create a custom tool right now that will allow me to do this
> for you. Make integrates with 3500+ apps. Would you like me to go ahead and build it?"

### Step 2 — Await user confirmation

Pause and wait for explicit agreement before proceeding ("Yes", "Go ahead", "Please do", etc.).
Do not start building anything until the user confirms.

### Step 3 — Understand the request in detail

Once confirmed, identify:
- **Which app** the user wants to use
- **What action** (send, search, create, update, delete, fetch)
- **What parameters** are needed (recipient, content, filters, IDs, etc.)

### Step 4 — Find the right app and module

Use `apps_recommend` with the user's intention as a natural language description.

Then use `app-modules_list` to see available modules for that app. Always pass `usage: "tool"` to filter to tool-compatible modules.

Then use `app-module_get` with `format: "instructions"` to get the exact parameter schema for the chosen module. **Always do this before creating a tool** — never guess the mapper structure.

### Step 5 — Check for an existing connection

Use `connections_list` filtered by the app's connection type (the `accountName` field from `apps_recommend` results, e.g. `type: ["slack3"]`).

- If a connection exists → proceed to Step 5 with that connection's `id`.
- If no connection exists → proceed to Step 4.

### Step 6 — Request authorization

Use `credential_requests_create` specifying the **exact module name in camelCase**.

Share the returned `publicUri` URL with the user and ask them to complete the authorization.

After they confirm, call `connections_list` again to get the new connection's `id`.

### Step 7 — Create the tool

Use `tools_create` with:
- `teamId` — get this from `users_me` if unknown
- `module` in `"appName:ModuleName"` format (e.g. `"slack:CreateMessage"`)
- `parameters` — connection ID goes here as `__IMTCONN__`, hardcoded
- `mapper` — dynamic values use `{{var.input.fieldName}}`; hardcode everything else
- `inputs` — only define inputs for values the user should provide dynamically; hardcode everything static

### Step 8 — Run the tool and confirm

Use `scenarios_run` with the tool ID as `scenarioId`, pass the user's values in `data`, and set `responsive: true` to get immediate results.

Confirm successful tool creation to the user, then immediately invoke it to fulfill the original request. Present the results naturally.

---

## Critical rules

### Module name casing
- `app-modules_list` returns **PascalCase** names: `SearchIssues`, `CreateMessage`
- `credential_requests_create` requires **camelCase**: `searchIssues`, `createMessage`
- Conversion: lowercase the first letter — `ActionGetIssue` → `actionGetIssue`, `CreateMessage` → `createMessage`

### Credential requests and auth type
- Using `appModules: ["*"]` defaults to **OAuth** for most apps
- Specifying the exact module name (e.g. `["searchIssues"]`) selects the correct auth type for that module (may be API key / basic)
- If a user hits an OAuth redirect they didn't expect, the module name was likely wrong or used `["*"]`

### Tool creation
- **Always** call `app-module_get` with `format: "instructions"` before creating a tool — this gives you the exact mapper and parameter schema
- Hardcode static values (like "send to Anton", a fixed channel, etc.) directly in the mapper
- Only expose dynamic `inputs` for values the user needs to provide per-run
- Connection IDs always go in `parameters`, never in `mapper`
- `{{var.input.fieldName}}` is the syntax for referencing inputs in the mapper

### Known API issues (Jira)
- The `jira:SearchIssues` module uses a deprecated API (returns HTTP 410)
- For Jira searches, use `jira:makeApiCall2` with `GET /3/search/jql` and pass the `jql` query as a query string parameter
- `jira:ActionGetIssue` works correctly for fetching a single ticket by key

### Updating vs recreating tools
- `tools_update` can return internal server errors for significant structural changes
- Prefer creating a new tool (`tools_create`) over updating if you need to change the module or mapper structure

---

## Available Make MCP tools (reference)

**Discovery**
- `apps_recommend` — find the right app given a natural language intention
- `app-modules_list` — list modules for an app (`usage: "tool"` to filter)
- `app-module_get` — get full parameter schema (`format: "instructions"`)

**Connections**
- `connections_list` — list existing connections, optionally filtered by `type`
- `credential_requests_create` — ask the user to authorize a new connection

**Tools**
- `tools_create` — create a new Make tool
- `tools_update` — update an existing tool
- `tools_get` — inspect a tool's current configuration

**Execution**
- `scenarios_run` — run a tool (use tool ID as `scenarioId`, set `responsive: true`)

**Account**
- `users_me` — get current user info (includes user ID)
- `organizations_list` — list orgs (to find `organizationId` and `teamId`)

---

## Examples

### Send a Slack DM
1. `apps_recommend` → "slack" v4
2. `app-modules_list` (slack, v4, usage: tool) → `CreateMessage`
3. `app-module_get` (slack, v4, CreateMessage, instructions)
4. `connections_list` type: ["slack2", "slack3"] → find existing connection
5. If none: `credential_requests_create` appModules: ["createMessage"] → share URL
6. `tools_create`: module `slack:CreateMessage`, hardcode channel to user's ID, dynamic `message` input
7. `scenarios_run` with the message text

### Search Jira tickets
1. `apps_recommend` → "jira" v1
2. `app-modules_list` (jira, v1, usage: tool) → use `makeApiCall2` for search
3. `app-module_get` (jira, v1, makeApiCall2, instructions)
4. `connections_list` type: ["jira"] → find existing connection
5. If none: `credential_requests_create` appModules: ["searchIssues"] → share URL
6. `tools_create`: module `jira:makeApiCall2`, GET `/3/search/jql`, dynamic `query` input mapped to `jql` query param
7. `scenarios_run` with the search term

### Get a Jira ticket by key
Same as above but use `jira:ActionGetIssue` with a dynamic `issueKey` input mapped to `issueId`.
