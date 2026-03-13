# Make Connector — Claude Plugin

Connect Claude to 3500+ apps via Make. When you ask Claude to do something involving a
third-party service — send a Slack message, search Jira, update a spreadsheet — Claude
will use Make to build and run a tool for you on the fly, right in the conversation.

## What it does

1. Detects when you need a third-party integration
2. Finds the right Make app and module for your request
3. Walks you through authorizing a connection (if you haven't already)
4. Creates a Make tool and runs it immediately
5. Shows you the result

No scenario-building, no manual configuration. One conversation, done.

## Setup

### 1. Install the plugin

**From the Claude Code marketplace:**

```bash
claude plugin install https://github.com/blackeyeliner/make-connector-plugin
```

**Or manually:**

```bash
git clone https://github.com/blackeyeliner/make-connector-plugin ~/.claude/plugins/make-connector
```

### 2. Connect Make

The plugin includes `.mcp.json` which automatically registers the Make MCP server.
When you first use a Make tool, Claude Code will prompt you to authenticate
with your Make account via OAuth.

### 3. Use it

Just ask Claude naturally:

- _"Send a Slack message to my team"_
- _"Search Jira for tickets about login"_
- _"Add a row to my Google Sheet"_
- _"Create a Notion page"_
- _"Look up a contact in HubSpot"_

Claude will handle the rest.

## How it works

The plugin includes a skill (`make-tool-builder`) that Claude auto-invokes
whenever it detects a request that requires a third-party integration.
The skill contains a step-by-step workflow and rules for using Make's MCP tools
to authorize, build, and execute integrations on demand.

## Requirements

- Claude Code with MCP support
- A [Make account](https://make.com) (free tier works)
