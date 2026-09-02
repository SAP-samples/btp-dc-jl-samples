# Trial BTP SAP Build Work Zone Playbook

> **How to run this playbook:** In Claude Code, type:
> ```
> run this playbook trial-btp-workzone-playbook.md
> ```
> Claude will read the playbook, execute each step using the `BTP-Administration` MCP tools, and report progress. Some steps require manual action in the BTP Cockpit — these are clearly marked.

## System Details

| Property | Value |
|---|---|
| Global Account | `<your-global-account-subdomain>` |
| Global Account ID | `<your-global-account-id>` |
| Subaccount | `trial` |
| Subaccount ID | `<your-subaccount-id>` |
| Region | `<your-region>` |
| Service Technical Name | `build-workzone-standard` |
| Plan | `standard` |
| Auth | IAS |
| IAS Tenant | `<your-ias-tenant>.accounts.ondemand.com` |

### Entitlements

| Service | Service Technical Name | Plan | Quota | Purpose |
|---|---|---|---|---|
| SAP Build Work Zone, standard edition | `build-workzone-standard` | `standard` | 100 shared units | Subscription (user access) |
| SAP Build Work Zone, standard edition | `SAPLaunchpad` | `standard (Application)` | 1 shared unit | Service instance for API access |

### User Role Assignments

| Role Collection | Purpose |
|---|---|
| `Launchpad_Admin` | Assign after subscription completes |

### Status

| Task | Status |
|---|---|
| Establish BTP-Administration connection and select subaccount | ✅ Step 1 |
| Check entitlements and existing subscriptions/instances | ✅ Step 2 |
| Check IAS trust configuration | ✅ Step 3 — IAS trust established |
| Provision IAS tenant (Cloud Identity Services) | ✅ Step 3a — subscribed, activated |
| Work Zone subscription (`build-workzone-standard`) | ✅ Step 4 — SUBSCRIBED |
| Role collections assigned | ✅ Step 5 — `Launchpad_Admin` assigned via `sap.custom` |
| Create Work Zone site | ⏸ Step 6 — optional |

---

## Playbook

### Step 1 — Connect via BTP-Administration MCP Server and Select Subaccount

Check whether the `BTP-Administration` MCP server is connected (authentication is handled automatically). List the available global accounts and identify the trial account:

```
mcp__BTP-Administration__GlobalAccount-list
```

Then list subaccounts under the trial global account:

```
mcp__BTP-Administration__Subaccount-list  global_account: <trial-ga-subdomain>
```

- If only one subaccount exists, use it automatically — no need to ask.
- If multiple subaccounts exist, ask the user which one to target before proceeding.

Note the **subaccount ID** — it is required for all subsequent steps.

> **Trial Cockpit URL:** `https://cockpit.hanatrial.ondemand.com/trial/` — direct link pattern: `https://cockpit.hanatrial.ondemand.com/trial/#/globalaccount/<global-account-id>/subaccount/<subaccount-id>/subscriptions`

---

### Step 2 — Check Entitlements and Existing Subscriptions

Before making any changes, verify what is already in place.

**Check entitlements for Work Zone:**

```
mcp__BTP-Administration__SubaccountServiceQuota-list  subaccount_id: <subaccount-id>
```

Look for `build-workzone-standard` with plan `standard`. If the entitlement is missing, it must be assigned at the global account level before proceeding.

**Check for an existing subscription:**

```
mcp__BTP-Administration__Subscription-list  subaccount_id: <subaccount-id>
```

Look for `build-workzone-standard` with plan `standard`. If a subscription already exists and its status is `SUBSCRIBED`, skip to Step 4.

**Check for an existing service instance:**

```
mcp__BTP-Administration__ServiceInstance-list  subaccount: <subaccount-id>
```

Look for any instance tied to `build-workzone-standard`. If one exists, note its name and plan.

> **Before proceeding:** Confirm the entitlement is present and note whether a subscription or instance already exists. This determines which steps below are needed.

---

### Step 3 — Check IAS Trust Configuration

> **Note:** If Step 2 confirmed a subscription already exists, skip to Step 4 (role assignments).

Check whether the subaccount already has trust configured to an SAP Cloud Identity Services (IAS) tenant:

