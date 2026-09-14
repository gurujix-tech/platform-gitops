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
observability/
  prometheus/           # Prometheus server + scrape config
  grafana/              # Grafana + datasource + service-orders dashboard
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

Grafana talks to Prometheus in-cluster at `http://prometheus.observability.svc.cluster.local:9090` (no need to port-forward Prometheus for Grafana itself).

## Bootstrap (kind)

1. Argo CD installed on kind (namespace `argocd`).
2. Chart on GitHub: `service-orders` → `deploy/helm/service-orders`.
3. **App-of-apps (preferred):**
   ```sh
   kubectl apply -f bootstrap/root-app.yaml
   ```
   Root syncs everything under `apps/` from this repo into `argocd`.
4. Or apply a single child directly (early learning):
   ```sh
   kubectl apply -f apps/service-orders.yaml
   ```

## Drift / self-heal demo

Git says `replicaCount: 1`. Then:

```sh
kubectl scale deploy/service-orders --replicas=2
# within ~seconds Argo selfHeal returns replicas to 1
kubectl get deploy service-orders
```

## Repo

```text
https://github.com/gurujix-tech/platform-gitops
```
