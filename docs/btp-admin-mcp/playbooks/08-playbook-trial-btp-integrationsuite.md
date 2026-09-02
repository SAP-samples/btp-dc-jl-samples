# BTP Admin — Integration Suite Setup (Trial)

Playbook for setting up SAP Integration Suite in a BTP trial subaccount.
Generic placeholders are in `<angle brackets>`. Fill in your concrete values in the table at the end.

> **Note:** This playbook targets trial accounts. For productive accounts, service and plan names differ (e.g. `integrationsuite` instead of `integrationsuite-trial`, plan `standard` instead of `trial`).

---

## How to use this playbook

Start Claude Code, then pass this playbook with your account values:

```
Follow playbook-trial-btp-integrationsuite.md to set up Integration Suite.
My global account is <GLOBAL_ACCOUNT>, subaccount ID is <SUBACCOUNT_ID>.
```

Claude reads the playbook, fills in your values, and executes each step using the BTP-Administration MCP Server. Before any create or write operation, Claude asks for confirmation.

---

## Prerequisites

### Required BTP permissions

| Role Collection | Required for |
|---|---|
| `Global Account Administrator` | Assigning entitlements |
| `Subaccount Administrator` | Subscribing apps, assigning roles |

See `10-btp-adm-mcp-general-prerequisites.md` for full details on BTP access and CLI setup.

### CLIs (optional)

All steps can be done manually via the BTP Cockpit. The BTP-Administration MCP Server covers entitlements, subscriptions, and role assignments. For `it-rt` service instances, the BTP Cockpit is required — neither CF CLI nor btp CLI work for this.

Install (macOS): `brew install cloudfoundry/tap/cf-cli@8` / `brew install btp`

