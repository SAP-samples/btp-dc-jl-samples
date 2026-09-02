# BTP Admin — AI Core + AI Launchpad Setup

Playbook for setting up AI Core + AI Launchpad in a BTP enterprise subaccount.
Generic placeholders are in `<angle brackets>`. Fill in your concrete values in the table at the end.

> **Note:** AI Core is only available in enterprise (paid) BTP accounts — not in trial accounts.

---

## How to use this playbook

Start Claude Code, then pass this playbook with your account values:

```
Follow btp-enterprise-aicore-playbook.md to set up AI Core and AI Launchpad.
My global account is <GLOBAL_ACCOUNT>, subaccount ID is <SUBACCOUNT_ID>.
```

Claude reads the playbook, fills in your values, and executes each step using the BTP-Administration MCP Server. Before any create or write operation, Claude asks for confirmation.

---

## Prerequisites

### Required BTP permissions

| Role Collection | Required for |
|---|---|
| `Global Account Administrator` | Assigning entitlements (Steps 1, 3) |
| `Subaccount Administrator` | Creating service instances, subscribing apps, assigning roles (Steps 2–4) |

See `10-btp-adm-mcp-general-prerequisites.md` for full details on BTP access and CLI setup.

### CLIs (optional)

All steps in this playbook can be done manually via the BTP Cockpit. The BTP-Administration MCP Server covers entitlements, service instances, subscriptions, and role assignments — but has no tool for creating or reading service keys and bindings. For those steps, the options are CLI or BTP Cockpit:

| CLI | Used in | MCP available? | Manual alternative |
|---|---|---|---|
| CF CLI (`cf`) | Step 2 (create CF service instance) | ✅ `ServiceInstance-create` covers BTP-level | BTP Cockpit → Instances and Subscriptions → Create |
| CF CLI (`cf`) | Step 5 (service key), Step 9 (verify) | ❌ no ServiceKey tool | BTP Cockpit → Instances and Subscriptions → Service Binding |
| btp CLI (`btp`) | Step 5 (service binding) | ❌ no ServiceBinding tool | BTP Cockpit → Instances and Subscriptions → Service Binding |

Install (macOS): `brew install cloudfoundry/tap/cf-cli@8` / `brew install btp`

