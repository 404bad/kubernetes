# Helm — Kubernetes Package Manager

## What is Helm?

Helm is the package manager for Kubernetes — think of it like `apt` for Ubuntu or `dnf` for Fedora, but for Kubernetes applications. Instead of manually writing and applying dozens of YAML files, Helm bundles everything into a **Chart** and lets you deploy, upgrade, and roll back applications with a single command.

---

## Why Helm Over `kubectl apply -f`?

|             | Manual (`kubectl apply -f`)       | Helm                                 |
| ----------- | --------------------------------- | ------------------------------------ |
| Deployment  | Apply each YAML file one by one   | Single command deploys everything    |
| Upgrades    | Manually track and update files   | `helm upgrade` handles it            |
| Rollbacks   | Manually revert YAML and reapply  | `helm rollback` to any revision      |
| Reusability | Copy-paste and edit files per app | Parameterize once, reuse everywhere  |
| History     | No built-in tracking              | Full revision history out of the box |

**Bottom line:** Helm wraps all your Kubernetes manifests (Deployment, Service, ConfigMap, Ingress, etc.) into one reusable, versioned package called a _chart_.

---

## Installing Helm

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
chmod 700 get_helm.sh
./get_helm.sh
```

Verify installation:

```bash
helm version
```

---

## Creating Your First Chart

```bash
helm create bankapp
cd bankapp
```

### Chart Directory Structure

```
bankapp/
├── Chart.yaml          # Metadata about the chart
├── values.yaml         # Default configuration values
├── charts/             # Dependency charts (sub-charts)
└── templates/          # Kubernetes manifest templates
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── configmap.yaml
    ├── hpa.yaml
    ├── serviceaccount.yaml
    ├── NOTES.txt
    └── _helpers.tpl
```

### File & Directory Descriptions

#### `Chart.yaml`

The identity card of your chart. Contains metadata such as:

- `apiVersion` — Helm API version (`v2` for Helm 3)
- `name` — Name of the chart (e.g., `bankapp`)
- `description` — Short description of what the chart does
- `type` — `application` or `library`
- `version` — Chart version (increment on each chart change)
- `appVersion` — Version of the application being packaged

```yaml
apiVersion: v2
name: bankapp
description: A Helm chart for the Bank Application
type: application
version: 0.1.0
appVersion: "1.0.0"
```

#### `values.yaml`

The single source of truth for all configurable values. Templates reference these values using `{{ .Values.<key> }}`. You override these at deploy time without touching the templates.

```yaml
replicaCount: 1

image:
  repository: myrepo/bankapp
  tag: "latest"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false # <-- disable ingress if not needed per app
```

#### `charts/` Directory

Stores **dependency charts** (sub-charts). If your app depends on Redis or PostgreSQL, their charts live here. Managed via `helm dependency update`.

#### `templates/` Directory

Contains all Kubernetes manifest templates. Helm processes these with Go templating and substitutes values from `values.yaml`.

```bash
ls templates/
# deployment.yaml  service.yaml  ingress.yaml
# configmap.yaml   hpa.yaml      serviceaccount.yaml
# NOTES.txt        _helpers.tpl
```

| File                  | Purpose                                                |
| --------------------- | ------------------------------------------------------ |
| `deployment.yaml`     | Defines Pods, containers, resources, env vars          |
| `service.yaml`        | Exposes the application within the cluster             |
| `ingress.yaml`        | Routes external traffic (HTTP/HTTPS) to the service    |
| `configmap.yaml`      | Stores non-sensitive configuration as key-value pairs  |
| `hpa.yaml`            | Horizontal Pod Autoscaler — auto-scales pods           |
| `serviceaccount.yaml` | Kubernetes ServiceAccount for the app                  |
| `NOTES.txt`           | Post-install instructions printed to the terminal      |
| `_helpers.tpl`        | Reusable template helpers (naming conventions, labels) |

---

## Configuring Your Chart

### Disabling Ingress (when not needed per app)

Since you won't always create a separate `ingress.yaml` for every application (you may use a **shared/global ingress**), disable it in `values.yaml`:

```yaml
# values.yaml
ingress:
  enabled: false
```

Helm will skip rendering `ingress.yaml` because of the conditional block inside the template:

```yaml
# templates/ingress.yaml
{ { - if .Values.ingress.enabled - } }
...
{ { - end } }
```

### Adding a ConfigMap for Backend Application

Create `templates/configmap.yaml` for your backend-specific configuration:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "bankapp.fullname" . }}-config
data:
  DB_HOST: "postgres-service"
  DB_PORT: "5432"
  APP_ENV: "production"
```

### Customizing `deployment.yaml` and `service.yaml`

Edit these templates to match your application's actual container image, ports, environment variables, and resource limits:

```yaml
# templates/deployment.yaml (relevant section)
containers:
  - name: {{ .Chart.Name }}
    image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
    ports:
      - containerPort: {{ .Values.service.port }}
    envFrom:
      - configMapRef:
          name: {{ include "bankapp.fullname" . }}-config
```

---

## Deploying with Helm

Navigate to the **parent directory** of your chart folder, then run:

```bash
# Syntax: helm install <release-name> <chart-directory>
helm install bankapp bankapp
```

> **Note:** The first `bankapp` is the **release name** (what Helm tracks it as). The second `bankapp` is the **chart directory** name created by `helm create`.

---

## Day-to-Day Helm Commands

### List All Releases

```bash
helm list
```

### Upgrade a Release (used in CI/CD pipelines after first deploy)

```bash
helm upgrade bankapp bankapp
```

