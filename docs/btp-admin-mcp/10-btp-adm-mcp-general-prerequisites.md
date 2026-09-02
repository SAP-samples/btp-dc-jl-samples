# General Prerequisites for BTP Account Administration with MCP

Before working with this mission, the following tools and access must be in place.

### SAP BTP Access

- An active **SAP BTP account** (trial or non-productive) with sufficient permissions to manage global accounts, subaccounts, entitlements, service instances, and role collections
- Access to the **SAP BTP Cockpit** at [account.hanatrial.ondemand.com](https://account.hanatrial.ondemand.com) (trial) or [cockpit.btp.cloud.sap](https://cockpit.btp.cloud.sap) (productive)
- A user with *Global Account Administrator* or *Subaccount Administrator* role

---

### MCP Server: BTP-Administration

The `BTP-Administration` MCP server is required for the AI coding assistant to interact with SAP BTP directly (list accounts, assign roles, create instances, etc.).

- Install and configure the `BTP-Administration` MCP server in your AI coding assistant's settings
- Authenticate before each session — tokens expire, re-authenticate via `/mcp` if you see authorization errors
- Without `BTP-Administration`, all BTP operations must be done manually via the Cockpit or CLI

Two authentication modes are available:

| Authentication mode | When to use | What you need |
|---|---|---|
| **Single Sign-On (SSO)** | Your BTP platform user comes from the Default Identity Provider (accounts.sap.com). | An SAP user account and browser access for the one-time login flow. |
| **Direct Connection** | Your BTP platform user comes from a custom identity provider configured as a platform trust in your global account. | Your BTP username, password (Base64-encoded), and IAS tenant subdomain. |

The server endpoints and OAuth client ID for SSO:

| Authentication mode | URL |
|---|---|
| **Single Sign-On** | `https://sso.mcp.btp.cloud.sap/mcp` |
| **Direct Connection** | `https://proxy.c-769d49e.kyma.ondemand.com/mcp` |

SSO OAuth client ID: `e789ba01-5612-47ee-bfe7-79e26411c1ca` (fixed, does not change)

> **Note:** After the initial login, your session is maintained automatically. Access tokens expire after 30 minutes; refresh tokens are valid for 1 day and rotate on every use. Your session ends if the server goes unused for a full day or if you explicitly log out from the browser.

> **Caution:** The MCP server operates with the full scope of your SAP user identity — it has access to all global accounts associated with your identity. Use a dedicated trial account if you want to restrict access.

For step-by-step setup instructions, see `21-btp-mcp-quick-start.md` (Claude Code / SSO) or `22-btp-adm-mcp-connect.md` (all clients and authentication modes).

---

### BTP CLIs (Optional)

Some mission steps reference CLI commands as alternatives to UI operations.

| CLI | Purpose | Required |
|---|---|---|
| **BTP CLI** (`btp`) | Global account management, entitlements, role assignments | Optional |
| **CF CLI** (`cf`) | Cloud Foundry space management, service instances, service keys | Optional (required for CF instances setup) |

Install via:
- BTP CLI: [tools.hana.ondemand.com](https://tools.hana.ondemand.com/#cloud) or use package manager, e.g. on mac os `brew install btp`
- CF CLI: [github.com/cloudfoundry/cli/releases](https://github.com/cloudfoundry/cli/releases) or use package manager, e.g. on mac os `brew install cloudfoundry/tap/cf-cli@8`

> **Note:** `tools.hana.ondemand.com` requires SAP login.

---

### AI Coding Assistant

Any AI client that supports the MCP standard can be used. Examples of compatible AI coding assistants:

- **Claude Code** (CLI or desktop app)
- **GitHub Copilot** in VS Code
- **OpenCode**

Requirements:
- The assistant must support MCP servers to use `BTP-Administration` or optional integrations like browser automation
- Access to an LLM — either via an LLM API key directly or a compatible proxy
- Sufficient context window for multi-step sessions


---

### Browser with DevTools

A browser with developer tools access is recommended for navigating certain service UIs that cannot be automated. To control your browser via an AI Coding assistant, you can use "browser automation" or "browser control" MCP Servers. For example Google Chrome DevTools, Playwright MCP servers, or others.

---