For other platforms see the [CF CLI installation guide](https://docs.cloudfoundry.org/cf-cli/install-go-cli.html) and the [btp CLI installation guide](https://help.sap.com/docs/btp/sap-business-technology-platform/cli-tool).

### Activate and log in to BTP-Administration MCP Server

The automated steps require the BTP-Administration MCP Server to be registered and authenticated in Claude Code. Run once per session before starting:

```
/mcp
```

Select `BTP-Administration` → complete browser login. Access tokens expire after 30 minutes and are renewed automatically. If you see an authorization error mid-playbook, run `/mcp` again.

If BTP-Administration is not yet registered, follow `01-playbook-btp-adm-mcp-playbook.md` first.

---

## System Details

| Property | Value |
|---|---|
| Global Account | `<GLOBAL_ACCOUNT>` |
| Global Account ID | `<GLOBAL_ACCOUNT_ID>` |
| Subaccount | `<SUBACCOUNT>` |
| Subaccount ID | `<SUBACCOUNT_ID>` |
| Region | `<REGION>` (e.g. `us10`) |
| CF API Endpoint | `<CF_API>` |
| CF Org | `<CF_ORG>` |
| CF Space | `<CF_SPACE>` |
| Cockpit URL | `<COCKPIT_URL>` |

### Entitlements

| Service | Plan | Quota |
|---|---|---|
| `integrationsuite-trial` | `trial` | 1 |
| `it-rt` | `integration-flow` | 2 |
| `it-rt` | `api` | 1 |
| `apimanagement-apiportal-trial` | `on-premise-connectivity` | 1 |
| `apimanagement-apiportal-trial` | `apiportal-apiaccess` | 1 |
| `apimanagement-apiportal-trial` | `apim-as-route-service` | 1 |
| `apimanagement-devportal-trial` | `devportal-apiaccess` | 1 |

### User Role Assignments

User: `<USER>`

| Role Collection | Area |
|---|---|
| `Subaccount Administrator` | BTP |
| `Destination Administrator` | BTP |
| `Integration_Provisioner` | Integration Suite |
| `APIManagement.SelfService.Administrator` | API Management |
| `APIPortal.Administrator` | API Management |
| `AuthGroup.API.Admin` | Developer Hub |
| `AuthGroup.SelfService.Admin` | Developer Hub |
| `AuthGroup.Content.Admin` | Developer Hub |
| `AuthGroup.Site.Admin` | Developer Hub |
| `AuthGroup.ContentAuthor` | Developer Hub |
| `AuthGroup.External.Reviewer` | Developer Hub |
| `AuthGroup.API.ApplicationDeveloper` | Developer Hub |
| `PI_Administrator` | Cloud Integration |
| `PI_Integration_Developer` | Cloud Integration |

### Status

| Task | Required | Status |
|---|---|---|
| CF Environment enabled | ✅ Required | ⬜ |
| CF Space `dev` created | ✅ Required | ⬜ |
| Integration Suite subscription | ✅ Required | ⬜ |
| `Integration_Provisioner` role assigned | ✅ Required | ⬜ |
| Activate API Management capability | ✅ Required | ⬜ |
| Configure API Management Runtime | ✅ Required | ⬜ |
| Capability role collections assigned | ✅ Required | ⬜ |
| Access Developer Hub | ✅ Required | ⬜ |
| Activate Integration Cell | Optional — required for MCP server runtime | ⬜ |
| Activate Cloud Integration | Optional | ⬜ |
| `it-rt` api instance | Optional — requires Cloud Integration | ⬜ |
| `it-rt` integration-flow instance | Optional — requires Cloud Integration | ⬜ |
| API Management programmatic access (`apiportal-apiaccess`) | Currently not available on trial | — |
| Developer Hub programmatic access (`devportal-apiaccess`) | Currently not available on trial | — |

---

## Known Issues

### API Management / Developer Hub Programmatic Access Not Available on Trial

**Services affected:**
- `apimanagement-apiportal-trial` / plan `apiportal-apiaccess`
- `apimanagement-devportal-trial` / plan `devportal-apiaccess`

**Symptom:** Creating the service instance returns `403 Forbidden`.

**Root cause:** Programmatic access to API Management APIs via service key is restricted on trial accounts.

**Impact:** CRUD operations on API Management entities (API proxies, products, policies etc.) via the OData API are not available on trial.

**Workaround:** Use the Integration Suite UI for API Management operations.

---

## Step 1 — Subscribe to Integration Suite

Subscribe the subaccount to the Integration Suite application. Provisioning takes ~15–20 minutes.

- Service: `integrationsuite-trial` (trial) 
- Plan: `trial` 

> **Note:** On productive accounts the service name is `integrationsuite` and the plan is `standard`.

**Automated (via BTP-Administration MCP):**
```
SubaccountEntitlement-assign: integrationsuite-trial / trial → <SUBACCOUNT>
Subaccount-subscribe: integrationsuite-trial / trial
```

**Manual (BTP Cockpit):**
1. BTP Cockpit → `<GLOBAL_ACCOUNT>` → Entitlements → Entity Assignments → select subaccount → **Configure Entitlements** → **Add Service Plans** → `integrationsuite-trial / trial` → **Add** → **Save**
2. `<SUBACCOUNT>` → Service Marketplace → **SAP Integration Suite** → **Subscribe** → Plan `trial`
3. Wait until status is `Subscribed`

---

## Step 2 — Assign `Integration_Provisioner`

Assign only `Integration_Provisioner` before opening Integration Suite for the first time — this is the only role available at this stage and is required to access the cockpit and activate capabilities.

**Automated (via BTP-Administration MCP):**
```
RoleCollection-assign: Integration_Provisioner → <USER>
```

**Via btp CLI:**
```bash
btp assign security/role-collection "Integration_Provisioner" \
  --subaccount <SUBACCOUNT_ID> \
  --to-user <USER>
```

> **Note:** Log out and back in after assigning for the role to take effect.

---

## Step 3 — Activate Capabilities in Integration Suite

> **Manual step — no API or CLI available for this.**

1. Open the Integration Suite cockpit
2. Go to **Settings → Capabilities**
3. Activate the desired capabilities:

| Capability | Required | Notes |
|---|---|---|
| **API Management** | ✅ Core | Enables API proxies, products, Developer Hub |
| **Integration Cell** | Optional | Required to run MCP servers as iFlows on the Integration Cell runtime. **Not the same as Edge Integration Cell** — these are completely separate runtimes. |
| **Cloud Integration** | Optional | Required for `it-rt` service instances and building integration flows |

4. Follow the setup wizard for each capability

> **Note:** Capability-specific role collections (e.g. `APIPortal.Administrator`, `PI_Administrator`) are only created in the subaccount after the corresponding capability is activated. Do not try to assign them before this step.

---

## Step 4 — Check and Assign Capability Role Collections

Once capabilities are activated, check which role collections exist and assign the relevant ones.

**Check available role collections:**

Via BTP-Administration MCP: `RoleCollection-list` on the subaccount.

Via CLI:
```bash
btp list security/role-collection --subaccount <SUBACCOUNT_ID>
```

**Assign the relevant role collections** based on which capabilities were activated:

**API Management**

| Role Collection | Purpose |
|---|---|
| `APIManagement.SelfService.Administrator` | Onboard API Portal |
| `APIPortal.Administrator` | Full admin on the API portal |
| `APIPortal.Configurator` | Manage API providers, certificates, rate plans |
| `APIPortal.Developer` | Manage API proxies and products |

**Developer Hub**

| Role Collection | Purpose |
|---|---|
| `AuthGroup.API.Admin` | Manage users and apps in Developer Hub |
| `AuthGroup.SelfService.Admin` | Self-service admin on Developer Hub |
| `AuthGroup.Content.Admin` | Manage content in Developer Hub |
| `AuthGroup.Site.Admin` | Manage site updates in Developer Hub |
| `AuthGroup.ContentAuthor` | Author content in Developer Hub |
| `AuthGroup.External.Reviewer` | External reviewer in Developer Hub |
| `AuthGroup.API.ApplicationDeveloper` | Application developer in Developer Hub |

**Cloud Integration** (optional — only if capability activated)

| Role Collection | Purpose |
|---|---|
| `PI_Administrator` | Full admin for Cloud Integration |
| `PI_Integration_Developer` | Develop integration flows |
| `PI_Business_Expert` | Access business-sensitive monitoring data |
| `PI_Read_Only` | Read-only access for support users |

**Automated (via BTP-Administration MCP):**
```
RoleCollection-assign: <ROLE_COLLECTION> → <USER>
```

**Via btp CLI:**
```bash
btp assign security/role-collection "<ROLE_COLLECTION>" \
  --subaccount <SUBACCOUNT_ID> \
  --to-user <USER>
```

**Manual (BTP Cockpit):** Subaccount → Security → Users → select user → **Assign Role Collection**.

> **Note (trial accounts):** On trial, the following Developer Hub role collections are assigned automatically upon first access: `AuthGroup.API.Admin`, `AuthGroup.Content.Admin`, `AuthGroup.Site.Admin`, `AuthGroup.ContentAuthor`. `AuthGroup.SelfService.Admin` must be assigned manually beforehand.

---

## Step 5 — Configure API Management Runtime

> **Manual step — must be done via the Integration Suite cockpit UI.**

> **Caution:** The Account Type is defined at creation and **cannot be changed later**.

1. In the Integration Suite cockpit, go to **Settings → Runtimes**
2. Select the **Configure** tab under API Management
3. Choose Account Type: **Non-Production** (trial) or **Production**
4. Enter a Host Alias for the Virtual Host (max 63 characters)
   - API proxies will be available at: `https://<virtualHost>.apimanagement.hana.ondemand.com`
5. Enter Notification Contact email(s)
6. Choose **Activate** → **Confirm**
7. Wait for all indicators to turn green

> **Trial behavior:** No virtual host, account type, or notification email configuration required — just activate. After successful provisioning you will be logged out automatically.

> **Integration Cell:** As of 2026, trial accounts provision API Management on the **Integration Cell** runtime — SAP's next-generation unified runtime replacing the legacy Apigee-based runtime.

---

## Step 6 — Access Developer Hub

Developer Hub is accessed via the product switcher in the Integration Suite cockpit:

1. Open the Integration Suite cockpit
2. Click the **9-dot grid icon** (product switcher) in the top-right toolbar
3. Select **Developer Hub**
4. On first access, Developer Hub runs a one-time setup. When complete, it shows **"Developer Hub setup complete! Please login again to continue."** — click **Logout** and log back in.
5. After re-login, open Developer Hub again via the product switcher — it will now load the full catalog.

> **Note:** Developer Hub must be activated as a sub-capability of API Management first. The Developer Hub checkbox is **not selected by default** when activating API Management — you must explicitly enable it under **Settings → Capabilities → API Management**.

---

## Step 7 — Create CF Space and Verify CF Marketplace

> **Optional — only required if Cloud Integration is activated.**

**Check existing CF spaces first:**

```bash
cf login -a <CF_API> --sso
cf target -o <CF_ORG>
cf spaces
```

> **Trial behavior:** Trial accounts come with a pre-configured CF space (`dev`). Verify it exists before creating a new one.

**Create a CF space (if `dev` does not exist):**

```bash
cf login -a <CF_API> --sso
# Get passcode at: <CF_LOGIN_URL>
cf target -o <CF_ORG>
cf create-space dev
cf target -s dev
```

> **Note:** `cf login --sso` requires an interactive terminal. Use a separate terminal and paste the passcode manually.

**Manual (BTP Cockpit):** Subaccount → Cloud Foundry → Spaces → **Create Space**.

**Verify `it-rt` is available in the CF marketplace:**

```bash
cf marketplace -e it-rt
```

Expected output:
```
plan               description
integration-flow   Creates an OAuth client for Cloud Integration runtime access
api                Creates an OAuth client for Cloud Integration public API access
```

---

## Step 8 — Create Process Integration Runtime Service Instance

> **Optional — only applicable if Cloud Integration is activated.**

`it-rt` provides OAuth credentials to access the **Cloud Integration API** programmatically (packages, artifacts, message logs, data stores, access policies etc.). This is **not** the API Management API — programmatic access to API Management is currently not available on trial (see "Known Issues").

> **Note:** Both `cf create-service` and `btp create services/instance` fail for `it-rt` with "invalid input parameters". Create the instance **manually in the BTP Cockpit**:
> **Subaccount → Services → Service Marketplace → Process Integration Runtime → Create** — select plan `api`.

**Instance parameters (use all roles for full access):**

```json
{
    "grant-types": ["client_credentials"],
    "redirect-uris": [],
    "token-validity": 43200,
    "roles": [
        "WorkspacePackagesRead",
        "WorkspacePackagesConfigure",
        "WorkspacePackagesEdit",
        "WorkspaceArtifactsDeploy",
        "WorkspacePackagesTransport",
        "WorkspaceArtifactLocksRead",
        "WorkspaceArtifactLocksDelete",
        "MonitoringDataRead",
        "MonitoringArtifactsDeploy",
        "AccessAllAccessPoliciesArtifacts",
        "AccessPoliciesRead",
        "AccessPoliciesEdit",
        "DataStoresAndQueuesRead",
        "DataStoresAndQueuesDelete",
        "DataStoresAndQueuesConfig",
        "DataStorePayloadsRead",
        "MessagePayloadsRead",
        "MessageProcessingLocksRead",
        "TraceConfigurationRead",
        "TraceConfigurationEdit",
        "QueuesActivate",
        "QueuesRetry",
        "CredentialsEdit",
        "SecurityMaterialEdit",
        "UserRolesEdit"
    ]
}
```

**To update roles on an existing instance:**
```bash
cf update-service <INSTANCE_NAME> -c '{"grant-types":["client_credentials"],"roles":["MonitoringDataRead","AccessAllAccessPoliciesArtifacts"]}'
```

### Available roles for `it-rt` / `api` plan

**Design**

| Role | Purpose |
|---|---|
| `WorkspacePackagesRead` | Read integration packages |
| `WorkspacePackagesConfigure` | Configure packages |
| `WorkspacePackagesEdit` | Edit packages |
| `WorkspaceArtifactsDeploy` | Deploy artifacts |
| `WorkspacePackagesTransport` | Export/import packages via transport |
| `WorkspaceArtifactLocksRead` | Read design-time artifact locks |
| `WorkspaceArtifactLocksDelete` | Delete design-time artifact locks |

**Monitor**

| Role | Purpose |
|---|---|
| `MonitoringDataRead` | Monitor overview, message processing logs, deployed artifacts |
| `MonitoringArtifactsDeploy` | Edit number ranges, undeploy integration flows |
| `AccessAllAccessPoliciesArtifacts` | Override access policies (bypass data protection) |
| `AccessPoliciesRead` | View access policies |
| `AccessPoliciesEdit` | Create/edit access policies |
| `DataStoresAndQueuesRead` | View data stores and queues |
| `DataStoresAndQueuesDelete` | Delete data store entries/queues |
| `DataStoresAndQueuesConfig` | Configure data stores and queues |
| `DataStorePayloadsRead` | Read message payloads from data stores |
| `MessagePayloadsRead` | Read payloads from message store |
| `MessageProcessingLocksRead` | View runtime processing locks |
| `TraceConfigurationRead` | View trace configuration |
| `TraceConfigurationEdit` | Enable/disable trace |
| `QueuesActivate` | Activate/deactivate queues |
| `QueuesRetry` | Retry queues |
| `CredentialsEdit` | Add/edit credentials and security material |
| `SecurityMaterialEdit` | Manage keystores and PGP keyrings |
| `UserRolesEdit` | Create/edit user roles |

---

## Step 9 — Create Service Key

> **Optional — only applicable if Cloud Integration is activated.**

`it-rt` uses **service keys** (not bindings) for credentials.

```bash
cf target -o <CF_ORG> -s <CF_SPACE>
cf create-service-key <INSTANCE_NAME> <INSTANCE_NAME>-key
cf service-key <INSTANCE_NAME> <INSTANCE_NAME>-key
```

> **Note:** After updating roles on an instance, delete and recreate the service key — new scopes are only reflected in a fresh key:
> ```bash
> cf delete-service-key <INSTANCE_NAME> <INSTANCE_NAME>-key -f
> cf create-service-key <INSTANCE_NAME> <INSTANCE_NAME>-key
> ```

The service key contains:
- `clientid` — OAuth client ID
- `clientsecret` — OAuth client secret
- `tokenurl` — URL to obtain an access token (`POST /oauth/token`)
- `url` — Base URL for the Cloud Integration OData API (`/api/v1/...`)

**Test the API:**
```bash
TOKEN=$(curl -s -X POST "<tokenurl>" \
  --data-urlencode "grant_type=client_credentials" \
  --data-urlencode "client_id=<clientid>" \
  --data-urlencode "client_secret=<clientsecret>" \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

curl -s "<url>/api/v1/IntegrationPackages" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/json" | python3 -c "
import sys,json
d=json.load(sys.stdin)
for p in d['d']['results']:
    print(p['Id'], '-', p['Name'])"
```

---

## Note: Integration Cell vs. Edge Integration Cell

These are two completely separate runtimes — do not confuse them:

- **Edge Integration Cell** — a hybrid deployment option where you run integrations in your own private Kubernetes cluster. Design and monitor in the cloud, execute locally. **This playbook does not cover Edge Integration Cell setup.**

- **Integration Cell** — one of the available runtimes for the API Management capability on BTP, alongside Classic API Management. As of 2026, trial accounts provision Integration Cell for API Management.

---

## Subaccount-specific values

| Placeholder | Value |
|---|---|
| `<GLOBAL_ACCOUNT>` | _(Global Account subdomain — visible top-left in BTP Cockpit)_ |
| `<GLOBAL_ACCOUNT_ID>` | |
| `<SUBACCOUNT>` | _(display name)_ |
| `<SUBACCOUNT_ID>` | |
| `<REGION>` | e.g. `us10`, `eu10` |
| `<CF_API>` | `https://api.cf.<REGION>-001.hana.ondemand.com` |
| `<CF_LOGIN_URL>` | `https://login.cf.<REGION>-001.hana.ondemand.com/passcode` |
| `<CF_ORG>` | |
| `<CF_SPACE>` | `dev` |
| `<COCKPIT_URL>` | `https://account.hanatrial.ondemand.com/trial/#/globalaccount/<GLOBAL_ACCOUNT_ID>/accountModel` |
| `<USER>` | |
| `<INSTANCE_NAME>` | e.g. `it-rt-api` |
