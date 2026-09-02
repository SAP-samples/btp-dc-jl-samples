# BTP Account Administration — REST APIs Playbook

Reference for the direct SAP BTP REST APIs — for programmatic access, scripts, and CI/CD.
Generic placeholders are in `<angle brackets>`, concrete values for your subaccount have to be provided in addition.

> **When to use REST APIs instead of MCP Server?** When no AI client is available, for automation/scripts, or for operations not covered by the MCP server (e.g. Usage Reporting, XSUAA). With AI coding assistant (e.g. Claude Code): prefer the MCP server.

For more information, see [SAP Help Portal](https://help.sap.com/docs/btp/sap-business-technology-platform/account-administration-using-apis).

---

## Prerequisites: CLI Login

### BTP CLI

The BTP CLI operates at global account level and is used for BTP-native service instances (required for `grantType: clientCredentials`).

```bash
btp login --url https://cli.btp.cloud.sap --subdomain <GLOBAL_ACCOUNT> --sso
```

The `--sso` flag triggers a **Single Sign-On** login using the **OAuth2 Device Authorization Grant** (RFC 8628): instead of entering your SAP credentials in the terminal, a browser window opens where you authenticate with your SAP Universal ID (the same account you use for the BTP Cockpit). The CLI then receives a token from the browser session — your password never touches the terminal. After login, set the target subaccount:

```bash
btp target --subaccount <SUBACCOUNT_ID>
```

### CF CLI

The CF CLI operates at Cloud Foundry space level and is used for CF-space service instances and service keys.

1. Get a one-time passcode from your browser:
   `https://login.cf.<region>.hana.ondemand.com/passcode`
   (e.g. `https://login.cf.ap21.hana.ondemand.com/passcode` for AP21)

2. Log in:
   ```bash
   cf login -a https://api.cf.<region>.hana.ondemand.com \
     -o <CF_ORG> -s <CF_SPACE> --sso
   ```
   Paste the passcode when prompted.

   The `--sso` flag works the same way as for the BTP CLI — **OAuth2 Device Authorization Grant** (RFC 8628): you authenticate in the browser and paste a short-lived one-time passcode (called a **Temporary Authentication Code**) into the terminal instead of your password.

> **Why two CLIs?** BTP CLI manages the BTP account layer (subaccounts, entitlements, BTP-native service instances). CF CLI manages the Cloud Foundry runtime layer (apps, CF-space service instances, service keys). They authenticate independently — logging into one does not log you into the other.

---

## Subaccount-specific Values 

| Variable | Value |
|---|---|
| `<GLOBAL_ACCOUNT>` | `your global account name` |
| `<SUBACCOUNT_ID>` | `your subaccount ID` |
| `<UAA_URL>` | `https://<your-subaccount-subdomain>.authentication.eu10.hana.ondemand.com` |
| `<ACCOUNTS_SERVICE_URL>` | from `cis` service key: `endpoints.accounts_service_url` |
| `<ENTITLEMENTS_SERVICE_URL>` | from `cis` service key: `endpoints.entitlements_service_url` |
| `<PROVISIONING_SERVICE_URL>` | from `cis` service key: `endpoints.provisioning_service_url` |

> **Prerequisite:** A `cis` instance must exist. The service key provides all endpoint URLs.

---

## API Overview

| API / Microservice | Description | Region |
|---|---|---|
| `Accounts` | Manage directories and subaccounts | Central region (eu10) |
| `Entitlements` | Manage product entitlements and assignments | Central region (eu10) |
| `Provisioning` | Manage environment instances, app subscriptions, services | All regions |
| `Events` | Query administrative events | All regions |

> For `Accounts` and `Entitlements`, the endpoint must point to the central region (eu10 for Europe).

Complete API specifications: [SAP Business Accelerator Hub](https://api.sap.com/) → Core Services for SAP BTP

---

## Step 1: Create cis Instance and Retrieve Service Key

If not already present:

**Client Credentials grant type (recommended for scripts and automation):**

The instance must be created with `{"grantType": "clientCredentials"}`. Without this parameter, the service defaults to Password grant type and the token only carries `uaa.resource` scope — causing a `403 insufficient_scope` on every CIS API call.

> **CF CLI limitation:** `cf create-service` does not support `grantType: clientCredentials` for `cis` — use the BTP CLI instead:

```bash
btp create services/instance --subaccount <SUBACCOUNT_ID> \
  --offering-name cis --plan-name central \
  --name cis-central \
  --parameters '{"grantType": "clientCredentials"}'

btp create services/binding --name cis-central-binding \
  --instance-name cis-central \
  --subaccount <SUBACCOUNT_ID>

btp get services/binding --name cis-central-binding \
  --subaccount <SUBACCOUNT_ID>
```

**Password grant type (interactive use only):**

If you only need Password grant type (interactive login with user/password), CF CLI works:

```bash
cf create-service cis central cis-central
cf create-service-key cis-central cis-central-key
cf service-key cis-central cis-central-key
```

Relevant fields from the service key:
```json
{
  "endpoints": {
    "accounts_service_url": "https://accounts-service.cfapps.eu10.hana.ondemand.com",
    "entitlements_service_url": "https://entitlements-service.cfapps.eu10.hana.ondemand.com",
    "provisioning_service_url": "https://provisioning-service.cfapps.eu10.hana.ondemand.com"
  },
  "uaa": {
    "clientid": "<CLIENT_ID>",
    "clientsecret": "<CLIENT_SECRET>",
    "url": "<UAA_URL>"
  }
}
```

---

## Step 2: Retrieve Token

### Service Plans

| Plan | Usage | Grant Type |
|---|---|---|
| `central` | Manage global accounts, subaccounts, directories, entitlements | Password or Client Credentials |
| `local` | Manage environments and app subscriptions | Password or Client Credentials |
| `central-viewer` | Read-only access to global account structure | Client Credentials only |
| `local-viewer` | Read-only access to environments and subscriptions | Client Credentials only |

**Password Grant Type (macOS/Linux):**
```bash
TOKEN=$(curl -s -L -X POST '<UAA_URL>/oauth/token' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -u '<clientid>:<clientsecret>' \
  -d 'grant_type=password' \
  -d 'username=<user email>' \
  -d 'password=<password>' | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")
echo "Token set"
```

**Client Credentials Grant Type:**
```bash
TOKEN=$(curl -s -L -X POST '<UAA_URL>/oauth/token' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -u '<clientid>:<clientsecret>' \
  -d 'grant_type=client_credentials' | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")
echo "Token set"
```

> **2FA:** Append the passcode to the password (e.g. `Password1234`). Relevant for Password Grant Type only.

---

## Step 3: Execute API Calls

### Create Subaccount

```bash
curl -s -X POST "<ACCOUNTS_SERVICE_URL>/accounts/v1/subaccounts" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"displayName": "<name>", "region": "<region>", "subdomain": "<subdomain>"}' | python3 -m json.tool
```

→ Responds with `202 Accepted` (asynchronous job) — see Step 4.

### Assign Entitlement

```bash
curl -s -X PUT "<ENTITLEMENTS_SERVICE_URL>/entitlements/v1/subaccountServicePlans" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"servicePlans": [{"serviceName": "<service>", "servicePlanName": "<plan>", "assignedServiceInstances": {"amount": 1}}], "subaccountGUID": "<SUBACCOUNT_ID>"}' | python3 -m json.tool
```

### Subscribe to App

```bash
curl -s -X POST "<PROVISIONING_SERVICE_URL>/saas-manager/v1/applications/<app>/subscription" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"planName": "<plan>"}' | python3 -m json.tool
```

---

## Step 4: Check Asynchronous Jobs (202)

Many write operations respond with `202 Accepted`. Check status via the `Location` header:

```bash
curl -s "<location_url>" \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
```

Response:
```json
{
  "status": "IN_PROGRESS | COMPLETED | FAILED",
  "description": "<optional>"
}
```

---

## Error Format and Codes

```json
{
  "error": {
    "code": 11000,
    "message": "Request payload is invalid",
    "target": "/accounts/v1/subaccounts/...",
    "details": [{"code": 11001, "message": "description: length must be between 0 and 255"}]
  }
}
```

| Code Range | Meaning |
|---|---|
| `10XXX` | Server errors |
| `11XXX` | General request errors |
| `12XXX` | General operations failures |
| `2XXXX` | Accounts service errors |
| `3XXXX` | Entitlements service errors |
| `40XXX` | Tenants operations failures |
| `41XXX` | Environment instances operations failures |
| `42XXX` | Provisioning service errors |
| `8XXXX` | Asynchronous jobs failures |
| `9XXXX` | SaaS Provisioning service errors |
| `10XXXX` | Events service errors |

> **Rate Limiting:** On `HTTP 429` the rate limit has been exceeded (Error Code `11006`). Wait briefly and retry.

---

## Usage Data Management APIs

For resource and cost reporting.

**Base URI:** `https://uas-reporting.cfapps.<landscape domain>/reports/v1`

| Endpoint | Description |
|---|---|
| `/monthlyUsage` | Monthly usage of a global account |
| `/monthlyDirectoryUsage` | Monthly usage of a directory |
| `/subaccountUsage` | Usage of a subaccount |
| `/monthlySubaccountsCost` | Monthly costs of all subaccounts (CPEA) |
| `/cloudCreditsDetails` | Cloud credits history |

**Auth:** OAuth Client Credentials, Service Plan `reporting-ga-admin` (technical name: `uas`)

---

## Authorization and Trust Management APIs (XSUAA)

For user management, roles, and role collections in Cloud Foundry subaccounts.

**API URL:** `https://api.authentication.<region>.hana.ondemand.com`

**Retrieve credentials** (via btp CLI):
```bash
btp create security/api-credential --name my-credential
```

**Retrieve token:**
```bash
curl --request POST \
  --url 'https://<subdomain>.authentication.<region>.hana.ondemand.com/oauth/token' \
  --header 'Accept: application/json' \
  --data 'client_id=<client_id>' \
  --data 'client_secret=<secret>' \
  --data 'grant_type=client_credentials' \
  --data 'response_type=token'
```

| API | Description |
|---|---|
| User Management (SCIM) | Manage users and shadow users, assign roles |
| Authorization | Manage roles, role templates, role collections |
| Identity Provider Management | Manage IDPs in Cloud Foundry |
| Security Settings | Access token validity, signing keys |