For other platforms see the [CF CLI installation guide](https://docs.cloudfoundry.org/cf-cli/install-go-cli.html) and the [btp CLI installation guide](https://help.sap.com/docs/btp/sap-business-technology-platform/cli-tool).

### Activate and log in to BTP-Administration MCP Server

The automated steps in this playbook require the BTP-Administration MCP Server to be registered and authenticated in Claude Code. Run once per session before starting:

```
/mcp
```

Select `BTP-Administration` → complete browser login. Access tokens expire after 30 minutes and are renewed automatically. If you see an authorization error mid-playbook, run `/mcp` again.

If BTP-Administration is not yet registered, follow `01-playbook-btp-adm-mcp-playbook.md` first.

---

## Step 1: Assign AI Core Entitlement

**Prerequisite:** `aicore / extended` must be available at Global Account level.

**Automated (via BTP-Administration MCP):**
```
SubaccountEntitlement-assign: aicore / extended → <SUBACCOUNT>
```

**Manual (BTP Cockpit):**
1. BTP Cockpit → `<GLOBAL_ACCOUNT>` → Entitlements → Entity Assignments
   _(Global Account subdomain: visible in the top-left of the BTP Cockpit)_
2. Select subaccount → **Configure Entitlements** → **Add Service Plans**
3. Service `AI Core` → Plan `extended` → **Add** → **Save**

---

## Step 2: Create AI Core Service Instance

The instance can be created at BTP level or CF level — CF is required for app binding (`cf bind-service`); BTP level is sufficient for subaccount-only access.

**BTP level (automated via BTP-Administration MCP):**
```
ServiceInstance-create: aicore / extended → "aicore"
```

**Manual (BTP Cockpit):**
1. `<SUBACCOUNT>` → Instances and Subscriptions → **Create**
2. Service: `AI Core`, Plan: `extended`, Instance Name: `aicore` → **Create**

**CF level** (only needed if you want a CF-managed instance for app binding):

First, enable the CF environment and log in if not already done:

1. BTP Cockpit → `<SUBACCOUNT>` → Cloud Foundry Environment → **Enable Cloud Foundry**
2. Set an org name (e.g. `org-<SUBDOMAIN>`) → wait until status is `OK`
3. Create a space: CF Environment → Spaces → **Create Space** → Name: `dev`
4. Find the CF API endpoint under: BTP Cockpit → `<SUBACCOUNT>` → Cloud Foundry Environment → API Endpoint

```bash
cf login -a <CF_API> --sso
# Browser opens → get passcode from <CF_LOGIN_URL> and enter it
# Select org and space

cf create-service aicore extended aicore
```

**Note:** Creating an AI Core instance automatically provisions an Orchestration Deployment — Step 7 is therefore usually not needed.

---

## Step 3: Assign AI Launchpad Entitlement and Subscribe

**Automated (via BTP-Administration MCP):**
```
SubaccountEntitlement-assign: ai-launchpad / standard → <SUBACCOUNT>
Subaccount-subscribe: ai-launchpad / standard
```

**Manual (BTP Cockpit):**
1. BTP Cockpit → `<GLOBAL_ACCOUNT>` → Entitlements → Entity Assignments → select subaccount → **Configure Entitlements** → **Add Service Plans** → `ai-launchpad / standard` → **Add** → **Save**
2. Service Marketplace → **SAP AI Launchpad** → **Subscribe** → Plan `standard`
3. Wait until status is `Subscribed`

---

## Step 4: Assign AI Launchpad Roles

**Assign all 5 at once** — do not forget `ailaunchpad_genai_administrator`:

| Role Collection | Purpose |
|---|---|
| `ailaunchpad_aicore_admin_editor` | AI Core admin access (connections, resource groups) |
| `ailaunchpad_genai_manager` | GenAI Hub — deploy and manage models |
| `ailaunchpad_connections_editor` | Create and edit AI Core connections |
| `ailaunchpad_allow_all_resourcegroups` | Access to all resource groups |
| `ailaunchpad_genai_administrator` | Generative AI Hub administrator — full access |

**Automated (via BTP-Administration MCP):**
```
RoleCollection-assign: all 5 above → <USER>
```

**Manual (BTP Cockpit):**
1. `<SUBACCOUNT>` → Security → Users → `<USER>`
2. Assign Role Collections → select all 5

### Troubleshooting: "Access Denied" in AI Launchpad

**Cause:** Newly assigned roles only take effect after a fresh browser session.

**Fix:**
1. Close the browser completely (all windows)
2. Restart the browser
3. Reopen AI Launchpad

If "Access Denied" persists:
- Check that `ailaunchpad_genai_administrator` is assigned (frequently overlooked)
- Check that `ailaunchpad_allow_all_resourcegroups` is assigned

### Note: `_v2` Role Collections

Some roles exist as a `_v2` variant (e.g. `ailaunchpad_functions_explorer_editor_v2`). Variants without `_v2` are deprecated. For pure GenAI Hub usage these roles are not relevant — they concern the ML training area.

---

## Step 5: Create AI Core Service Key

The service key is needed to connect AI Launchpad to AI Core (Step 6) and to authenticate direct API calls. The CF service key is recommended — it reliably provides all required fields (`clientid`, `clientsecret`, `AI_API_URL`, `url`).

> **Recommendation: Use the CF instance and CF service key.** It covers both use cases — AI Launchpad and direct API calls. A separate BTP-level binding is redundant.

> **Finding: Two AI Core instances — CF vs. sapcp**
> If AI Core is created once via MCP/BTP Cockpit (`sapcp` context) and once via `cf create-service` (`cloudfoundry` context), two instances are created. According to SAP documentation, **all instances within a subaccount reference the same AI Core tenant** — you are not billed twice. The only difference is the credential format:
> - **CF instance** (`origin: cloudfoundry`) → CF service key via `cf create-service-key`
> - **BTP instance** (`origin: sapcp`) → btp service binding
>
> AI Launchpad expects a service key with the fields `clientid`, `clientsecret`, `AI_API_URL`, and `url` — the CF service key reliably provides this format.
> **Recommendation:** Keep only the CF instance — it covers both use cases.

**Via CF CLI (recommended):**
```bash
cf create-service-key aicore aicore-key
cf service-key aicore aicore-key
```

**Via btp CLI (alternative, for BTP-level instance):**
```bash
btp create services/binding \
  --subaccount <SUBACCOUNT_ID> \
  --binding <BINDING_NAME> \
  --service-instance <INSTANCE_ID>
```

Display credentials:
```bash
btp --format json list services/binding --subaccount <SUBACCOUNT_ID>
```

> **Note:** `btp get services/binding <NAME>` does not work via the btp CLI when the instance was created in the `sapcp` context — use `list` instead and filter by name.

Relevant fields from the credentials:
```json
{
  "clientid": "sb-...|aicore!b540",
  "clientsecret": "...",
  "url": "https://<SUBDOMAIN>.authentication.<REGION>.hana.ondemand.com",
  "serviceurls": {
    "AI_API_URL": "https://api.ai.prod.<REGION>.aws.ml.hana.ondemand.com"
  }
}
```

---

## Step 6: Create AI Core Connection in AI Launchpad

**Manual (required — no API path for this step):**
1. BTP Cockpit → `<SUBACCOUNT>` → Instances and Subscriptions → SAP AI Launchpad → **Go to Application**
2. **AI API Connections** → **Add**
3. Connection Name: any name (e.g. `<SUBACCOUNT>-aicore`)
4. Paste the **CF Service Key JSON** (from `cf service-key aicore aicore-key`)
5. **Save**

**Important:** Use the CF service key (`cf service-key`), not the BTP service key from the cockpit — the BTP key does not work here and the page will remain empty.

---

## Step 7: Check / Deploy Orchestration Service

**Check if already present** (created automatically when the instance is provisioned):

```bash
# Get token
TOKEN=$(curl -s -X POST "<TOKEN_URL>" \
  --user "<clientid>:<clientsecret>" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

# Check deployments
curl -s "<AI_API_URL>/v2/lm/deployments" \
  -H "Authorization: Bearer $TOKEN" \
  -H "AI-Resource-Group: default" | python3 -c "
import sys,json
d=json.load(sys.stdin)
for r in d.get('resources',[]):
    print(r['id'], r['status'], r.get('scenarioId','?'))"
```

If no deployment is present, create one manually:

**Via AI Launchpad:**
1. AI Launchpad → select connection → **GenAI Hub** → **Model Deployments** → **Create**
2. Scenario: `Orchestration`, Executable: `orchestration`
3. **Next** → **Create** → wait until status is **Running**

**Via API:**
```bash
# Create configuration
curl -s -X POST "<AI_API_URL>/v2/lm/configurations" \
  -H "Authorization: Bearer $TOKEN" \
  -H "AI-Resource-Group: default" \
  -H "Content-Type: application/json" \
  -d '{"name":"orchestration-config","scenarioId":"orchestration","executableId":"orchestration","parameterBindings":[],"inputArtifactBindings":[]}'
# → note the config ID

# Start deployment
curl -s -X POST "<AI_API_URL>/v2/lm/deployments" \
  -H "Authorization: Bearer $TOKEN" \
  -H "AI-Resource-Group: default" \
  -H "Content-Type: application/json" \
  -d '{"configurationId":"<CONFIG_ID>"}'
```

**Delete duplicate deployments:**
```bash
# Stop
curl -s -X PATCH "<AI_API_URL>/v2/lm/deployments/<ID>" \
  -H "Authorization: Bearer $TOKEN" \
  -H "AI-Resource-Group: default" \
  -H "Content-Type: application/json" \
  -d '{"targetStatus":"STOPPED"}'
# Wait until status is STOPPED (~60s), then:

# Delete
curl -s -X DELETE "<AI_API_URL>/v2/lm/deployments/<ID>" \
  -H "Authorization: Bearer $TOKEN" \
  -H "AI-Resource-Group: default"
```

**Available models (selection):**

| Provider | Model | Status |
|---|---|---|
| Anthropic | `anthropic--claude-4.5-sonnet` | ✅ |
| Anthropic | `anthropic--claude-4.5-haiku` | ✅ |
| Anthropic | `anthropic--claude-4.6-sonnet` | ✅ |
| Anthropic | `anthropic--claude-4.6-opus` | ✅ |
| Anthropic | `anthropic--claude-3-haiku` | DEPRECATED |
| OpenAI | `gpt-4.1`, `o3`, `o4-mini` | ✅ |
| Google | `gemini-2.5-pro`, `gemini-2.5-flash` | ✅ |
| SAP | `sap-abap-1`, `sap-rpt-1.5` | ✅ |
| Perplexity | `sonar-pro`, `sonar` | ✅ |
| Mistral | `mistralai--mistral-medium`, `mistralai--mistral-small` | ✅ |

> **Note:** List models via `GET /v2/lm/scenarios/foundation-models/models` (not `/v2/lm/models` — that endpoint returns `RBAC: access denied`).

> **SAP AI Core model names:** SAP uses its own naming scheme with a provider prefix (e.g. `anthropic--claude-4.6-sonnet`). The native Anthropic API name is `claude-sonnet-4-6`. Both refer to the same model.

---

## Step 7b: Create Foundation Model Deployment (optional)

Foundation Model Deployments provide direct access to a specific model via a dedicated deployment URL — for example, to use AI Core as the LLM backend for Claude Code.

### Check available models and executableIds

```python3
# Determine executableId per model
import urllib.request, json, base64

client_id = "<AICORE_CLIENT_ID>"
client_secret = "<AICORE_CLIENT_SECRET>"
token_url = "<AICORE_TOKEN_URL>"
api_url = "<AICORE_API_URL>"

auth = base64.b64encode(f'{client_id}:{client_secret}'.encode()).decode()
req = urllib.request.Request(token_url, data=b'grant_type=client_credentials',
    headers={'Authorization': f'Basic {auth}', 'Content-Type': 'application/x-www-form-urlencoded'})
token = json.loads(urllib.request.urlopen(req).read())['access_token']

req2 = urllib.request.Request(f'{api_url}/v2/lm/scenarios/foundation-models/models',
    headers={'Authorization': f'Bearer {token}', 'AI-Resource-Group': 'default'})
d = json.loads(urllib.request.urlopen(req2).read())
for m in d['resources']:
    if 'claude' in m['model'].lower():
        print(m['model'], '->', m['executableId'])
```

> **Note:** Anthropic Claude models run under `executableId: aws-bedrock` — not `anthropic-ai`. `/v2/lm/models` returns `RBAC: access denied` — use `/v2/lm/scenarios/foundation-models/models` instead.

> **Note:** Credentials with `|` in `clientid` can break when running `source .env` in bash. Use Python for API calls to avoid shell escaping issues.

### Create configuration

```bash
curl -s -X POST "<AI_API_URL>/v2/lm/configurations" \
  -H "Authorization: Bearer $TOKEN" \
  -H "AI-Resource-Group: default" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "<CONFIG_NAME>",
    "scenarioId": "foundation-models",
    "executableId": "aws-bedrock",
    "parameterBindings": [
      {"key": "modelName", "value": "<MODEL_NAME>"},
      {"key": "modelVersion", "value": "latest"}
    ],
    "inputArtifactBindings": []
  }'
# → note the id from the response
```

### Start deployment

```bash
curl -s -X POST "<AI_API_URL>/v2/lm/deployments" \
  -H "Authorization: Bearer $TOKEN" \
  -H "AI-Resource-Group: default" \
  -H "Content-Type: application/json" \
  -d '{"configurationId": "<CONFIG_ID>"}'
# → note id and deploymentUrl
```

The deployment starts immediately — check status:

```bash
curl -s "<AI_API_URL>/v2/lm/deployments/<DEPLOYMENT_ID>" \
  -H "Authorization: Bearer $TOKEN" \
  -H "AI-Resource-Group: default" | python3 -c "
import sys,json; d=json.load(sys.stdin)
print('Status:', d.get('status'))
print('URL:', d.get('deploymentUrl',''))"
```

Once `RUNNING`, the `deploymentUrl` is available — use it as `ANTHROPIC_BASE_URL` for Claude Code (append `/anthropic/` suffix).

---

## Step 8: Add AI Core Credentials to .env

For local development — `.env` file in the project directory:

```bash
# AI Core Credentials
AICORE_CLIENT_ID=<clientid>
AICORE_CLIENT_SECRET=<clientsecret>
AICORE_TOKEN_URL=https://<SUBDOMAIN>.authentication.<REGION>.hana.ondemand.com/oauth/token
AICORE_API_URL=https://api.ai.prod.<REGION>.aws.ml.hana.ondemand.com
AICORE_IDENTITY_ZONE=<SUBDOMAIN>
```

> **Note:** Never commit the `.env` file — add it to `.gitignore`. The `|` in `clientid` can cause errors when running `source .env` (the shell interprets it as a pipe). Set credentials directly as shell variables or wrap the value in quotes in the `.env` file.

---

## Step 9: Verify Installation

**CF services:**
```bash
cf services
# Expected: aicore / extended / create succeeded
```

**Retrieve AI Core credentials:**
```bash
cf service-key aicore aicore-key
# Use clientid + clientsecret from the output in the verification script below
```

**Check deployments and models:**
```bash
CLIENT_ID="<clientid>"
CLIENT_SECRET='<clientsecret>'
TOKEN_URL="<TOKEN_URL>"
AI_API_URL="<AI_API_URL>"

TOKEN=$(curl -s -X POST "$TOKEN_URL" \
  --user "$CLIENT_ID:$CLIENT_SECRET" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" | python3 -c "import sys,json; print(json.load(sys.stdin)['access_token'])")

# Deployments
curl -s "$AI_API_URL/v2/lm/deployments" \
  -H "Authorization: Bearer $TOKEN" \
  -H "AI-Resource-Group: default" | python3 -c "
import sys,json
d=json.load(sys.stdin)
for r in d.get('resources',[]):
    print(r['id'], r['status'], r.get('scenarioId','?'))"

# Models
curl -s "$AI_API_URL/v2/lm/scenarios/foundation-models/models" \
  -H "Authorization: Bearer $TOKEN" \
  -H "AI-Resource-Group: default" | python3 -c "
import sys,json
d=json.load(sys.stdin)
print('Total:', d['count'], 'models')"
```

**Expected result:**
- At least 1 deployment `RUNNING` with scenario `orchestration`
- Models endpoint responds successfully

**AI Launchpad:**
1. BTP Cockpit → Subaccount → Instances and Subscriptions → AI Launchpad → **Go to Application**
2. AI API Connections → connection must appear
3. GenAI Hub → Model Deployments → status `Running`

---

## Troubleshooting: MCP and authorization errors

| Error | Cause | Fix |
|---|---|---|
| Auth error / token expired | Access token expired (30 min) | `/mcp` → select server → browser login again |
| 403 on SubaccountEntitlement-assign | Missing `Global Account Administrator` role | Assign role collection in BTP Cockpit, then re-run |
| 403 on ServiceInstance-create or RoleCollection-assign | Missing `Subaccount Administrator` role | Assign role collection in BTP Cockpit, then re-run |
| ServiceInstance-create fails | Entitlement not yet assigned | Complete Step 1 first, then retry |
| BTP-Administration server not found | Server not registered or wrong URL | Run `claude mcp list`; follow `01-playbook-btp-adm-mcp-playbook.md` to register |

---

## Expected State After Completion (<SUBACCOUNT>)

| Component | Status |
|---|---|
| AI Core Entitlement | ✅ assigned |
| AI Core Service Instance (BTP) | ✅ created (`<AICORE_INSTANCE_ID_SHORT>`) |
| AI Core Service Instance (CF) | ✅ created |
| AI Launchpad Subscription | ✅ subscribed |
| AI Launchpad Roles (5) | ✅ assigned |
| AI Core Connection in AI Launchpad | ✅ created (CF service key) |
| Resource Group | `default` |
| Document Grounding | ❌ not enabled |
| Orchestration Service Deployment | ✅ RUNNING (`<ORCHESTRATION_DEPLOYMENT_ID>`) |
| AI Core Credentials in .env | ✅ added |
| Model | ✅ `<MODEL_NAME>` |

---

## Document Grounding (Vector API)

Document Grounding enables RAG (Retrieval-Augmented Generation) — your own documents are vectorized and automatically included as context in LLM requests.

### Supported document sources

| Source | Description |
|---|---|
| **Vector API** | Direct upload via API (no external storage required) |
| **Amazon S3** | S3 bucket as document source |
| **Microsoft SharePoint** | SharePoint Online site/folder |
| **ServiceNow** | Available since March 2026 |
| **SAP Help Portal** | Only help.sap.com content (no custom content) |

### Prerequisites

- AI Core `extended` service plan (already in place)
- **New resource group with grounding label** — the `default` resource group cannot be used for grounding (update not allowed, 409)
- The label `ext.ai.sap.com/document-grounding: true` must be set when creating the resource group

### Create resource group for grounding

```bash
POST /v2/admin/resourceGroups
{
  "resourceGroupId": "<GROUNDING_RESOURCE_GROUP>",
  "labels": [{"key": "ext.ai.sap.com/document-grounding", "value": "true"}]
}
```

→ Responds with **202 Accepted** (asynchronous). Check status until `PROVISIONED`:

```bash
GET /v2/admin/resourceGroups/<GROUNDING_RESOURCE_GROUP>
```

### Create collection

```bash
POST /v2/lm/document-grounding/vector/collections
AI-Resource-Group: <GROUNDING_RESOURCE_GROUP>
{
  "title": "<COLLECTION_NAME>",
  "embeddingConfig": {"modelName": "text-embedding-3-large"}
}
```

> **Note:** `embeddingConfig` is camelCase — that is what the API expects.

### Upload document (Vector API)

The API expects **pre-split text chunks** (not raw PDF). The PDF must first be extracted with `pdftotext` and split into ~500-character chunks:

```json
POST /v2/lm/document-grounding/vector/collections/{collectionId}/documents
{
  "documents": [{
    "metadata": [{"key": "fileName", "value": ["document.pdf"]}],
    "chunks": [
      {"content": "Text section 1...", "metadata": []},
      {"content": "Text section 2...", "metadata": []}
    ]
  }]
}
```

Script: `<VECTOR_UPLOAD_SCRIPT>` (extracts PDF with `pdftotext`, splits into chunks, uploads)

### Search / Retrieval

```bash
POST /v2/lm/document-grounding/vector/search
AI-Resource-Group: {resourceGroup}
{
  "query": "search query",
  "filters": [{
    "id": "{collectionId}",
    "data_repository_type": "vector",
    "data_repositories": ["{collectionId}"],
    "search_config": {"max_chunk_count": 3}
  }]
}
```

### Expected State After Completion (<SUBACCOUNT>)

| Component | Value | Status |
|---|---|---|
| Resource Group | `<GROUNDING_RESOURCE_GROUP>` | ✅ PROVISIONED |
| Collection ID | `<COLLECTION_ID>` | ✅ created |
| Test document | `<TEST_DOCUMENT>` | ✅ uploaded |
| Document ID | `<DOCUMENT_ID>` | ✅ vectorized |
| Embedding model | `text-embedding-3-large` | ✅ active |
| Vector search | Test query | ✅ 3 relevant chunks returned |

### Key findings

- `default` resource group **cannot** be used for grounding → create a new resource group
- Vector API expects text chunks, **not** raw PDF — extract first with `pdftotext`
- `filters` requires `id`, `data_repository_type`, `data_repositories`, and `search_config` — all mandatory
- Resource group provisioning is asynchronous (202) — wait for status `PROVISIONED`
- Scripts: `<VECTOR_UPLOAD_SCRIPT>` (extract PDF + upload), `<VECTOR_SEARCH_SCRIPT>` (search)

### Next step: Integrate grounding into orchestration

This combines vector search and LLM in a single API call, via the `/completion` endpoint:

```bash
POST /v2/inference/deployments/<GROUNDING_DEPLOYMENT_ID>/completion
AI-Resource-Group: <GROUNDING_RESOURCE_GROUP>
{
  "orchestration_config": {
    "module_configurations": {
      "grounding_module_config": {
        "type": "document_grounding_service",
        "config": {
          "filters": [{
            "id": "<COLLECTION_ID>",
            "data_repository_type": "vector",
            "data_repositories": ["<COLLECTION_ID>"],
            "search_config": {"max_chunk_count": 3}
          }],
          "input_params": ["groundingRequest"],
          "output_param": "groundingOutput"
        }
      },
      "llm_module_config": {
        "model_name": "<MODEL_NAME>",
        "model_params": {"max_tokens": 500}
      },
      "templating_module_config": {
        "template": [
          {"role": "user", "content": "Context: {{ ?groundingOutput }}\n\nQuestion: {{ ?groundingRequest }}"}
        ]
      }
    }
  },
  "input_params": {"groundingRequest": "<YOUR_QUESTION>"}
}
```

Script: `<GROUNDING_ORCHESTRATION_SCRIPT>` — returns a full LLM response based on the PDF content.

> **Note — `/completion` vs `/v2/completion`:** For ad-hoc calls with a custom template, `/completion` (v1) with `orchestration_config` + `input_params` is the right approach — it works reliably. The `/v2/completion` endpoint is for **pre-configured workflows** built in AI Launchpad: build the workflow there, save it, then call it with `placeholder_values` only. For dynamic queries, v2 is not suitable.

---

## Subaccount-specific values

| Placeholder | Value |
|---|---|
| `<GLOBAL_ACCOUNT>` | _(Global Account subdomain — visible top-left in BTP Cockpit)_ |
| `<SUBACCOUNT_ID>` | |
| `<SUBACCOUNT>` | _(display name)_ |
| `<SUBDOMAIN>` | |
| `<CF_API>` | `https://api.cf.<REGION>.hana.ondemand.com` |
| `<CF_LOGIN_URL>` | `https://login.cf.<REGION>.hana.ondemand.com/passcode` |
| `<CF_ORG>` | |
| `<CF_SPACE>` | `dev` |
| `<TOKEN_URL>` | `https://<SUBDOMAIN>.authentication.<REGION>.hana.ondemand.com/oauth/token` |
| `<AI_API_URL>` | `https://api.ai.prod.<REGION>.aws.ml.hana.ondemand.com` |
| `<USER>` | |
| `<AICORE_INSTANCE_ID>` | |
| `<AICORE_INSTANCE_ID_SHORT>` | _(first 8 chars)_ |
| `<ORCHESTRATION_DEPLOYMENT_ID>` | _(auto-created)_ |
| `<MODEL_NAME>` | e.g. `anthropic--claude-4.6-sonnet` |
| `<GROUNDING_RESOURCE_GROUP>` | |
| `<COLLECTION_ID>` | |
| `<TEST_DOCUMENT>` | |
| `<DOCUMENT_ID>` | |
| `<GROUNDING_DEPLOYMENT_ID>` | _(Orchestration Deployment in grounding resource group)_ |
| `<VECTOR_UPLOAD_SCRIPT>` | e.g. `vector_upload.py` |
| `<VECTOR_SEARCH_SCRIPT>` | e.g. `vector_search.py` |
| `<GROUNDING_ORCHESTRATION_SCRIPT>` | e.g. `grounding_orchestration.py` |

> **Note on SAP AI Core model names:** SAP uses its own naming scheme with a provider prefix (e.g. `anthropic--claude-4.6-sonnet`). The native Anthropic API name is `claude-sonnet-4-6`. Both refer to the same model — the SAP name is used in the GenAI Hub and AI Core API calls; the Anthropic name is used when accessing the Anthropic API directly.