> In production pipelines: `helm install` on first deployment, `helm upgrade` on every subsequent deployment.

### Uninstall a Release

```bash
helm uninstall bankapp
```

### View Revision History

```bash
helm history bankapp
```

**Output columns explained:**

| Column        | Description                                                              |
| ------------- | ------------------------------------------------------------------------ |
| `REVISION`    | Incremental revision number (starts at 1, increases on each upgrade)     |
| `UPDATED`     | Timestamp of when this revision was deployed                             |
| `STATUS`      | Current state of this revision — see detailed breakdown below            |
| `CHART`       | Chart name and version used for this revision                            |
| `APP VERSION` | Application version from `Chart.yaml`                                    |
| `DESCRIPTION` | Human-readable description: `Install complete`, `Upgrade complete`, etc. |

#### STATUS Values — Detailed Breakdown

Each revision in `helm history` carries a STATUS that tells you exactly what happened to that revision and whether it is currently active.

---

**`deployed`**

This is the currently **active, live revision**. Only one revision can be in `deployed` state at any time. When you run `helm install` or a successful `helm upgrade`, the new revision gets this status. If you roll back, the revision you rolled back to becomes `deployed` again.

```
REVISION  STATUS      DESCRIPTION
1         superseded  Install complete
2         superseded  Upgrade complete
3         deployed    Upgrade complete   ← live right now
```

---

**`superseded`**

A revision that **was deployed in the past but has since been replaced** by a newer upgrade or rollback. Think of it as "retired but recorded." Helm keeps these in history so you can roll back to them. They are not running — the cluster is running whatever the `deployed` revision defines.

```
REVISION  STATUS      DESCRIPTION
1         superseded  Install complete   ← was live, now replaced
2         superseded  Upgrade complete   ← was live, now replaced
3         deployed    Upgrade complete   ← currently live
```

---

**`failed`**

The revision **did not deploy successfully**. This happens when `helm install` or `helm upgrade` encounters an error — for example, a pod fails its readiness probe, a resource is invalid, or Kubernetes rejects the manifest. Helm marks that revision as `failed` and (for upgrades) automatically rolls back to the last `deployed` revision.

```
REVISION  STATUS      DESCRIPTION
1         superseded  Install complete
2         superseded  Upgrade complete
3         failed      Upgrade "bankapp" failed: timed out waiting for pods
4         deployed    Rollback to 2
```

> **Real-world tip:** When a pipeline upgrade fails, always run `helm history <release>` first — the `failed` revision and its DESCRIPTION will tell you exactly what went wrong before you dig into `kubectl logs`.

---

**`pending-install`**

A revision is **in the middle of being installed right now**. You normally see this only for a brief moment during `helm install`. If a release is stuck in `pending-install` (e.g., the Helm process was killed mid-install), the release is left in a broken state and you'll need to run `helm uninstall <release>` to clean it up before reinstalling.

```
REVISION  STATUS           DESCRIPTION
1         pending-install  Initial install underway...
```

> **Real-world tip:** If `helm list` shows a release stuck in `pending-install` or `pending-upgrade` after a crash or timeout, it means Helm never finished writing the release state. Clean it up with `helm uninstall` and redeploy.

---

**Quick STATUS Reference**

| STATUS            | Meaning                           | Is it running? |
| ----------------- | --------------------------------- | -------------- |
| `deployed`        | Currently active revision         | ✅ Yes         |
| `superseded`      | Previously deployed, now replaced | ❌ No          |
| `failed`          | Deployment attempt errored out    | ❌ No          |
| `pending-install` | Install in progress (or stuck)    | ⚠️ Partial     |

### Roll Back to a Previous Revision

```bash
helm rollback bankapp <revision-number>

# Example: roll back to revision 2
helm rollback bankapp 2
```

---

## Shared Ingress for Multiple Apps

Instead of one ingress per app, maintain a **single global ingress** that routes traffic to all services by hostname or path. After deploying new apps with Helm, update and apply the shared ingress:

```yaml
# global-ingress.yaml
spec:
  rules:
    - host: bank.example.com
      http:
        paths:
          - path: /
            backend:
              service:
                name: bankapp # <-- Helm-deployed service name
                port:
                  number: 80
    - host: pokemon.example.com
      http:
        paths:
          - path: /
            backend:
              service:
                name: pokemonapp
                port:
                  number: 80
```

```bash
kubectl apply -f global-ingress.yaml
```

---

## Adding a Second Application (pokemonapp)

The power of Helm: copy an existing chart and rename it.

```bash
cp -rvf bankapp pokemonapp
cd pokemonapp
```

### Update `Chart.yaml`

```bash
vim Chart.yaml
# Change: name: bankapp → name: pokemonapp
# Change: description accordingly
```

### Update `values.yaml`, `deployment.yaml`, `service.yaml`

Replace all references to `bankapp` with `pokemonapp`, and update image names, ports, and config as needed.

### Deploy the Second App

```bash
helm install pokemonapp pokemonapp
```

### Upgrade when image changes

```bash
helm upgrade pokemonapp pokemonapp
```

---

## Full Command Reference

```bash
helm create <chart-name>              # Scaffold a new chart
helm install <release> <chart>        # First-time deploy
helm upgrade <release> <chart>        # Update existing deploy
helm list                             # List all releases
helm history <release>                # Show revision history
helm rollback <release> <revision>    # Roll back to a revision
helm uninstall <release>              # Delete a release
helm lint <chart>                     # Validate chart syntax
helm template <chart>                 # Render templates locally (dry-run)
helm diff upgrade <release> <chart>   # Preview changes before upgrade (plugin)
```

---