# BTP-Administration MCP Server with Claude Code — Playbook

Playbook for setting up and using the `BTP-Administration` MCP Server with a compatible AI coding assistant to manage SAP BTP accounts using natural language.

Compatible AI clients: Claude Code, GitHub Copilot in VS Code, OpenCode

---

## Server Endpoints

| Authentication mode | URL |
|---|---|
| **SSO** | `https://sso.mcp.btp.cloud.sap/mcp` |
| **Direct Connection** | `https://proxy.c-769d49e.kyma.ondemand.com/mcp` |

---

## Authentication Modes

| Mode | When to use | Required |
|---|---|---|
| **SSO** | BTP user comes from the Default Identity Provider (`accounts.sap.com`) | SAP account + browser |
| **Direct Connection** | BTP user comes from a custom IDP | BTP username, password, IAS tenant subdomain |

> **Note:** The server operates with the full scope of the SAP user's identity — if the user has access to multiple global accounts, the server can see all of them. The server enforces the same role-based access control as the BTP Cockpit.

**Finding the IAS tenant subdomain** (Direct Connection only):
BTP Cockpit → Global Account → Security → Trust Configuration

---

## Installation — Claude Code (Recommended)

### Option A: via `claude mcp add` (one-time, persists across sessions)

**Live (SSO):**
```bash
claude mcp add --transport http BTP-Administration \
  "https://sso.mcp.btp.cloud.sap/mcp" \
  --client-id e789ba01-5612-47ee-bfe7-79e26411c1ca
```

**Live (Direct Connection):**
```bash
export BTP_CREDENTIALS=$(echo -n "<your-username>:<your-password>" | base64)
export BTP_ORIGIN="<your-ias-tenant-subdomain>"  # omit for Default IDP

claude mcp add --transport http BTP-Administration \
  "https://proxy.c-769d49e.kyma.ondemand.com/mcp" \
  --header 'Authorization: Basic ${BTP_CREDENTIALS}' \
  --header 'X-Platform-Origin: ${BTP_ORIGIN}'
```

The command only needs to be run **once** — the server remains registered across sessions.

### Option B: manually via `~/.claude/mcp.json`

```json
{
  "mcpServers": {
    "BTP-Administration": {
      "type": "http",
      "url": "https://sso.mcp.btp.cloud.sap/mcp",
      "oauth": {
        "clientId": "e789ba01-5612-47ee-bfe7-79e26411c1ca"
      }
    }
  }
}
```

> **Note:** Option A is simpler and recommended.

---

## Permissions

The MCP server uses the permissions of the logged-in BTP user — it can only perform operations that the user is authorized to perform in the BTP Cockpit. A 403 error means the user is missing the required role collection.

Commonly required role collections:
- `Global Account Administrator` — for global account operations
- `Subaccount Administrator` — for subaccount operations
- `Connectivity and Destination Administrator` — for destinations

---

## First Login

1. Start `claude`
2. Type `/mcp` → select the registered server
3. Browser opens automatically → complete SAP login
4. Return to the terminal

---

## Usage Notes

### Tool Discovery
You don't need to know tool names — Claude automatically discovers all available BTP operations. Just ask in natural language:

```
"What BTP operations can you help me with?"
"List all subaccounts in my global account."
"Which subaccounts have Cloud Foundry enabled?"
"Show all role collections assigned to user alice@example.com across all subaccounts."
```

### Write Operations
Claude **asks for confirmation before every write or destructive operation** — no accidental deletions or creations. The API call is only executed after explicit confirmation.

### Multi-Step Workflows
The server can call multiple BTP tools in sequence and summarize results:

```
"I want to create a new subaccount 'dev-test' in EU10. What do I need to prepare?"
```

Claude checks prerequisites, lists the required steps, and executes them after confirmation.

---

## Session Management

| Token | Validity | Behavior |
|---|---|---|
| Access Token | 30 minutes | Renewed automatically by the client in the background |
| Refresh Token | 1 day, rotates on every use | Session ends after 1 day of inactivity |

Session ends when:
- The server has not been used for 1 day
- The user explicitly logs out from SAP Identity Service (`accounts.sap.com`)

In both cases, the client automatically opens the browser login again.

---

## Troubleshooting

| Error | Cause | Solution |
|---|---|---|
| Auth error: token expired | Access token expired | `/mcp` → select server → repeat browser login |
| Auth error: IDP not recognized | `BTP_ORIGIN` incorrect or missing (Direct Connection only) | Check IAS tenant subdomain: BTP Cockpit → Global Account → Trust Configuration |
| 403 on tool call | BTP user is missing the required role collection | The `server_message` in the error detail shows the required role collection → assign it in the BTP Cockpit |
| Server does not appear | URL or client ID incorrect | `claude mcp list` → if incorrect, `claude mcp remove <name>` and re-register |
| Support | — | SAP Component `BC-CP-ADMINMCP` |

---

## Available Tools (Selection)

| Tool | Description |
|---|---|
| `mcp__BTP-Administration__GlobalAccount-list` | List all global accounts |
| `mcp__BTP-Administration__GlobalAccount-get` | Get details of a global account |
| `mcp__BTP-Administration__Subaccount-list` | List all subaccounts |
| `mcp__BTP-Administration__Subaccount-create` | Create a new subaccount |
| `mcp__BTP-Administration__Subaccount-get` | Get details of a subaccount |
| `mcp__BTP-Administration__SubaccountServiceQuota-list` | List entitlements / service quota |
| `mcp__BTP-Administration__SubaccountEntitlement-assign` | Assign an entitlement to a subaccount |
| `mcp__BTP-Administration__ServiceInstance-create` | Create a service instance |
| `mcp__BTP-Administration__ServiceInstance-list` | List service instances |
| `mcp__BTP-Administration__Destination-create` | Create a destination |
| `mcp__BTP-Administration__Destination-list` | List destinations |
| `mcp__BTP-Administration__Destination-get` | Get a destination |
| `mcp__BTP-Administration__Destination-update` | Update a destination |
| `mcp__BTP-Administration__Destination-delete` | Delete a destination |
| `mcp__BTP-Administration__RoleCollection-assign` | Assign a role collection to a user |
| `mcp__BTP-Administration__Subaccount-subscribe` | Subscribe an app in a subaccount |
| `mcp__BTP-Administration__Navigation-buildLink` | Generate a BTP Cockpit URL |

---

## Required Information

For most operations Claude needs:

| Variable | Where to find it |
|---|---|
| `global_account` | Global Account subdomain — BTP Cockpit top left |
| `subaccount` | Subaccount ID — BTP Cockpit → Subaccount → Overview |

---

## Usage in Playbooks

This server is used in the following playbooks:

- `09-playbook-btp-entpr-aicore.md` — AI Core + AI Launchpad setup
- `03-playbook-btp-dest-noauth.md` — Create NoAuth destination
- `02-playbook-btp-admin-rest-apis.md` — BTP REST APIs (for scripts/CI-CD without AI client)
