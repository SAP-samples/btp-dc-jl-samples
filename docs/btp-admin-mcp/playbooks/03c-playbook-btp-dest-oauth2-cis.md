# BTP Destination — Cloud Management Service OAuth2 (Playbook)

Playbook for creating a BTP Destination to the SAP Cloud Management Service (technical name: `cis`) using OAuth2 Client Credentials.
The `cis` service exposes the BTP Account Administration APIs (subaccounts, entitlements, provisioning, events).
Generic placeholders are in `<angle brackets>`. Concrete values for the reference setup are in the table at the end.

## What Is the Cloud Management Service?

The SAP Cloud Management Service (`cis`) is the REST API layer behind the BTP Cockpit. It exposes the same account administration operations you perform manually in the UI — but as programmable APIs:

| API | What it does |
|---|---|
| **Accounts** | Create, update, delete subaccounts and directories |
| **Entitlements** | Assign and manage service quotas across subaccounts |
| **Provisioning** | Enable environments (CF, Kyma) and manage SaaS subscriptions |
| **Events** | Query audit events for administrative operations |

### Why create a BTP destination for it?

A destination lets any BTP-connected app or integration call the CIS APIs without handling OAuth2 tokens directly — the Destination Service fetches and caches the token automatically. Typical use cases:

- **Automation** — scripts or CAP apps that provision subaccounts, assign entitlements, or manage subscriptions programmatically
- **Monitoring** — dashboards that read account structure or entitlement consumption
- **Self-service portals** — internal tooling where teams can request and receive BTP resources without going through the Cockpit

---

### Usage

- Copy this playbook to your project folder so your AI coding assistant can read it.
- Connect your BTP Admin MCP Server to your trial or dev account.
- Command: `Create a destination for the CIS API following 03c-playbook-btp-dest-oauth2-cis.md`

### Prerequisites

- A BTP subaccount with `cis` plan `central` entitled
- BTP CLI (`btp`) installed and logged in:

```bash
btp login --url https://cli.btp.cloud.sap --subdomain <GLOBAL_ACCOUNT> --sso
```

The `--sso` flag triggers a **Single Sign-On** login using the **OAuth2 Device Authorization Grant** (RFC 8628): instead of entering your SAP credentials in the terminal, a browser window opens where you authenticate with your SAP Universal ID (the same account you use for the BTP Cockpit). The CLI then receives a token from the browser session — your password never touches the terminal. After login, set the target subaccount:

```bash
btp target --subaccount <SUBACCOUNT_ID>
```

---

## Security Considerations

The `CIS-Central` destination carries credentials with **full administrative scope** over your BTP account structure (create/update/delete subaccounts, assign entitlements). Before deploying it, consider the following:

**Who can use this destination?**
The destination is not publicly accessible. Only apps running in the same subaccount with a Destination Service binding can resolve and use it. An anonymous internet caller cannot reach it. However:

- Any app in the subaccount with a `destination` service binding can call the CIS Accounts API with full write access
- Any user with the `Destination Administrator` role collection in the subaccount can read the stored `clientSecret`

**Recommendations:**

| Risk | Mitigation |
|---|---|
| Over-privileged credentials | Use `cis central-viewer` plan instead if only read access is needed — it grants read-only scopes and supports Client Credentials by default |
| Shared dev subaccount | Put the destination in a dedicated subaccount used only by the automation app, not a shared dev environment |
| Broad destination access | Restrict the `Destination Administrator` role collection to only the users and apps that need it |
| Leaked credentials | Rotate the service binding periodically (`btp delete services/binding` + recreate) and update the destination |

---

## Important: Two Instance Creation Methods

BTP has two execution environments for service instances:

| Method | Visible to CF CLI | Visible to BTP CLI | Visible to BTP MCP Server |
|---|---|---|---|
| `cf create-service` | ✅ yes | ❌ no | ✅ yes |
| `mcp__BTP-Administration__ServiceInstance-create` | ❌ no | ✅ yes | ✅ yes |
| `btp create services/instance` | ❌ no | ✅ yes | ✅ yes |

**For `cis central` with OAuth2 Client Credentials: use the BTP MCP Server or BTP CLI** — the CF CLI does not support the required `grantType: clientCredentials` parameter (it returns an error).

---

## Step 0: Check if Destination Already Exists

**Via Claude MCP (`mcp__BTP-Administration__Destination-list`):**

```
mcp__BTP-Administration__Destination-list(
  global_account = "<GLOBAL_ACCOUNT>",
  subaccount = "<SUBACCOUNT_ID>"
)
```

→ If `<DESTINATION_NAME>` already exists → skip to Step 3.

---

## Step 1: Create Service Instance

The instance **must** be created with `{"grantType": "clientCredentials"}`. Without this parameter, the service defaults to Password grant type and the resulting credentials only carry the `uaa.resource` scope — not the CIS API scopes — causing a `403 insufficient_scope` error on every API call.

**Via Claude MCP (`mcp__BTP-Administration__ServiceInstance-create`):**

```
mcp__BTP-Administration__ServiceInstance-create(
  global_account = "<GLOBAL_ACCOUNT>",
  subaccount = "<SUBACCOUNT_ID>",
  instance_name = "cis-central",
  service_offering_name = "cis",
  service_plan_name = "central",
  parameters = {"grantType": "clientCredentials"}
)
```

**Via BTP CLI:**
```bash
btp create services/instance --subaccount <SUBACCOUNT_ID> \
  --offering-name cis --plan-name central \
  --name cis-central \
  --parameters '{"grantType": "clientCredentials"}'
```

