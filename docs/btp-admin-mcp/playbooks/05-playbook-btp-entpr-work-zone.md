# SAP Build Work Zone Standard Edition — Setup Playbook

> **How to run this playbook:** In Claude Code, type:
> ```
> run this playbook btp-enterprise-acc-work-zone-playbook.md
> ```
> Claude will read the playbook, execute each step using the `BTP-Administration` MCP tools, and report progress. Some steps require manual action in the BTP Cockpit — these are clearly marked.

## Context

This playbook sets up **SAP Build Work Zone, standard edition** in an enterprise BTP subaccount.

---

## Prerequisites

- Global Account Admin role
- Subaccount Admin role on the target subaccount

---

## Step 1: Connect via BTP-Administration MCP Server and Select Subaccount

Check whether the `BTP-Administration` MCP server is connected (authentication is handled automatically). List the available global accounts and identify the correct one:

```
mcp__BTP-Administration__GlobalAccount-list
```

Then list subaccounts under the global account:

```
mcp__BTP-Administration__Subaccount-list  global_account: <ga-subdomain>
```

- If only one subaccount exists, use it automatically — no need to ask.
- If multiple subaccounts exist, ask the user which one to target before proceeding.

Note the **subaccount ID** — it is required for all subsequent steps.

---

## Step 2: Verify Entitlements and Existing Subscriptions

Before making any changes, verify what is already in place.

**Check entitlements:**

```
mcp__BTP-Administration__SubaccountServiceQuota-list  subaccount_id: <subaccount-id>
```

Check that the subaccount has:

| Service | Service Technical Name | Plan | Purpose |
|---|---|---|---|
| SAP Build Work Zone, standard edition | `build-workzone-standard` | `standard` | Subscription (user access) |
| SAP Build Work Zone, standard edition | `SAPLaunchpad` | `standard (Application)` | Service instance for API access |
| SAP Build Work Zone, standard edition | `build-workzone-standard` | `foundation` | Auto-assigned |

If the `standard` plan is missing, assign it via the BTP Cockpit: **Global Account → Entitlements → Configure Entitlements → select subaccount → Add `SAP Build Work Zone, standard edition` → plan `standard` → Save**.

**Check for an existing subscription:**

```
mcp__BTP-Administration__Subscription-list  subaccount_id: <subaccount-id>
```

Look for `SAPLaunchpadSMS` with status `SUBSCRIBED`. If already subscribed, skip to Step 5 (role assignments).

**Check for an existing service instance:**

```
mcp__BTP-Administration__ServiceInstance-list  subaccount: <subaccount-id>
```

Look for any instance tied to `build-workzone-standard`. If one exists, note its name and plan.

> **Before proceeding:** Confirm the entitlement is present and note whether a subscription or instance already exists. This determines which steps below are needed.

---

## Step 3: Check IAS Trust Configuration

> **Note:** If Step 2 confirmed a subscription already exists, skip to Step 5 (role assignments).

Before subscribing, verify IAS trust is configured — SAP Build Work Zone requires IAS authentication:

```
mcp__BTP-Administration__TrustConfiguration-list  subaccount_id: <subaccount-id>
```

Look for a custom IAS entry (origin `sap.custom`) alongside the default `sap.default`.

**If no IAS trust is configured:** establish trust manually in the BTP Cockpit: **Security → Trust Configurations → Establish Trust**, select the IAS tenant, and keep all default values. SAP Build Work Zone will fail to provision without IAS trust.

> **No IAS tenant available?** Do NOT provision one via MCP on an enterprise account — this may incur billing costs. Provision Cloud Identity Services manually in the BTP Cockpit: **Subaccount → Services → Service Marketplace → Cloud Identity Services → Create**, plan `default`. After provisioning, SAP sends an activation email to the account owner. The user **must click the activation link** and set a password before the IAS tenant can be used. After activation, the IAS Admin UI is accessible at `https://<ias-tenant-id>.accounts.ondemand.com/admin/`. Establishing trust itself cannot be done via MCP — it is a manual step in the BTP Cockpit.

---

## Step 4: Subscribe to SAP Build Work Zone

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

> **Note:** The internal app name for the subscription is `SAPLaunchpadSMS` (not `SAPLaunchpad`, which is the service instance offering).

---

## Step 5: Assign Role Collections

**SAP BTP has two user types that need separate role assignments:**

| User Type | Origin | Used for |
|---|---|---|
| Platform User | `sap.default` | BTP Cockpit, CLI, admin tasks |
| Business User | `sap.custom` (IAS) | Actual applications: Work Zone, Joule |

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
| `Launchpad_Admin` | Manage Work Zone content and sites |
| `Launchpad_Advanced_Theming` | Customize themes (optional) |

> **Important:** Always use `origin: sap.custom` for Work Zone access. Assigning only to `sap.default` will not grant access when the user logs in via IAS.
>
> **Note on user creation:** Under IAS, the user record in the subaccount is normally created on first login. The BTP API may create a shadow user when a role is assigned before first login — the assignment will be in place, but the user should log in to Work Zone once to fully activate it.
>
> **After assigning roles:** log out and log back in — role assignments only take effect after a new login session.

---

## Step 6: Create a Work Zone Site

1. Open Work Zone: BTP Cockpit → Subaccount → **Instances and Subscriptions** → **SAP Build Work Zone** → **Go to Application**
2. In the Site Manager: **Create New Site**
3. Enter a site name and configure it
4. Add apps and content as needed

---

## Step 7: Create Work Zone Service Instance _(optional, for API access)_

**When do you need a service instance?**

| | Subscription | Service Instance |
|---|---|---|
| What it is | The Work Zone application itself | API access credentials |
| Work Zone Site Manager | ✅ Yes | Not needed |
| Joule integration | ✅ Yes | Not needed |
| Add apps and tiles | ✅ Yes | Not needed |
| API access | ❌ No | ✅ Yes |
| External app calling Work Zone APIs | ❌ No | ✅ Yes |
| CI/CD automation of Work Zone content | ❌ No | ✅ Yes |

**For this Joule setup: the service instance is NOT needed.** The subscription alone is sufficient.

If you do need it:

> **Note:** This step cannot be done via MCP. The instance must be created with **CF runtime** — create it manually in the BTP Cockpit: **Services → Service Marketplace → SAP Build Work Zone, standard edition → Create**, and select your CF space in the dialog.

---

## Lessons Learned

| Issue | Root Cause | Fix |
|---|---|---|
| Work Zone opens but user has no access | Role only assigned to platform user (`sap.default`), not business user (`sap.custom`) | Assign `Launchpad_Admin` to `sap.custom` origin |
| Roles don't take effect immediately | BTP token still contains old role set | Log out and log back in to get a new token with updated roles |
| No IdP selection screen at Work Zone login | `sap.default` trust configuration has `available_for_user_logon: false` | In BTP Cockpit → Subaccount → Security → Trust Configuration → `sap.default` → set `available_for_user_logon: true` |
| Work Zone subscription deleted accidentally | Subscription was assumed tied to a CF service instance | Work Zone subscription is independent of CF — can be re-subscribed without side effects |
