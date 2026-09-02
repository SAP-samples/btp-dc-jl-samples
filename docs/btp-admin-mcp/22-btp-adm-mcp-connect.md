# Connect to the MCP Server for SAP BTP Administration

Register the MCP Server for SAP BTP Administration in your AI client to start managing your BTP account using natural language.

For more information, see SAP Help Portal [Connect to the MCP Server for SAP BTP Administration](https://help.sap.com/docs/btp/sap-business-technology-platform/connect-to-mcp-server-for-sap-btp-administration).

### Prerequisites

Ensure that you meet the following requirements before connecting:

- You have an MCP-compatible AI client installed. Examples of supported clients include Claude Code, GitHub Copilot in VS Code, and OpenCode.
- You have access to your non-production SAP BTP global account.
- You have determined which authentication mode applies to your setup:

| Authentication mode | When to use | What you need |
|---|---|---|
| **Single Sign-On (SSO)** | Your BTP platform user comes from the Default Identity Provider (accounts.sap.com). | An SAP user account and browser access for the one-time login. |
| **Direct Connection** | Your BTP platform user comes from a custom identity provider configured as a platform trust in your global account. | Your BTP username, password, and IAS tenant subdomain. |

> **Note:** After the initial login, your session is maintained automatically:
> - Access tokens expire after 30 minutes.
> - Refresh tokens are valid for 1 day and rotate on every use.
> - Your MCP client renews access tokens silently in the background.
>
> Your session ends if the server goes unused for a full day, or if you explicitly log out of your SAP account from the browser (SAP Identity Service at accounts.sap.com). In either case, your client opens the browser login again automatically.

### Context

The MCP Server for SAP BTP Administration is available at the following endpoint:

| Authentication mode | URL |
|---|---|
| **Single Sign-On** | `https://sso.mcp.btp.cloud.sap/mcp` |
| **Direct Connection** | `https://proxy.c-769d49e.kyma.ondemand.com/mcp` |

For SSO, you must also provide the OAuth client ID when registering the server: `e789ba01-5612-47ee-bfe7-79e26411c1ca`

> **Note:** The authorization server does not support Dynamic Client Registration, which is why you must supply the client ID explicitly. The value is fixed and does not change.


> **Caution:** The MCP Server operates with the full scope of your SAP user identity. If your trial account shares the same email address as your productive accounts, the server will have access to all global accounts associated with that identity. To restrict access to a trial account only, register it under a separate email address.


### Procedure

#### Claude Code

##### Single Sign-On

1. Open a terminal and run the following command:

   ```bash
   claude mcp add --transport http BTP-Administration \
     "https://sso.mcp.btp.cloud.sap/mcp" \
     --client-id e789ba01-5612-47ee-bfe7-79e26411c1ca
   ```

   You only need to run this command once — the server remains registered across sessions.

2. Start Claude Code: `claude`
3. Type `/mcp` at the prompt and select the server you registered.
4. Complete the SAP login in the browser that opens, then return to your terminal.

##### Direct Connection

1. Encode your credentials as Base64 in the format `<username>:<password>` and store them in an environment variable, along with your IAS tenant subdomain.

   **macOS and Linux:**
   ```bash
   export BTP_CREDENTIALS=$(echo -n "<your-username>:<your-password>" | base64)
   export BTP_ORIGIN="<your-ias-tenant-subdomain>"
   ```

   **Windows (PowerShell):**
   ```powershell
   $Env:BTP_CREDENTIALS = [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes('<your-username>:<your-password>'))
   $Env:BTP_ORIGIN = '<your-ias-tenant-subdomain>'
   ```

   > **Tip:** Add these exports to your shell profile (`~/.zshrc` or `~/.bashrc`) so they are available whenever you run `claude`.

2. Register the server:

   ```bash
   claude mcp add --transport http BTP-Administration \
     "https://proxy.c-769d49e.kyma.ondemand.com/mcp" \
     --header 'Authorization: Basic ${BTP_CREDENTIALS}' \
     --header 'X-Platform-Origin: ${BTP_ORIGIN}'
   ```

3. Start Claude Code: `claude`

---

#### GitHub Copilot in VS Code

##### Single Sign-On

1. Open a terminal and run the following command:

   ```json
   {
     "mcpServers": {
       "BTP Administration": {
         "type": "streamableHttp",
         "url": "https://sso.mcp.btp.cloud.sap/mcp"
       }
     }
   }
   ```

2. When VS Code prompts for the OAuth client ID, enter `e789ba01-5612-47ee-bfe7-79e26411c1ca`. VS Code stores this value and will not ask again.
3. Complete the SAP login in the browser that opens, then return to VS Code.

##### Direct Connection

1. Before configuring VS Code, compute the Base64 encoding of your credentials in the format `<username>:<password>`:

   **macOS and Linux:**
   ```bash
   echo -n "<your-username>:<your-password>" | base64
   ```

   **Windows (PowerShell):**
   ```powershell
   [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes('<your-username>:<your-password>'))
   ```

   Copy the output — you will enter it in the next step.

2. Open or create `.vscode/mcp.json` and add the following. VS Code prompts for credentials once per session and does not store them on disk.

   ```json
   {
     "inputs": [
       {
         "id": "btp-credentials",
         "type": "promptString",
         "description": "Base64-encoded BTP credentials (<username>:<password>)",
         "password": true
       },
       {
         "id": "btp-origin",
         "type": "promptString",
         "description": "IAS tenant subdomain"
       }
     ],
     "mcpServers": {
       "BTP Administration": {
         "type": "streamableHttp",
         "url": "https://proxy.c-769d49e.kyma.ondemand.com/mcp",
         "headers": {
           "Authorization": "Basic ${input:btp-credentials}",
           "X-Platform-Origin": "${input:btp-origin}"
         }
       }
     }
   }
   ```

---

#### OpenCode

##### Single Sign-On

1. Open your `opencode.json` configuration file and add the following entry under the `mcp` key:

   ```json
   {
     "mcp": {
       "BTP Administration": {
         "type": "remote",
         "url": "https://sso.mcp.btp.cloud.sap/mcp",
         "oauth": {
           "clientId": "e789ba01-5612-47ee-bfe7-79e26411c1ca"
         }
       }
     }
   }
   ```

2. Start OpenCode. Complete the SAP login in the browser that opens, then return to OpenCode.

##### Direct Connection

1. Encode your credentials as Base64 in the format `<username>:<password>` and store them in an environment variable, along with your IAS tenant subdomain.

   **macOS and Linux:**
   ```bash
   export BTP_CREDENTIALS=$(echo -n "<your-username>:<your-password>" | base64)
   export BTP_ORIGIN="<your-ias-tenant-subdomain>"
   ```

   **Windows (PowerShell):**
   ```powershell
   $Env:BTP_CREDENTIALS = [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes('<your-username>:<your-password>'))
   $Env:BTP_ORIGIN = '<your-ias-tenant-subdomain>'
   ```

2. Add the following to your `opencode.json`:

   ```json
   {
     "mcp": {
       "BTP Administration": {
         "type": "remote",
         "url": "https://proxy.c-769d49e.kyma.ondemand.com/mcp",
         "headers": {
           "Authorization": "Basic {env:BTP_CREDENTIALS}",
           "X-Platform-Origin": "{env:BTP_ORIGIN}"
         }
       }
     }
   }
   ```

### Results

Your AI client is now connected to the MCP Server for SAP BTP Administration and can invoke BTP administration operations on your behalf.

---

### Next Steps

If you encounter errors during or after connecting, see [Troubleshooting the MCP Server for SAP BTP Administration](https://help.sap.com/docs/btp/sap-business-technology-platform/troubleshooting-mcp-server-for-sap-btp-administration).

---

### Troubleshooting

If you encounter errors during or after connecting, refer to the common causes and solutions below.

| Error | Cause | Solution |
|---|---|---|
| Authentication fails: session expired or token no longer valid. | The authentication token from your last login has expired. Tokens are short-lived by design. | Re-authenticate: **Claude Code:** Type `/mcp`, select the server, and complete the browser login again. **GitHub Copilot in VS Code:** VS Code detects the expired session and prompts you to log in again automatically. **OpenCode:** Restart OpenCode and complete the browser login when prompted. |
| Authentication fails: identity provider not recognized or tenant not found. | The IAS tenant subdomain in your Direct Connection configuration is incorrect or missing. | Check the subdomain value: **Claude Code:** Verify that `BTP_ORIGIN` is set correctly and exported in the shell where you run `claude`. **GitHub Copilot in VS Code:** Check the `X-Platform-Origin` header value in your `mcp.json`. **OpenCode:** Check the `X-Platform-Origin` header value in your `opencode.json`. You can find the correct subdomain in the SAP BTP Cockpit under your global account trust configuration. If your platform user comes from the Default Identity Provider (accounts.sap.com), omit this header entirely. |
| A tool call returns a 403 error or an "operation not authorized" message. | Your BTP user is missing the role collection required for the requested operation. The MCP Server enforces the same role-based access control as the SAP BTP Cockpit. | Check the error message for the name of the required role collection, then ask your administrator to assign it to your user in the SAP BTP Cockpit. No reconnection is needed after the assignment. |
| The MCP server does not appear after registration, or the client cannot reach it. | The server URL or client ID is incorrect, or the required environment variables are not available in the current shell session. | **Claude Code:** Run `claude mcp list` to verify the registered URL. If incorrect, run `claude mcp remove <server-name>` and re-register. For Direct Connection, confirm that `BTP_CREDENTIALS` and `BTP_ORIGIN` are exported in your current shell. **GitHub Copilot in VS Code:** Check the URL in `.vscode/mcp.json`. **OpenCode:** Check the URL and, for SSO, the `clientId` value in `opencode.json`. |

### Reporting an Issue

If the steps above do not resolve your issue, open a support request using SAP component **BC-CP-ADMINMCP**.