> **Why not CF CLI?** `cf create-service cis central cis-central -c '{"grantType": "clientCredentials"}'` returns: *"client credentials authorization type is not supported in Cloud Foundry or Kubernetes (Other Environments only)"*. Use the BTP MCP Server or BTP CLI instead.

---

## Step 2: Create Service Binding and Get Credentials

The BTP MCP Server cannot create service bindings — use the BTP CLI.

```bash
# Create binding
btp create services/binding --name cis-central-binding \
  --instance-name cis-central \
  --subaccount <SUBACCOUNT_ID>

# Read credentials
btp get services/binding --name cis-central-binding \
  --subaccount <SUBACCOUNT_ID>
```

From the output, note down:

| Field in binding | Placeholder |
|---|---|
| `credentials.uaa.clientid` | `<CLIENT_ID>` |
| `credentials.uaa.clientsecret` | `<CLIENT_SECRET>` |
| `credentials.uaa.url` + `/oauth/token` | `<TOKEN_URL>` |
| `credentials.endpoints.accounts_service_url` | `<CIS_API_URL>` |

Verify `credentials.grant_type` is `client_credentials` — if it shows `user_token` the instance was created without the parameter (see Step 1).

> **Shell quoting:** `clientid` and `clientsecret` contain `!`, `|`, and `$` characters. Always wrap them in **single quotes** in shell commands.

---

## Step 3: Create Destination

**Via Claude MCP (`mcp__BTP-Administration__Destination-create`):**

```
mcp__BTP-Administration__Destination-create(
  global_account = "<GLOBAL_ACCOUNT>",
  subaccount = "<SUBACCOUNT_ID>",
  destination_configuration = {
    "Name": "<DESTINATION_NAME>",
    "URL": "<CIS_API_URL>",
    "Type": "HTTP",
    "ProxyType": "Internet",
    "Authentication": "OAuth2ClientCredentials",
    "clientId": "<CLIENT_ID>",
    "clientSecret": "<CLIENT_SECRET>",
    "tokenServiceURL": "<TOKEN_URL>"
  }
)
```

> At runtime, the BTP Destination Service fetches a bearer token from `<TOKEN_URL>` using the client credentials and injects `Authorization: Bearer <token>` into every outbound request. The token is cached until expiry — the calling app never handles tokens directly.

**Manually (BTP Cockpit):**
1. BTP Cockpit → `<SUBACCOUNT>` → Connectivity → Destinations → **New Destination**
2. Fill in the fields:

| Property | Value |
|---|---|
| Name | `<DESTINATION_NAME>` |
| URL | `<CIS_API_URL>` |
| Type | HTTP |
| ProxyType | Internet |
| Authentication | OAuth2ClientCredentials |
| Client ID | `<CLIENT_ID>` |
| Client Secret | `<CLIENT_SECRET>` |
| Token Service URL | `<TOKEN_URL>` |

3. **Save**

---

## Step 4: Verify Destination

**Via Claude MCP (`mcp__BTP-Administration__Destination-get`):**

```
mcp__BTP-Administration__Destination-get(
  global_account = "<GLOBAL_ACCOUNT>",
  subaccount = "<SUBACCOUNT_ID>",
  name = "<DESTINATION_NAME>"
)
```

→ Confirm `Authentication: OAuth2ClientCredentials`, `clientId`, and `tokenServiceURL` are present.

---

## Step 5: Test the OAuth2 Flow

Test the token fetch and API call directly with curl to verify credentials work end-to-end.

```bash
# Single quotes required — credentials contain !, |, $ characters
CLIENT_ID='<CLIENT_ID>'
CLIENT_SECRET='<CLIENT_SECRET>'
TOKEN_URL='<TOKEN_URL>'

# 1. Fetch a token
TOKEN=$(curl -s -X POST "$TOKEN_URL" \
  --data-urlencode "grant_type=client_credentials" \
  --data-urlencode "client_id=$CLIENT_ID" \
  --data-urlencode "client_secret=$CLIENT_SECRET" | jq -r '.access_token')

echo "Token length: ${#TOKEN}"   # expect ~5000+ chars, not 4 (null)

# 2. Call the CIS Accounts API — list subaccounts
curl -s "<CIS_API_URL>/accounts/v1/subaccounts" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/json" | jq '{total: .total, subaccounts: [.value[]? | {displayName, guid, region, state}]}'
```

→ Expect a JSON response listing your subaccounts. Example:
```json
{
  "subaccounts": [
    { "displayName": "trial", "guid": "<your-subaccount-id>", "region": "ap21", "state": "OK" }
  ]
}
```

---

## Reference Setup

| Variable | Value |
|---|---|
| `<GLOBAL_ACCOUNT>` | `<your-global-account-subdomain>` |
| `<SUBACCOUNT_ID>` | `<your-subaccount-id>` |
| `<DESTINATION_NAME>` | `CIS-Central` |
| `<CIS_API_URL>` | `https://accounts-service.cfapps.<REGION>.hana.ondemand.com` |
| `<TOKEN_URL>` | `https://<your-global-account-subdomain>.authentication.<REGION>.hana.ondemand.com/oauth/token` |
| `<CLIENT_ID>` | from binding `credentials.uaa.clientid` |
| `<CLIENT_SECRET>` | from binding `credentials.uaa.clientsecret` |

| Component | Status |
|---|---|
| Service instance `cis-central` (BTP-native, `grantType: clientCredentials`) | |
| Service binding `cis-central-binding` | |
| Destination `CIS-Central` | |
| API test (`/accounts/v1/subaccounts`) | |
