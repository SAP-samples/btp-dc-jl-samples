# SAP BTP Account Administration Using MCP Server

The MCP Server for SAP BTP Administration lets you manage your SAP BTP account — subaccounts, entitlements, services, and role assignments — through natural language in a compatible AI coding assistant (examples of supported clients include Claude Code, GitHub Copilot in VS Code, and OpenCode). It implements the Model Context Protocol (MCP), an open standard for connecting AI assistants to external tools. The server exposes BTP administration operations across four domains (accounts, security, services, connectivity) as MCP tools that any compatible AI client can discover and invoke. You describe what you want; the assistant determines which tools to call.

All domains are accessible through a single endpoint — there is no need to connect to multiple servers or manage multiple configurations.

For more information, see SAP Help Portal, [Account Administration Using MCP Servers](https://help.sap.com/docs/btp/sap-business-technology-platform/account-administration-using-mcp-servers)

---

#### Example 

You ask: *"List all subaccounts in my global account and tell me which ones have Cloud Foundry enabled."* 

The assistant calls the appropriate tools, receives the data, and presents a clear summary — no CLI, no scripts, no manual API calls.

Comparison of ways to administer BTP accounts:

| | BTP Cockpit | btp CLI | MCP Server |
|---|---|---|---|
| Interaction | Mouse/Browser | Learn commands | Natural language |
| Multi-Step | Many clicks | Multiple commands | One sentence |
| Discoverability | Browse menus | Read `--help` | Just ask |
| Audit Trail | Cockpit logs | Shell history | Every tool call logged |
| Automation | No | Scripts required | Conversation turn |

---

#### Why Use the MCP Server

- **Natural language interface** — describe what you want instead of learning CLI syntax or navigating the cockpit.
- **AI-driven discoverability** — the assistant figures out which tools to invoke and in what order; you do not need to know operation names.
- **Scriptless automation** — multi-step workflows (inspect, compare, act) run as a single conversation turn.
- **Audit trail** — every tool call is logged, giving you a record of what the assistant did and when.

---

#### AI Clients for the BTP MCP Server

Any AI client that supports the MCP standard can connect to the MCP Server for SAP BTP Administration. Compatible clients include:

- Claude Code
- GitHub Copilot in VS Code
- OpenCode

---

### Beyond the MCP Server

The MCP Server is the recommended integration for most BTP administration tasks, but the MCP server does not cover all administrative tasks and an AI coding assistant is not limited to it. Depending on the tools available in your client, the assistant can also:

- **Run CLIs directly** — execute `btp` or `cf` commands in a terminal and interpret the output.
- **Call BTP REST APIs** — use the SAP BTP Account Administration APIs for operations not covered by the MCP Server (for example, usage reporting or XSUAA management).
- **Automate the BTP Cockpit** — use browser automation tools (for example, Playwright) to navigate the cockpit and perform actions through the UI.

These approaches can be combined in a single conversation: the assistant might use the MCP Server to list subaccounts, call a REST API to retrieve usage data for one of them, and open the cockpit to confirm a subscription — all from a single natural-language request.

---