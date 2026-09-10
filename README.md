# platform-gitops

Desired state for the Gurujix platform clusters (GitOps).

**Phase 5d:** Argo CD on local **kind** (`gurujix`) reconciles apps declared here.

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
apps/
  service-orders.yaml   # Argo Application → Helm chart in service-orders repo
```

Later: more Applications, app-of-apps, env folders (`dev/`, `prod/`).

## Bootstrap (kind)

1. Push `service-orders` so `deploy/helm/service-orders` exists on `main` (Argo reads GitHub, not your laptop folder).
2. Install Argo CD on kind (see below or project README notes).
3. `kubectl apply -f apps/service-orders.yaml`
4. Remove any manual Helm release that conflicts:  
   `helm uninstall service-orders -n default` (Argo will recreate from Git).

## Create this GitHub repo

When ready:

```text
https://github.com/gurujix-tech/platform-gitops
```

Push this tree to `main`. Optional later: Argo “app of apps” that watches *this* repo.
