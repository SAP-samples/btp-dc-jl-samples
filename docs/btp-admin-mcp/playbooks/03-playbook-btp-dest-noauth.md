# BTP Destination — Northwind (Playbook)

Playbook for creating a BTP Destination to an OData service.
Generic placeholders are in `<angle brackets>`, concrete values for the reference setup must be provided in addition.

---

### Usage

- Copy this playbook to your project folder, so your AI Coding Assistant can read it.
- Connect your BTP Admin MCP Server to your trial or dev account.
- Command: create a destination for northwind v4 following `03-playbook-btp-dest-noauth.md`

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



**Manually (BTP Cockpit):**
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
| Usage | Joule capability configuration |

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
> "Add a destination for `https://petstore3.swagger.io/api/v3/openapi.json` following `03-playbook-btp-dest-noauth.md`"
