# BTP Destination — SAP Business Accelerator Hub Sandbox (Playbook)

Playbook for creating a BTP Destination to the SAP Business Accelerator Hub sandbox.
The sandbox uses a static API key passed as a custom request header — no OAuth token exchange required.
Generic placeholders are in `<angle brackets>`. Concrete values for the reference setup are in the table at the end.

---

### Usage

- Copy this playbook to your project folder so your AI coding assistant can read it.
- Connect your BTP Admin MCP Server to your trial or dev account.
- Command: `Create a destination for the SAP Business Accelerator Hub sandbox following 03b-playbook-btp-dest-apihub-sandbox.md`

### Prerequisites

- A free SAP Universal ID account at [api.sap.com](https://api.sap.com)
- Your **API Key** from the SAP Business Accelerator Hub (Settings → Show API Key)

---

## Step 0: Check if Destination Already Exists

**Via Claude MCP (`mcp__BTP-Administration__Destination-list`):**

Required:
- `<GLOBAL_ACCOUNT>` — Global Account subdomain (e.g. `my-global-account`)
- `<SUBACCOUNT_ID>` — Subaccount ID (e.g. `a1b2c3d4-1234-5678-abcd-ef1234567890`)

```
mcp__BTP-Administration__Destination-list(
  global_account = "<GLOBAL_ACCOUNT>",
  subaccount = "<SUBACCOUNT_ID>"
)
```

→ Returns all existing destinations. If `<DESTINATION_NAME>` already exists → skip Step 1.

---

## Step 1: Create Destination

The SAP Business Accelerator Hub sandbox authenticates via a static API key sent as a custom HTTP request header (`APIKey`). In BTP Destination Service this is modelled as `NoAuthentication` with an additional property that injects the header.

**Via Claude MCP (`mcp__BTP-Administration__Destination-create`):**

Required:
- `<GLOBAL_ACCOUNT>` — Global Account subdomain
- `<SUBACCOUNT_ID>` — Subaccount ID
- `<DESTINATION_NAME>` — desired name for the destination (e.g. `APIHubSandbox`)
- `<API_KEY>` — your API key from api.sap.com

```
mcp__BTP-Administration__Destination-create(
  global_account = "<GLOBAL_ACCOUNT>",
  subaccount = "<SUBACCOUNT_ID>",
  destination_configuration = {
    "Name": "<DESTINATION_NAME>",
    "URL": "https://sandbox.api.sap.com",
    "Type": "HTTP",
    "ProxyType": "Internet",
    "Authentication": "NoAuthentication",
    "URL.headers.APIKey": "<API_KEY>"
  }
)
```

> The `URL.headers.APIKey` property instructs the BTP Destination Service to inject the header `APIKey: <API_KEY>` into every outbound request. The API key itself is stored securely in the destination and never exposed to the calling application.

**Manually (BTP Cockpit):**
1. BTP Cockpit → `<SUBACCOUNT>` → Connectivity → Destinations → **New Destination**
2. Fill in the fields:

| Property | Value |
|---|---|
| Name | `<DESTINATION_NAME>` |
| URL | `https://sandbox.api.sap.com` |
| Type | HTTP |
| ProxyType | Internet |
| Authentication | NoAuthentication |

3. Under **Additional Properties**, add:

| Key | Value |
|---|---|
| `URL.headers.APIKey` | `<API_KEY>` |

4. **Save**

---

## Step 2: Verify Destination

**Via Claude MCP (`mcp__BTP-Administration__Destination-get`):**

```
mcp__BTP-Administration__Destination-get(
  global_account = "<GLOBAL_ACCOUNT>",
  subaccount = "<SUBACCOUNT_ID>",
  name = "<DESTINATION_NAME>"
)
```

→ Confirm `URL`, `Authentication: NoAuthentication`, and the `URL.headers.APIKey` additional property are present.

**Manually (BTP Cockpit):**
1. BTP Cockpit → `<SUBACCOUNT>` → Connectivity → Destinations
2. Find `<DESTINATION_NAME>` in the list → **Check Connection**

---

## Step 3: Test the API

Once the destination is in place, verify it works end-to-end by calling the Business Partner sandbox API directly:

```bash
curl -s "https://sandbox.api.sap.com/s4hanacloud/sap/opu/odata/sap/API_BUSINESS_PARTNER/A_BusinessPartner?\$top=5" \
  -H "APIKey: <API_KEY>" \
  -H "Accept: application/json" \
  --compressed
```

→ Expect a JSON response with up to 5 business partner records. A `200 OK` with data confirms the API key is valid and the sandbox is reachable.

---

## Reference Setup

| Variable | Value |
|---|---|
| `<GLOBAL_ACCOUNT>` | `<your-global-account-subdomain>` |
| `<SUBACCOUNT_ID>` | `<your-subaccount-id>` |
| `<DESTINATION_NAME>` | `APIHubSandbox` |
| `<API_KEY>` | obtain from [api.sap.com](https://api.sap.com) → Settings → Show API Key |

### Examples for Available Sandbox APIs (same base URL)

| API | Path |
|---|---|
| Business Partner | `/s4hanacloud/sap/opu/odata/sap/API_BUSINESS_PARTNER` |
| Maintenance Notification | `/s4hanacloud/sap/opu/odata/sap/API_MAINTNOTIFICATION` |

See the SAP Business Accelerator Hub documentation for example queries and OData tips.
