# BTP Kyma Landscape Setup — Playbook

Setup of a Kyma environment in a BTP subaccount from scratch via BTP MCP Server and CLIs.

Typically you enable the Kyma runtime through the BTP Cockpit UI after entitling your subaccount. This playbook shows how to do the same step-by-step with your AI coding assistant.

At the end is a small hello-world example to test your setup.

---

## Prerequisites

### Tool Installation

Example is for macOS only, via Homebrew.

```bash
brew install btp
brew install kubernetes-cli  # kubectl
brew install kyma-cli
brew install int128/kubelogin/kubelogin  # for OIDC-based kubeconfig; int128/kubelogin is a tap (external Homebrew repository from github.com/int128/kubelogin)
brew install cloudfoundry/tap/cf-cli@8   # cloudfoundry/tap is also a tap
```

> **What is a tap?** A tap is an additional package source for Homebrew — like an extra repository. By default, Homebrew only includes `homebrew/core`. Packages not available there are provided by the vendor via their own tap. Homebrew adds the tap automatically when you run `brew install`.

### kubectl vs. kyma CLI

| | `kubectl` | `kyma` CLI |
|---|---|---|
| **What** | Universal Kubernetes CLI | SAP-specific wrapper tool |
| **Talks to** | Kubernetes API directly | BTP + Kubernetes |
| **Used for** | Managing pods, deployments, services, secrets, namespaces | Kyma setup, module management, fetching kubeconfig |
| **Typical commands** | `kubectl get pods`, `kubectl apply`, `kubectl logs` | `kyma deploy`, `kyma alpha kubeconfig get` |

**Rule of thumb:** Kyma setup and module management → `kyma` CLI. Everything after that (deploying apps, debugging, managing resources) → `kubectl`.

---

## Setup Steps

### 1. Check Versions

```bash
btp --version
kyma version
kubectl version --client
```

### 2. BTP CLI Login

```bash
btp login --url https://cli.btp.cloud.sap --subdomain <GA_SUBDOMAIN> --sso
```

> **Note:** `--subdomain` expects the **subdomain**, not the GUID of the global account. Find the subdomain via MCP `GlobalAccount-list` → `subdomain` column, or in the BTP Cockpit in the top left under the display name.

### 3. Assign Kyma Entitlement

Via btp CLI:
```bash
btp assign accounts/entitlement \
  --to-subaccount <SUBACCOUNT_ID> \
  --for-service kymaruntime --plan aws --amount 1
```

Or via MCP: `SubaccountEntitlement-assign` with `service=kymaruntime`, `plan=aws`, `amount=1`.

### 4. Enable Kyma Environment

> **Important:** Kyma **cannot** be enabled via the MCP Server — `CloudFoundryEnvironment-enable` only works for CF. Use the btp CLI instead.

```bash
btp create accounts/environment-instance \
  --subaccount <SUBACCOUNT_ID> \
  --display-name <DISPLAY_NAME> \
  --environment kyma \
  --service kymaruntime \
  --plan aws \
  --parameters '{"name":"<CLUSTER_NAME>","region":"<AWS_REGION>"}'
```

> **Important:** The `region` parameter expects the **AWS region name**, not the BTP region name (see Hint 3).

Runs in the background (~20 min). Note the instance ID from the output, then check status:
```bash
btp get accounts/environment-instance <INSTANCE_ID> --subaccount <SUBACCOUNT_ID>
```

### 5. Retrieve Kubeconfig URL and Cluster ID

Once state is `OK`:
```bash
btp --format json get accounts/environment-instance <INSTANCE_ID> \
  --subaccount <SUBACCOUNT_ID> | python3 -c \
  "import sys,json; l=json.load(sys.stdin)['labels']; print('KubeconfigURL:', l['KubeconfigURL']); print('APIServerURL:', l['APIServerURL'])"
```

Read the `<CLUSTER_ID>` from the `APIServerURL` (`https://api.<CLUSTER_ID>.kyma.ondemand.com`) — needed for the Hello World URL.

### 6. Fetch kubeconfig

Open the kubeconfig URL in your browser (SSO login, file is downloaded):
```
https://kyma-env-broker.cp.kyma.cloud.sap/kubeconfig/<INSTANCE_ID>
```

Set it as the default kubeconfig:
```bash
cp ~/Downloads/kubeconfig.yaml ~/.kube/config
```

> **Note:** `kyma alpha kubeconfig get --subaccount ... --shoot ...` does not work — unknown flags. The kubeconfig must be downloaded via the URL in the browser.

### 7. Test Cluster Access

```bash
kubectl get nodes
kubectl get pods -A
```

---

## Hello World — nginx Deployment

Complete example: nginx deployment + service + APIRule for external access.

### Manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
  namespace: hello-world
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html
        configMap:
          name: hello-world-html
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: hello-world-html
  namespace: hello-world
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head><title>Hello World</title></head>
    <body>
      <h1>Hello World! You set up Kyma successfully!</h1>
    </body>
    </html>
---
apiVersion: v1
kind: Service
metadata:
  name: hello-world
  namespace: hello-world
spec:
  selector:
    app: hello-world
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: gateway.kyma-project.io/v2
kind: APIRule
metadata:
  name: hello-world
  namespace: hello-world