```
mcp__BTP-Administration__TrustConfiguration-list  subaccount_id: <subaccount-id>
```

Look for a custom IAS entry (type `SAP Cloud Identity Services`) alongside the default `sap.default`.

**If IAS trust is already configured:** note the tenant hostname and proceed to Step 4.

**If no IAS trust is configured:** IAS trust must be established before subscribing. SAP Build Work Zone requires IAS — without it, the subscription will fail.

---

#### Step 3a — Provision an IAS Tenant (if none exists)

> **Always ask the user for confirmation before executing this step.**

If no IAS tenant is available, subscribe to Cloud Identity Services to provision one:

```
mcp__BTP-Administration__Subaccount-subscribe
  app_name: sap-identity-services-onboarding
  plan_name: default
  subaccount_id: <subaccount-id>
```

After the subscription completes, the IAS tenant hostname will be visible in the BTP Cockpit under **Security → Trust Configurations**. Note it — it is needed for the next step.

> **Email activation required:** After provisioning, SAP sends an activation email to the account owner. The user **must click the activation link** in that email and set a password before the IAS tenant can be used. Trust configuration will fail if the account has not been activated yet.
>
> After activation, the IAS Admin UI is accessible at `https://<ias-tenant-id>.trial-accounts.ondemand.com/admin/`

> **Note:** Establishing trust cannot be done via MCP — this is a manual step in the BTP Cockpit: **Security → Trust Configurations → Establish Trust**, then select the IAS tenant. Keep all default values unless you have a specific reason to change them.

---

### Step 4 — Subscribe to SAP Build Work Zone

Subscribe the subaccount to SAP Build Work Zone, standard edition:

```
mcp__BTP-Administration__Subaccount-subscribe
  app_name: SAPLaunchpadSMS
  plan_name: standard
  subaccount_id: <subaccount-id>
```

Wait for subscription state to become `SUBSCRIBED`:

```
mcp__BTP-Administration__Subscription-get
  app_name: SAPLaunchpadSMS
  plan_name: standard
  subaccount_id: <subaccount-id>
```

> **Note:** The `SAPLaunchpad` / `standard (Application)` entitlement is for creating a **service instance** for API access (service bindings/keys). It is not needed for the subscription itself.

---

### Step 5 — Assign Required Role Collections

After the subscription is provisioned, assign the following role collections.

**SAP BTP has two user types that need separate role assignments:**

| User Type | Origin | Used for |
|---|---|---|
| Platform User | `sap.default` | BTP Cockpit, CLI, admin tasks |
| Business User | `sap.custom` (IAS) | Actual applications: Work Zone |

Assign `Launchpad_Admin` to the **business user** (IAS origin — required for Work Zone access):

```
mcp__BTP-Administration__RoleCollection-assign
  role_collection_name: Launchpad_Admin
  user_email: <user-email>
  subaccount_id: <subaccount-id>
  origin: sap.custom
```

Optionally also assign to the **platform user** (`sap.default`) for BTP Cockpit admin access:

```
mcp__BTP-Administration__RoleCollection-assign
  role_collection_name: Launchpad_Admin
  user_email: <user-email>
  subaccount_id: <subaccount-id>
  origin: sap.default
```

| Role Collection | Purpose |
|---|---|
| `Launchpad_Admin` | Full admin — create and manage Work Zone sites |

> **Important:** SAP Build Work Zone uses IAS as its authentication provider. Role collections must be assigned under the **IAS origin** (`sap.custom`), not the default IdP (`sap.default`). Assigning under `sap.default` will not grant access when the user logs in via IAS.
>
> **Note on user creation:** Under IAS, the user record in the subaccount is normally created on first login. The BTP API may create a shadow user when a role is assigned before first login — the assignment will be in place, but the user should log in to Work Zone once to fully activate it.
>
> **After assigning roles:** log out and log back in — role assignments only take effect after a new login session.

---

### Step 6 — Create a Work Zone Site _(optional)_

1. Open SAP Build Work Zone from the BTP Cockpit subscriptions
2. Go to **Site Manager**
3. Choose **Create New Site**
4. Enter a site name and configure it
5. Add apps and content as needed
