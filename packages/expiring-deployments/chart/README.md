# Expiring Deployments Helm Chart

## What it installs
1. A **CronJob** that runs the helper `run.sh` script (bundled via `dtzar/helm-kubectl`) to evaluate every namespace and Helm lease marker.
2. A dedicated **ServiceAccount**, **ClusterRole**, and **ClusterRoleBinding** with namespace/Helm cleanup permissions. Namespace-scoped `RoleBindings` can be emitted when `rbac.scope=namespaces` and `rbac.namespaces` contains the target list.
3. ConfigMaps that hold both the helper script (`scripts`) and the serialized default values (`config`), making the logic observable and editable.

## Values
| Key | Description | Default |
| --- | ----------- | ------- |
| `schedule` | Cron schedule for the cleanup job | `"0 * * * *"` |
| `dryRun` | When `true`, logs deletions but never mutates the API | `true` |
| `features.namespaces` / `features.helmReleases` | Toggle namespace vs Helm release sweeps | `true` |
| `selectors.managedNamespaceRegex` | Regex allowlist for namespace names | `"^pr-|^preview-|^sandbox-"` |
| `selectors.managedNamespaceLabels` | Label selector for namespaces | `{}` |
| `safety.protectedNamespaces` | Always-skipped namespaces | `kube-system`, `kube-public`, `default`, `prod`, `production` |
| `safety.maxDeletionsPerRun` | Circuit breaker on combined deletes | `20` |
| `safety.requireOwnerAnnotation` | Force `expiring-deployments.io/owner` presence | `false` |
| `helm.uninstallTimeout` | Timeout for `helm uninstall` calls | `"5m"` |
| `helm.uninstallWait` | Append `--wait` when uninstalling | `true` |
| `rbac.scope` | `cluster` or `namespaces` mode | `cluster` |

## TTL metadata contract
Choose one expiration method per resource:
- **Absolute**: `expiring-deployments.io/expires-at=RFC3339`.
- **Relative**: `expiring-deployments.io/ttl-hours="24"` plus optionally `expiring-deployments.io/created-at`; the script falls back to `metadata.creationTimestamp`.

All managed objects must set `expiring-deployments.io/enabled="true"`. Helm releases require a ConfigMap lease template with `expiring-deployments.io/type=helm-release` and `expiring-deployments.io/ttl-hours` (or `expires-at`).

## Helm lease marker example
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ printf "%s-ttl-lease" .Release.Name | trunc 63 | trimSuffix "-" }}
  labels:
    expiring-deployments.io/type: helm-release
    app.kubernetes.io/instance: {{ .Release.Name }}
    app.kubernetes.io/managed-by: "Helm"
  annotations:
    expiring-deployments.io/enabled: "true"
    expiring-deployments.io/ttl-hours: {{ .Values.cleanup.ttlHours | default 24 | quote }}
    expiring-deployments.io/owner: {{ .Values.cleanup.owner | default "unknown" | quote }}
```

## Logging & safe behavior
Run logs include counts of scanned namespaces, Helm leases, deletions, and reasons for skips (protected namespace, allowlist mismatch, not expired). The script enforces the `maxDeletionsPerRun` circuit breaker and leaves the Helm lease marker in place when an uninstall fails so future runs retry.

## Development & packaging
- Render charts: `helm template expiring-deployments packages/expiring-deployments/chart`.
- Package: `npx nx run expiring-deployments:helm` → `dist/charts/expiring-deployments-0.1.3.tgz`.
- Publish: use `cr package`/`cr upload` to push releases and update `index.yaml` on GitHub Pages.
