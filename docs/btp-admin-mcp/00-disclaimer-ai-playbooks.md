# Disclaimer


### Read Before Use

**Using an AI to administer SAP BTP is powerful and convenient — and carries real risk.** An AI operates at machine speed, acts on live systems, and does not ask twice. One misunderstood instruction can assign wrong permissions, consume quota, or make irreversible configuration changes. Read this section before you start.

These playbooks are intended **for training and exploration purposes only**. They document procedures for setting up SAP BTP services in BTP trial accounts or Enterprise Accounts. They must **not** be used on productive systems.

---

### You Are Responsible for Every Action

AI tools execute actions on your behalf in real systems. Every tool call — assigning a role, creating a service instance, enabling a capability, subscribing to a service — has immediate real-world effect. There might be no undo for BTP operations.

**You must review every step before approving it.** Do not blindly confirm tool calls. Read what the AI is about to do and verify it matches your intent.

---

### AI Can Make Mistakes

AI assistants can:
- Misinterpret instructions and take the wrong action
- Hallucinate resource names, IDs, or parameters that do not exist
- Confuse accounts, subaccounts, or regions
- Apply changes to the wrong target if context is ambiguous
- Generate plausible-looking but incorrect CLI commands or API calls

Always cross-check suggested values against the actual state of your account before confirming.

---

### Irreversible Operations

The following BTP operations are difficult or impossible to reverse:

- **Account type selection** (e.g. API Management runtime: Non-Production vs Production — cannot be changed after activation)
- **Capability activation** — some capabilities cannot be deactivated after enabling
- **Subaccount deletion** — permanently removes all resources
- **Role collection assignments** — granting excessive permissions to wrong users
- **Service subscriptions** — some consume quota immediately
- **Data deletion** — deleting service instances, keys, or bindings may destroy credentials in use

---

### Never Use on Productive Systems

These playbooks are **solely for trial and training accounts**. Do not run AI-assisted BTP administration against:

- Productive accounts
- Shared platform infrastructure
- Any account that processes real business data

---

### Credential and Secret Handling

AI conversations may display or process sensitive values including OAuth client IDs and secrets, service keys, token URLs, and user email addresses. Do not paste production credentials into an AI session. Treat any credentials that appear in a conversation as potentially exposed and rotate them if they were unintentionally shared.

---

### Scope Creep

AI tools tend to be helpful — which means they may take additional steps beyond what was asked: assigning extra role collections, creating additional service instances, or making unrequested configuration changes. Review the full action history after a session and audit what was actually done.

---

### Quota and Billing

Some operations consume quota or trigger billing: subscribing to applications, creating service instances, enabling environments, activating capabilities with consumption-based pricing. Check entitlement limits before running automated setup steps.

---

### Session Context Limitations

AI assistants work within a conversation context window. In long sessions, earlier context may be summarized or lost — the AI may lose track of which account or subaccount is active. Re-state critical context (global account, subaccount ID, region) when resuming a session or switching targets.

---

### MCP Tool Execution

When the AI coding assistant uses MCP tools (e.g. `BTP-Administration`) to interact with SAP BTP:

- Tool calls execute immediately against live systems — there is no sandbox mode
- A single tool call can assign roles, create instances, or modify subscriptions
- Approving a tool call in the permission dialog is a real authorization — treat it as such
- If you deny a tool call, verify the AI does not attempt to work around the denial

---

## Checklist Before Starting

By proceeding, you confirm that:

- [ ] I am working on a **trial or non-productive account only**
- [ ] I have reviewed the steps I am about to execute
- [ ] I understand which operations are irreversible
- [ ] I am not pasting production credentials into this session
- [ ] I will audit what was actually done after the session
- [ ] I accept that I am responsible for all changes made

---