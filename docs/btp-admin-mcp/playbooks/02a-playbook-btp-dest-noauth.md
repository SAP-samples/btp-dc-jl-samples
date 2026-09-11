# Set Up a BTP Destination with Northwind OData Service

## AI BTP Admin Playbook

Playbook for creating a BTP Destination to an OData service.
Generic placeholders are in `<angle brackets>`, concrete values for the reference setup must be provided in addition.

---

## How to Use This Playbook

Connect to your BTP-Administration MCP Server with your global account and subaccount.
Start your local AI assistant, then pass this playbook with your account values:        

```
Follow 02a-playbook-btp-dest-noauth.md to create a destination.
Destination name: Northwind, URL: https://services.odata.org/V4/Northwind/Northwind.svc/
```

Claude reads the playbook, fills in your values, and executes each step using the BTP-Administration MCP Server. Before any create or write operation, Claude asks for confirmation.

---

## Step 0: Check if Destination Already Exists

**Via Claude MCP (`mcp__BTP-Administration__Destination-list`):**

Required:
- `<GLOBAL_ACCOUNT>` — Global Account subdomain (e.g. `my-global-account`)
- `<SUBACCOUNT_ID>` — Subaccount ID (for example, looks like: `a1b2c3d4-1234-5678-abcd-ef1234567890`)

```
mcp__BTP-Administration__Destination-list(
  global_account = "<GLOBAL_ACCOUNT>",
  subaccount = "<SUBACCOUNT_ID>"
)
```

→ Returns all existing destinations. If `<DESTINATION_NAME>` already exists → skip Step 1.

---

## Step 1: Create Destination

**Via Claude MCP (`mcp__BTP-Administration__Destination-create`):**

Required:
- `<GLOBAL_ACCOUNT>` — Global Account subdomain (e.g. `my-global-account`)
- `<SUBACCOUNT_ID>` — Subaccount ID (e.g. `a1b2c3d4-1234-5678-abcd-ef1234567890`)
- `<DESTINATION_NAME>` — desired name for the destination
- `<DESTINATION_URL>` — target URL of the backend service

```
mcp__BTP-Administration__Destination-create(
  global_account = "<GLOBAL_ACCOUNT>",
  subaccount = "<SUBACCOUNT_ID>",
  destination_configuration = {
    "Name": "<DESTINATION_NAME>",
    "URL": "<DESTINATION_URL>",
    "Type": "HTTP",
    "ProxyType": "Internet",
    "Authentication": "NoAuthentication"
  }
)
```



**Manually (Using BTP Cockpit):**
1. BTP Cockpit → `<SUBACCOUNT>` → Connectivity → Destinations → **New Destination**
2. Fill in the fields:

| Property | Value |
|---|---|
| Name | `<DESTINATION_NAME>` |
| URL | `<DESTINATION_URL>` |
| Type | HTTP |
| ProxyType | Internet |
| Authentication | NoAuthentication |

Here, you can also manually test the connection.

3. **Save**

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

**Manually (BTP Cockpit):**
1. BTP Cockpit → `<SUBACCOUNT>` → Connectivity → Destinations
2. Find `<DESTINATION_NAME>` in the list → **Check Connection**

---

## Reference Setup

| Variable | Value |
|---|---|
| `<GLOBAL_ACCOUNT>` | `<your-global-account-subdomain>` |
| `<SUBACCOUNT_ID>` | `<your-subaccount-id>` |
| `<DESTINATION_NAME>` | `Northwind` |
| `<DESTINATION_URL>` | `https://services.odata.org/V4/Northwind/Northwind.svc/` |

| Component | Status |
|---|---|
| Destination `Northwind` | |

---

## Optional: Create Destination from an OpenAPI Spec URL

If you have an OpenAPI spec URL (e.g. `https://example.com/api/v3/openapi.json`), derive the API base URL first, then create the destination.

**Step 1: Derive the base URL**

Strip the spec file path from the URL — the base URL is everything up to and including the API version segment:

| Spec URL | Base URL |
|---|---|
| `https://petstore3.swagger.io/api/v3/openapi.json` | `https://petstore3.swagger.io/api/v3` |

You can also ask your AI coding assistant: "What is the API base URL for `<SPEC_URL>`?" — it will fetch and analyze the spec.

**Step 2: Create the destination**

Use the derived base URL as `<DESTINATION_URL>` and follow Step 1 of this playbook normally.

```
mcp__BTP-Administration__Destination-create(
  global_account = "<GLOBAL_ACCOUNT>",
  subaccount = "<SUBACCOUNT_ID>",
  destination_configuration = {
    "Name": "<DESTINATION_NAME>",
    "URL": "<BASE_URL>",
    "Type": "HTTP",
    "ProxyType": "Internet",
    "Authentication": "NoAuthentication"
  }
)
```

**Example command:**
> "Add a destination for `https://petstore3.swagger.io/api/v3/openapi.json` following `02a-playbook-btp-dest-noauth.md`"
