# Threat model — golden path (Create → deploy) (Phase 7 wind-up)

**Scope:** Gurujix path for a Python service from Backstage scaffolder through CI and GitOps onto **kind** (learning). Not a full STRIDE worksheet for production AWS.

**Assets:** source repos, GitHub Actions secrets/OIDC, container image, cluster desired state (`platform-gitops`), running `service-orders` (and future services), logs/metrics.

```text
Developer / Backstage
    → GitHub (service repo + platform-gitops)
    → CI (lint, tests, gitleaks, pip-audit, Trivy OS, image build)
    → (optional) ECR publish
    → Argo CD sync
    → kind: Pods (Kyverno on namespace default)
```

## Top risks and mitigations (current controls)

| # | Risk | Example | Mitigation in place |
| --- | --- | --- | --- |
| 1 | Secret in Git | API key committed | **Gitleaks** on PR/push |
| 2 | Vulnerable app dependency | Bad version in `requirements.txt` | **pip-audit** fails CI |
| 3 | Vulnerable OS packages in image | Unpatched base image | **Trivy `vuln-type: os`**; bump `FROM` / base |
| 4 | Untrusted code reaches `main` | Malicious PR | Reviews + CI gates before merge |
| 5 | Privileged / root workload | `runAsUser: 0` in `default` | **7c Helm baseline** + **Kyverno Enforce** |
| 6 | Drift from Git | Manual `kubectl` weaken securityContext | Argo **selfHeal**; policy still on create/update |
| 7 | Supply chain / unknown image contents | “What did we ship?” | **SBOM artifact** on CI image build (wind-up) |
| 8 | Over-broad policy break platform | Kyverno blocks Promtail | Policy scoped to **`default` only** |

## Out of scope (later phases)

- Cloud IAM compromise, public ALB exposure, multi-tenant isolation → **Phase 8+**
- Image signing / provenance attestations → when registry + OIDC publish is standard
- Full STRIDE per service → when services leave kind learning

## Residual risk (accepted for kind learning)

- `service-orders:local` via `kind load` is not registry-attested.
- Grafana/Prometheus learning credentials (`admin`/`admin`) are **not** production patterns.
- Kyverno does not yet cover every namespace or `:latest` image bans.

## Review

Revisit this doc when: adding a new admission policy, enabling ECR publish by default, or starting Phase 8 (AWS).
