# Security exception process (Phase 7 wind-up)

**Purpose:** Allow a temporary, owned bypass of a Gurujix security control without silently weakening the platform.

Applies to (examples):

- Kyverno `ClusterPolicy` exclude / Audit instead of Enforce for one workload
- `.trivyignore` / pip-audit ignore for a known CVE
- Relaxing Helm `securityContext` defaults for a justified case

## Rules

1. **Default is deny / fail.** Exceptions are rare.
2. Every exception has an **owner**, **reason**, **risk acceptance**, and **expiry date** (max **90 days**; renew explicitly).
3. Exceptions live in **Git** (PR review), not only in chat or a ticket comment.
4. Expired exceptions are **removed or renewed** in a follow-up PR — no open-ended ignores.

## How to request

Open a PR that includes:

| Field | Required |
| --- | --- |
| Control | e.g. `require-workload-security-baseline`, Trivy OS, pip-audit |
| Resource | repo / namespace / image / policy name |
| Reason | why the control cannot be met yet |
| Risk | what could go wrong while excepted |
| Compensating control | e.g. network isolation, shorter TTL, Audit mode |
| Owner | GitHub handle / team |
| Expiry | `YYYY-MM-DD` (≤ 90 days out) |

Use the template below in the PR description **and** in the ignore/exclude file comment.

## File conventions

| Control | Where to record |
| --- | --- |
| Trivy | `.trivyignore` with `# owner: … expiry: … reason: …` |
| pip-audit | documented ignore + same comment block in PR |
| Kyverno | policy `exclude` match + comment in `policy/*.yaml` + this process |
| Helm securityContext | values override in a named env values file + PR table |

## Template (copy into PR)

```text
### Security exception
- Control:
- Resource:
- Reason:
- Risk:
- Compensating control:
- Owner:
- Expiry (YYYY-MM-DD):
```

## After expiry

1. Re-run the control (CI / deny drill).
2. Either fix the root cause, renew with a new expiry via PR, or remove the exception and enforce.
