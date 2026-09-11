# platform-gitops

Desired state for the Gurujix platform clusters (GitOps).

**Phase 5:** Argo CD on local **kind** (`gurujix`) reconciles apps declared here.

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
```

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
