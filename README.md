# platform-gitops

Desired state for the Gurujix platform clusters (GitOps).

**Phase 5–6:** Argo CD on local **kind** (`gurujix`) reconciles apps declared here.

## Mental model

> Git is the desired state. Argo CD makes the cluster match Git.  
> NOT: `kubectl` / `helm` as the long-term source of truth on the laptop.

```text
You merge to Git
    → Argo notices (or you Sync)
    → cluster updated
    → if someone kubectl-ed by hand, self-heal can revert (when enabled)
```

## Layout

```text
bootstrap/
  root-app.yaml         # App-of-apps (apply once) → watches apps/
apps/
  service-orders.yaml   # Child Application → Helm chart in service-orders repo
  prometheus.yaml       # Phase 6b → observability/prometheus
  grafana.yaml          # Phase 6c → observability/grafana
  loki.yaml             # Phase 6 → observability/loki
  promtail.yaml         # Phase 6 → observability/promtail
  kyverno.yaml          # Phase 7d → Kyverno Helm chart
  platform-policies.yaml # Phase 7d → policy/ ClusterPolicies
observability/
  prometheus/           # Prometheus server + scrape config for service-orders
  grafana/              # Grafana + datasources + service-orders dashboard
  loki/                 # Loki (log store)
  promtail/             # Promtail DaemonSet (stdout → Loki)
  runbooks/             # Failure drills
policy/                 # Kyverno ClusterPolicies (enforced)
drills/                 # Manual deny/allow drills (not synced by Argo)
docs/                   # Exception process, threat model (Phase 7 wind-up)
```

## Phase 6b — Prometheus scrapes service-orders

**Prerequisites**

1. `service-orders` Pod Running in `default` (Argo app synced).
2. Image includes Phase 6a metrics (`GET /metrics`). Rebuild + load if needed:
   ```sh
   cd service-orders
   docker build -t service-orders:local .
   kind load docker-image service-orders:local --name gurujix
   kubectl rollout restart deploy/service-orders -n default
   ```
3. Push `platform-gitops` so root picks up the Application, **or** apply locally:
   ```sh
   kubectl apply -f observability/prometheus/
   kubectl apply -f apps/prometheus.yaml
   ```

**Verify**

```sh
kubectl get pods -n observability
kubectl port-forward -n observability svc/prometheus 9090:9090
# open http://127.0.0.1:9090 → Status → Targets → service-orders should be UP
# Graph: orders_created_total or rate(http_requests_total{handler!~"/health|/ready"}[1m])
```

Prometheus TSDB data lives on the Pod volume (`emptyDir` here — fine for kind learning).

## Phase 6c — Grafana dashboards

```sh
kubectl apply -f observability/grafana/
kubectl apply -f apps/grafana.yaml   # or push + let root sync

kubectl port-forward -n observability svc/grafana 3000:3000
# open http://127.0.0.1:3000
# login: admin / admin  (kind learning only)
# Dashboards → Gurujix → service-orders
```

Grafana talks to Prometheus in-cluster at `http://prometheus.observability.svc.cluster.local:9090`.

Dashboard includes a **ServiceOrdersDown** stat panel (`ALERTS` metric) — a view of the Prometheus rule, not Grafana Alerting (bell).

## Phase 6e / 6f — Alert + failure drill

- Rule: `observability/prometheus/rules-configmap.yaml` (`ServiceOrdersDown`)
- Runbook: `observability/runbooks/service-orders-failure-drill.md`

Push this repo, then let Argo sync (or `kubectl apply -f observability/prometheus/`). Pause Argo sync before a scale-to-0 drill or selfHeal restores replicas.

## Phase 6 — Loki + Promtail (logs)

```text
service-orders stdout → Promtail (DaemonSet) → Loki → Grafana Explore
```

```sh
# After push (root app picks up apps/loki.yaml + apps/promtail.yaml), or DIY:
kubectl apply -f observability/loki/
kubectl apply -f observability/promtail/
kubectl apply -f apps/loki.yaml
kubectl apply -f apps/promtail.yaml

# Grafana needs a restart after datasource ConfigMap change
kubectl apply -f observability/grafana/
kubectl rollout restart deployment/grafana -n observability

kubectl port-forward -n observability svc/grafana 3000:3000
# Explore → Loki → {namespace="default", pod=~"service-orders.*"}
# Generate traffic: POST /orders, then search for request_id in the JSON line
```

## Phase 7d — Kyverno policy (enforce 7c baseline)

```text
kubectl apply bad Pod in default → Kyverno webhook DENY
service-orders (compliant Helm) → ALLOW
```

1. Push `platform-gitops` (apps/kyverno.yaml, apps/platform-policies.yaml, policy/).
2. Wait until Application `kyverno` is Healthy, then `platform-policies` Synced.
3. Deny drill (not synced by Argo):

```sh
kubectl apply -f drills/phase-7d-bad-pod-root.yaml
# expect: admission webhook denied ... require-workload-security-baseline
```

Policy scope: **namespace `default` only** (Promtail in `observability` stays root for host log paths).

## Phase 7 wind-up — exceptions, threat model, SBOM

- Exception process: `docs/security-exception-process.md` (owner, reason, expiry ≤ 90 days)
- Threat model (golden path): `docs/threat-model-golden-path.md`
- SBOM: CycloneDX artifact from `service-orders` CI (`Trivy SBOM` + upload-artifact; non-blocking)

## Bootstrap (kind)

1. Argo CD installed on kind (namespace `argocd`).
2. Chart on GitHub: `service-orders` → `deploy/helm/service-orders`.
3. **App-of-apps (preferred):**
   ```sh
   kubectl apply -f bootstrap/root-app.yaml
   ```
4. Or apply a single child directly:
   ```sh
   kubectl apply -f apps/service-orders.yaml
   ```

## Drift / self-heal demo

```sh
kubectl scale deploy/service-orders --replicas=2
# within ~seconds Argo selfHeal returns replicas to 1
kubectl get deploy service-orders
```

## Repo

```text
https://github.com/gurujix-tech/platform-gitops
```