spec:
  hosts:
  - hello-world
  service:
    name: hello-world
    port: 80
  gateway: kyma-system/kyma-gateway
  rules:
  - path: /*
    methods: ["GET"]
    noAuth: true
```

> **Note:** Custom HTML content is deployed as a ConfigMap and mounted into the nginx container at `/usr/share/nginx/html`.

### Deployment

```bash
# Create namespace with Istio sidecar injection
kubectl create namespace hello-world
kubectl label namespace hello-world istio-injection=enabled

# Deploy
kubectl apply -f hello-world.yaml

# Check status (pod must show 2/2 Ready — nginx + Istio sidecar)
kubectl get pods,apirule -n hello-world
```

### Result

```
https://hello-world.<CLUSTER_ID>.kyma.ondemand.com
```

Get the cluster ID from step 5 (`APIServerURL`).

---

## Hints

---

### Hint 5: Istio Sidecar Injection Must Be Explicitly Enabled

**Problem:** APIRule remains in `Error` state: *"Pod does not have an injected istio sidecar"*.

**Cause:** New namespaces do not have Istio sidecar injection enabled by default. Without the sidecar, Istio cannot route traffic.

**Solution:** Label the namespace and restart the deployment:
```bash
kubectl label namespace <namespace> istio-injection=enabled --overwrite
kubectl rollout restart deployment/<name> -n <namespace>
```

---

### Hint 4: APIRule v1beta1 Is No Longer Supported

**Problem:** `kubectl apply` fails: *"v1beta1 APIRule version is no longer supported, please use v2 instead"*.

**Solution:** Use `apiVersion: gateway.kyma-project.io/v2`. Key changes in v2:
- `host` → `hosts` (list)
- `path: /.*` → `path: /*` (no more regex, wildcard is `/*`)
- `accessStrategies` → `noAuth: true` for public access

---

### Hint 3: Kyma Region Parameter ≠ BTP Region

**Problem:** `btp create accounts/environment-instance` fails with "not a valid enum value" for `eu10` or `eu-west-1`.

**Cause:** The `region` parameter in the Kyma provisioning JSON expects the **AWS region name**, not the BTP region name.

**Mapping for AWS plan:**

| BTP Region | Kyma `region` Parameter |
|---|---|
| `eu10` — Europe (Frankfurt) | `eu-central-1` |
| `us10` — US East (VA) | `us-east-1` |
| `ap10` — Australia (Sydney) | `ap-southeast-2` |
| `jp10` — Japan (Tokyo) | `ap-northeast-1` |
| `ca10` — Canada (Montreal) | `ca-central-1` |
| `br10` — Brazil (São Paulo) | `sa-east-1` |

Source: [SAP Help — Provisioning Parameters Kyma](https://help.sap.com/docs/BTP/65de2977205c403bbc107264b8eccf4b/e2e13bfaa2f54a4fb179f0f1f840353a.html)

---

### Hint 2: Kyma Environment Cannot Be Enabled via MCP

**Problem:** The MCP Server has no tool for enabling Kyma environments. `CloudFoundryEnvironment-enable` only works for CF — calling it for Kyma fails with "upstream rejected".

**What is possible via MCP:**
- `SubaccountEntitlement-assign` — assign `kymaruntime / aws` to the subaccount ✅
- `Environment-list` — check whether Kyma is available in the subaccount ✅
- `EnvironmentInstance-list` — check the status of the Kyma cluster (after activation) ✅

**Solution:** Use the btp CLI (see setup step 4).

---

### Hint 1: Global Account Subdomain ≠ GUID

**Problem:** MCP tool calls fail with "resource not found" even though the global account GUID is correct.

**Cause:** The MCP Server expects the **subdomain** of the global account, not the GUID. For some global accounts the subdomain differs from the GUID.

**Solution:** Always call `GlobalAccount-list` first — the list contains both `guid` and `subdomain`. Use the `subdomain` value as the `global_account` parameter.

**Steps (if the global account subdomain is unknown):**
1. Call `GlobalAccount-list` → list of all accessible global accounts with subdomains
2. Identify the correct global account by `display_name`
3. Use the `subdomain` value as the `global_account` parameter for all subsequent calls

---

## Reference Setup

Replace the placeholders below with your actual values before running the setup steps.

| Variable | Value |
|---|---|
| `<GA_SUBDOMAIN>` | Your global account subdomain — BTP Cockpit top left, or via `GlobalAccount-list` |
| `<SUBACCOUNT_ID>` | Your subaccount ID — BTP Cockpit → Subaccount → Overview |
| `<DISPLAY_NAME>` | Display name for the Kyma environment instance (e.g. `my-kyma`) |
| `<CLUSTER_NAME>` | Name of the Kyma cluster (e.g. `my-kyma-cluster`) |
| `<AWS_REGION>` | AWS region name for your BTP region (see Hint 3, e.g. `eu-central-1` for `eu10`) |
| `<INSTANCE_ID>` | Kyma environment instance ID — returned by `btp create accounts/environment-instance` |
| `<CLUSTER_ID>` | Cluster ID — read from `APIServerURL` once the instance is `OK` |
