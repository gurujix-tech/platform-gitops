# service-orders failure drill (Phase 6e / 6f)

**Goal:** detect an outage with Prometheus (target + alert) and recover using a short procedure.

**Alert:** `ServiceOrdersDown` — `up{job="service-orders"} == 0` for 30s  
UI: Prometheus → **Status → Alerts**  
Grafana: dashboard **service-orders** → panel **ServiceOrdersDown alert** (PromQL `ALERTS{...}`; Explore works too)

## Preconditions

- `service-orders` Running in `default`
- Prometheus scraping it (Targets → UP)
- Optional: pause Argo sync so selfHeal does not restore replicas mid-drill:

```sh
kubectl patch application service-orders -n argocd --type merge \
  -p '{"spec":{"syncPolicy":null}}'
```

## Drill

### 1. Baseline

```sh
kubectl get deploy,pods -n default -l app.kubernetes.io/name=service-orders
kubectl port-forward -n observability svc/prometheus 9090:9090
# Status → Targets → service-orders = UP
# Status → Alerts → ServiceOrdersDown inactive
```

### 2. Inject failure

```sh
kubectl scale deploy/service-orders -n default --replicas=0
```

Wait ~45–60s (scrape interval + `for: 30s`).

### 3. Detect

| Signal | Expect |
| --- | --- |
| Targets | `service-orders` **DOWN** |
| Alerts | `ServiceOrdersDown` **firing** |
| Grafana | **ServiceOrdersDown alert** panel = FIRING (or Explore: `ALERTS{alertname="ServiceOrdersDown"}`) |
| API | port-forward / curls fail |

### 4. Recover

```sh
kubectl scale deploy/service-orders -n default --replicas=1
kubectl rollout status deploy/service-orders -n default
```

Confirm Targets **UP**, alert inactive, `POST /orders` works.

### 5. Restore GitOps (if you paused sync)

```sh
cd platform-gitops
kubectl apply -f apps/service-orders.yaml
```

If you never paused Argo, selfHeal may already have restored `replicaCount` from Git — still valid recovery evidence.

## On-call style checklist

1. Confirm alert / target DOWN  
2. `kubectl get pods -n default -l app.kubernetes.io/name=service-orders`  
3. `kubectl describe deploy/service-orders -n default`  
4. Logs if Pods exist: `kubectl logs -n default deploy/service-orders --tail=50`  
5. Restore replicas / fix image  
6. Verify `up{job="service-orders"} == 1` + a test request  
7. Note timeline (symptom → cause → fix → verify)

## Out of scope here

Alertmanager → Slack/PagerDuty (add later). This drill is learning evidence for the portfolio.
