# BTP Trial API Collection — Prerequisites & Disclaimer

Companion to `btp-trial-api-playbook.md`. Read this **first**. It lists everything you need before
running the playbook, and the ground rules for letting an AI coding assistant execute it.

> **PDF vs. Markdown — which copy to use:** A PDF export of these documents is for **humans to read and share**. An **AI coding assistant must work from the Markdown (`.md`) versions, never the PDF.** Reason: PDF text extraction reflows content — it rewraps long commands and URLs, collapses wide tables, and can drop or mangle exact whitespace and tokens like `{{process.env.*}}`. Those details are load-bearing (a reflowed URL or split token produces a wrong request), so only the Markdown preserves them faithfully. If you only have the PDF, convert it back to Markdown before running the playbook.

---

## Disclaimer — Read Before You Start

> **Using an AI coding assistant to administer SAP BTP is powerful and convenient — and carries real risk. You remain fully responsible for everything it does in your account.**

- **The AI can make mistakes.** It may misread a value, target the wrong subaccount, pick the wrong
  service plan, or run a command that does more than you intended. Treat every generated command and
  file as a *draft* until you have reviewed it.
- **Human in the loop is mandatory.** Before any operation that creates, changes, or deletes something
  (service instances, bindings, credentials, IAS applications), the assistant must show you the exact
  command or payload and wait for your explicit confirmation. Do not enable "auto-approve" / "yolo"
  modes for this workflow. If you don't understand what a step does, stop and ask before approving.
- **Never run this against a productive account.** This collection and playbook are for **trial and
  personal-development accounts only**. The credentials it creates carry broad, sometimes write-capable
  scopes. Using them against a production global account can disrupt real systems and real users.
- **You are responsible for secrets.** The setup produces real client IDs, client secrets, and access
  tokens. Keep them in the git-ignored `.env` file, never commit or paste them into shared channels,
  and rotate or delete the bindings when you are done (`btp delete services/binding` / `btp delete
  security/api-credential`).
- **Irreversible operations exist.** Deleting a subaccount, binding, or instance can be permanent and
  may not be recoverable. Read delete commands especially carefully.
- **Costs / quota.** Even in trial, creating instances consumes entitlement quota. Clean up what you
  don't need.
- **Verify, don't trust.** After the assistant reports success, check the result yourself (in the BTP
  Cockpit, via `btp` CLI, or by re-running the relevant request). "It said 200" is not the same as
  "it did the right thing."

By proceeding you accept that you — not the AI — own the outcome of every action taken in your account.

---

## Prerequisites

### 1. An SAP BTP trial account

- A free **SAP BTP trial** global account. Sign up at <https://www.sap.com/products/technology-platform/trial.html>.
- At least one **subaccount** (the trial wizard creates a `trial` subaccount for you). Note its
  **region** (e.g. `ap21`, `us10`) — the playbook needs it.
- Your **SAP Universal ID** login (used for the `--sso` browser login).
- Entitlement to the services the playbook uses (`cis`, `uas`, `service-manager`, `xsuaa`). These are
  available by default in trial; the playbook creates instances/bindings for them.

### 2. An AI coding assistant

- An agentic AI coding assistant that can read the playbook, run shell commands, and write files —
  e.g. **Claude Code**. It must support a **human-in-the-loop confirmation** step for write operations
  (see Disclaimer).
- Optional: a **browser automation tool** (e.g. Chrome DevTools MCP, Playwright MCP) connected to the
  assistant. It can semi-automate the manual IAS Admin Console step (Step 3). You still log in yourself
  (SSO/2FA) and capture the one-time client secret; treat it as convenience, not full automation.

### 3. Command-line tools

| Tool | Purpose | Verified version | Install |
|---|---|---|---|
| **btp CLI** | Create service instances, bindings, API credentials; discover account values | `2.106.1` | <https://tools.hana.ondemand.com/#cloud> |
| **Bruno CLI** (`bru`) | Run the collection from the terminal to verify requests | `4.0.0` | `npm i -g @usebruno/cli` |
| **Bruno Desktop** | GUI for the collection; **required** for the IAS OAuth2 browser flow | latest | <https://www.usebruno.com/downloads> |
| **cf CLI** (optional) | Only if you create service keys the CF-native way | `8.18.4` | <https://github.com/cloudfoundry/cli> |
| **python3** (optional) | Pretty-printing/parsing JSON in shell one-liners | `3.12` | preinstalled on macOS/Linux |

> **Login:** `btp login --url https://cli.btp.cloud.sap --subdomain <GLOBAL_ACCOUNT> --sso`
> — opens a browser for SSO; your password never touches the terminal.

### 4. Bruno specifics

- **Open-source Bruno allows max 2 workspaces.** Unlimited requires the Pro/Ultimate plan
  (<https://www.usebruno.com/pricing>). Plan your workspaces accordingly.
- For the IAS **Authorization Code** flow, set **Preferences → Use system browser for OAuth → OFF** in
  Bruno Desktop, so the embedded browser can intercept the `localhost:8686/callback` redirect.
- Secrets go in a git-ignored **`.env`** at the collection root; environment files reference them via
  `{{process.env.*}}` (see the playbook's Step 6).

### 5. IAS (only for the `trial-ias` environment)

- Access to the **IAS Admin Console** of the tenant trusted by your subaccount
  (`https://<IAS_HOST>/admin/`, trust origin `sap.custom`).
- Permission to create an OpenID Connect application and a client secret in that tenant. If you don't
  have IAS admin access, skip the `ias/` folder and the `trial-ias` environment — the rest of the
  collection works without it.

---

## What the playbook will create in your account

So you know what you're approving:

| Resource | Service / Plan | Purpose |
|---|---|---|
| `cis-central` instance + binding | `cis` / `central` | Global account, entitlements APIs |
| `cis-local` instance + binding | `cis` / `local` | Provisioning, events, subscriptions APIs |
| `uas-reporting` instance + binding | `uas` / `reporting-ga-admin` | Usage reporting APIs |
| `sm-audit` instance + binding | `service-manager` / `subaccount-audit` | Service Manager APIs |
| `xsuaa-api-cred` | XSUAA API credential | Users, roles, IdPs APIs |
| `bruno-test` IAS app + secret | IAS OpenID Connect app | OAuth2 browser-login flow |

All are created in **your trial subaccount**. Delete them when finished to free quota and reduce
credential exposure.

---

## Next step

Once the above is in place, run the playbook:

```
run this playbook btp-trial-api-playbook.md
```
