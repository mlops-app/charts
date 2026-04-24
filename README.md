# ITOps Helm Charts

Helm chart repository for [ITOps](https://www.mlops.hu) — a Kubernetes-native IT Operations Platform.

## Usage

```bash
helm repo add itops https://charts.mlops.hu
helm repo update
```

## Available Charts

| Chart | Version | Description |
|---|---|---|
| `itops/itops` | 1.16.1 | Core + UI + bundled PostgreSQL |
| `itops/itops-agent` | 1.4.1 | K8s operator for service discovery and SLA reporting |
| `itops/sla-portal` | 1.3.3 | Standalone public status page (SQLite) |

## Quick Start

### 1. Deploy the platform

```bash
helm install itops itops/itops -n itops --create-namespace
```

Default login: `admin` / `Password123!`.

External exposure:

```bash
helm install itops itops/itops -n itops --create-namespace \
  --set ingress.hosts[0].host=api.yourdomain.com \
  --set ingress.tls[0].hosts[0]=api.yourdomain.com \
  --set uiIngress.hosts[0].host=app.yourdomain.com \
  --set uiIngress.tls[0].hosts[0]=app.yourdomain.com
```

### 2. Connect each cluster with the agent

```bash
helm install itops-agent itops/itops-agent \
  --set node.id="myorg/platform/prod/cluster1" \
  --set itops.url="https://api.yourdomain.com" \
  --set itops.apiKey.value="YOUR_OPERATOR_API_KEY" \
  -n itops --create-namespace
```

`OPERATOR_API_KEY` comes from the platform chart's `secretEnv.ITOPS_SECURITY_OPERATOR_API_KEY`.

Multiple clusters are supported out of the box: each agent uses its own `node.id` prefix (4-level path `org/platform/env/cluster`) and pushes independently. The backend stores every service under `externalId = nodeId/serviceName`, so cross-cluster name collisions are impossible.

---

## GitOps schema — one line minimum

A service's `it-ops.yaml` needs exactly one required field: `path`. The 5-level identifier `organization/platform/environment/cluster/service` uniquely addresses the service across every cluster connected to the platform.

```yaml
# The smallest valid it-ops.yaml
path: "myorg/itops/prod/cluster1/payment-api"
```

That's it. The agent registers the service, the K8s workload health flows automatically. Every other field is opt-in — add a line for each feature you want.

### Progressive disclosure

| Add | Effect |
|---|---|
| `slaGroup: "..."` | SLA tracking turns on; criticality inherited from the group tier |
| `backup: { expected: true, maxAgeDays: 1 }` | Backup tab entry with overdue alerts |
| `type: api` `team: backend` `tags: [api]` | Presentation fields (icon, UI labels, filters) |
| `dependencies.requires: [{ path: "..." }]` | Clickable dep chip in the UI |

### Dependency references are globally unique

Every dep ref uses the same `path:` identifier. No more name collisions between clusters:

```yaml
path: "myorg/itops/prod/eu-west-1/payment-api"
slaGroup: "payment-system"

dependencies:
  requires:
    - path: "myorg/itops/prod/eu-west-1/payment-db"
      critical: true
  usedBy:
    - path: "myorg/itops/prod/us-east-1/payment-frontend"  # different cluster
```

### Loud-over-silent push endpoints

The `/api/v1/storage/report`, `/backup/report`, `/health/report` webhooks now handle malformed payloads by **padding to sentinels + returning warnings** instead of dropping data. A push without `nodeId` lands under an `unknown/unknown/unknown/unknown` hierarchy branch with a red badge in the UI (so the misconfig is visible), and the response body echoes a `warnings[]` array explaining what was normalized. See [monitoring docs](https://www.mlops.hu/pages/docs/monitoring.html) for examples.

---

## Documentation

- Full docs: [https://www.mlops.hu/pages/docs.html](https://www.mlops.hu/pages/docs.html)
- API reference: [https://www.mlops.hu/pages/api.html](https://www.mlops.hu/pages/api.html)
- Chart values: `helm show values itops/itops` / `helm show values itops/itops-agent`
