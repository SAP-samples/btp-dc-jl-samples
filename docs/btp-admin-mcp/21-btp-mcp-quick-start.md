# Quick Start for Claude Code

This guide walks you through connecting Claude Code to the MCP Server for SAP BTP Administration and running your first request. The whole process takes about five minutes.

This guide uses Default Identity Provider (SAP IDS). If your platform user comes from a custom trust configured in your global account, follow the SAP Help Portal guide instead [Connect to the MCP Server for SAP BTP Administration](https://help.sap.com/docs/btp/sap-business-technology-platform/connect-to-mcp-server-for-sap-btp-administration).

## Prerequisites

**1. Example with AI assistant Claude Code**

Install [Claude Code](https://code.claude.com/docs/en/overview) if you have not already. Choose your preferred client (e.g. terminal, VS Code, etc.)

Connect to your LLM model provider following the code assistant documentation.


**2. Access to a BTP Global Account**

You need a BTP Global Enterprise or Trial Account. 


---

## Step 1: Register the MCP Server

Run this command once in your terminal. It registers the MCP Server in Claude Code's configuration:

```bash
claude mcp add --transport http BTP-Administration \
  "https://sso.mcp.btp.cloud.sap/mcp" \
  --client-id e789ba01-5612-47ee-bfe7-79e26411c1ca
```

You only need to do this once. The server will remain registered across sessions.

## Step 2: Start Claude Code

In your terminal, start an interactive Claude Code session:

```bash
claude
```

See [Claude Code documentation](https://code.claude.com/docs/en/overview) for other ways to use it (project context, IDE integration, etc.).

## Step 3: Activate the MCP Server and Log In

In the Claude Code prompt, type:

```
/mcp
```

A list of your registered MCP servers appears. Select `BTP-Administration`.

Claude Code will open your default browser and take you through the SAP SSO login flow. Log in with your SAP account. Once the browser confirms the login, return to your terminal; the server is now connected and active for this session.

> **Note:** The SSO flow uses the OAuth 2.0 Authorization Code Flow against SAP Identity Service (`accounts.sap.com`). Claude Code handles token management and renewal automatically.

## Step 4: Try It

You are now ready to use natural language to manage BTP. The assistant automatically discovers all available tools; you do not need to know tool names or API syntax.

Some things to try:

**Basic exploration:**
> "List all subaccounts in my global account."

**Filtered query:**
> "Which of my subaccounts have a Cloud Foundry environment enabled?"

**Security audit:**
> "Show me all role collections assigned to user alice@example.com across all subaccounts."

**Multi-step workflow:**
> "I need to create a new subaccount called 'dev-test' in the EU10 region. What do I need to have in place first?"

The assistant will call the appropriate BTP tools in sequence, present results clearly, and ask for confirmation before any write or destructive operation.

> **Tip:** If you are unsure what the server can do, just ask: *"What BTP operations can you help me with?"*


