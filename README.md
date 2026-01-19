# Expiring Deployments

A reusable Helm chart that installs a CronJob to enforce TTL-based cleanup for temporary namespaces and Helm releases. It is safe-by-default (dry-run, protected namespaces, allowlists) while keeping adoption narrow: annotate resources once, and Expiring Deployments handles removal later.

## Features
- **Namespace TTL sweep**: annotations drive deletion when `expires-at` or `ttl-hours` elapses; protected namespaces and allowlists are honored.
- **Helm lease marker sweep**: Helm releases opt in via a labeled ConfigMap lease resource; when the lease expires, the release is uninstalled.
- **Safety controls**: dry-run default, circuit breaker (`maxDeletionsPerRun`), optional owner annotations, and namespace-scoped RBAC helpers.
- **Observability**: per-run summaries, dry-run indicators, and detailed skip/delete logging inside the helper script.

## Installation
### From the public GitHub Pages repository
```
helm repo add expiring-deployments https://aeyeio.github.io/expiring-deployments
helm repo update
helm install expiring-deployments expiring-deployments/expiring-deployments --version 0.1.3
```

### From the private Harbor OCI registry
```
helm registry login harbor.localhost --username username --password Password1 --insecure
helm install expiring-deployments oci://harbor.localhost/library/expiring-deployments/expiring-deployments --version 0.1.3 --insecure-skip-tls-verify --create-namespace --namespace expiring-deployments
```

## Key chart configuration
Headers from `packages/expiring-deployments/chart/values.yaml` expose:
- `schedule`, `dryRun`, and `features.*` flags to toggle namespace and Helm release sweeps.
- `selectors.managedNamespaceRegex` / `managedNamespaceLabels` to scope the cleanup work.
- `safety.protectedNamespaces`, `maxDeletionsPerRun`, and `requireOwnerAnnotation` for safety hardening.
- Helm uninstall tuning (`helm.uninstallTimeout`, `helm.uninstallWait`) and RBAC scope choices.

## Metadata contract
Annotate namespaces or Helm lease markers with the `expiring-deployments.io` prefix. Required keys:
- `expiring-deployments.io/enabled=true`
- `expires-at` (RFC3339) OR `ttl-hours` plus optional `created-at` (falls back to `creationTimestamp`).

Recommended governance annotations: `owner`, `purpose`, `source`, and the Helm lease marker label `expiring-deployments.io/type=helm-release`.

## Development
- `npx nx run expiring-deployments:helm` packages the chart into `dist/charts/expiring-deployments-0.1.3.tgz`.
- `helm template ...` and `helm install --dry-run` help validate the rendered manifests.
- Chart releases are published with [Helm Chart Releaser](https://github.com/helm/chart-releaser) to a GitHub Pages repo.

## Testing
- CronJob behavior is primarily exercised by Kubernetes dry-runs; logs (stdout) show the evaluated namespaces/releases, reasons for skips, and the run summary.

## Contribution
1. Update charts or docs in this repo.
2. Run `npm ci` and `npx nx run expiring-deployments:helm` to confirm packaging.
3. Push to `main`, then run `cr upload` (with credentials) to publish a new release and index entry.

For private usage, install directly from Harbor (OCI) and comment the same values above.
