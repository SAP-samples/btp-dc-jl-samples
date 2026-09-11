# How to Use BTP AI Admin Playbooks with local AI Assistants

Playbooks are step-by-step instruction files that guide AI Assistants through common SAP BTP administration tasks. You copy a playbook into your project, reference it in a prompt, and Claude follows the steps — calling the BTP MCP Server tools, checking results, and confirming before any write operation.

Compatible AI clients are: Claude Code, GitHub Copilot in VS Code, OpenCode.

## Prerequisites

- Local AI Assistant is installed and running.
- The MCP Server for SAP BTP Administration is registered in Claude Code.
- You have your **Global Account subdomain** and **Subaccount ID** ready (find them in the BTP Cockpit).

## How It Works

1. Copy the playbook file into your project folder (or any folder you open with `claude`).
2. Start AI Assistant, e.g., Claude Code: `claude`
3. Give AI Assistant a short instruction that references the playbook file by name.

AI Assistant reads the playbook, fills in your account values, and executes each step using the BTP MCP Server tools. Before any create, update, or delete operation, AI Assistant asks for your confirmation.

## Example Commands

**Set up the BTP MCP Server:**
```
Follow the playbook "01-playbook-btp-adm-mcp-setup.md" to connect me to my BTP account.
```

**Create a destination for an OData service:**
```
Create a destination for Northwind V4 following 02a-playbook-btp-dest-noauth.md.
My global account is my-global-account and my subaccount ID is a1b2c3d4-...
```

**Create a destination from an OpenAPI spec URL:**
```
Add a destination for https://petstore3.swagger.io/api/v3/openapi.json
following 02a-playbook-btp-dest-noauth.md.
```


## Using a Playbook as a Template

Playbooks are not limited to the exact scenario they describe — you can use them as a starting point for your own setup. For example, if you want to create a destination for a different service than Northwind, you do not need to write a new playbook. Just reference the existing one and tell Claude what to use instead:

```
Follow 02a-playbook-btp-dest-noauth.md to create a destination,
but instead of Northwind use the URL https://api.example.com/v2.
Name it MyService.
```

AI Assistant follows the same steps from the playbook — checking for existing destinations, creating, verifying — but substitutes your values. The playbook acts as the workflow; your prompt provides the specifics.

---

## Tips

- **You do not need to know the tool names.** AI Assistant discovers available BTP operations automatically and maps playbook steps to the right tools.
- **Placeholders in angle brackets** (e.g. `<GLOBAL_ACCOUNT>`) are filled in by AI Assistant based on your prompt or by asking you.
- **Write operations always require confirmation.** AI Assistant will show you what it is about to do before executing it.
- **You can ask follow-up questions mid-playbook.** For example: "Before creating the destination, show me what destinations already exist."

## Available Playbooks

| File | Purpose |
|---|---|
| `01-playbook-btp-adm-mcp-setup.md` | Connect Claude Code to the BTP MCP Server (SSO and Direct Connection) |
| `02a-playbook-btp-dest-noauth.md` | Create a BTP destination for an HTTP service without authentication |
| `02b-playbook-btp-dest-apihub-sandbox.md` | Create a BTP destination for the SAP Business Accelerator Hub sandbox (API key auth) |
| `03a-playbook-btp-trial-workzone.md` | Set up SAP Build Work Zone in a BTP trial subaccount |
| `03b-playbook-btp-ea-work-zone.md` | Set up SAP Build Work Zone, standard edition in an enterprise subaccount |
| `04-playbook-btp-trial-integrationsuite.md` | Set up SAP Integration Suite in a BTP trial subaccount |
| `05-playbook-btp-ea-aicore.md` | Set up AI Core and AI Launchpad in a BTP enterprise subaccount |
| `pb-using-cli-rest-apis/playbook-btp-admin-cli-apis.md` | Administer BTP accounts using REST APIs directly (no MCP Server needed) |
| `pb-using-cli-rest-apis/02c-playbook-btp-dest-cli-oauth2.md` | Create a BTP destination for the SAP Cloud Management Service API using OAuth2 Client Credentials |
| `pb-using-cli-rest-apis/playbook-btp-kyma-kubernetes-cli.md` | Set up a Kyma environment in a BTP subaccount from scratch via BTP MCP Server and CLIs |
