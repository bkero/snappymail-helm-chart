# snappymail-helm-chart

A Helm chart for deploying [SnappyMail](https://snappymail.eu) on Kubernetes
with sane production defaults: a non-root container, a read-only root
filesystem, a stateless data directory rebuilt on every pod start, and
HTTP probes against a cheap `/ping` endpoint.

The container image used by this chart is built and published from
[bkero/docker-snappymail](https://github.com/bkero/docker-snappymail) to
`ghcr.io/bkero/docker-snappymail`.

> **Out of scope.** No database, IMAP, or SMTP server is deployed by this
> chart. SnappyMail talks to whatever external services you point it at via
> the admin panel.

## Installing

Add the chart repository:

```sh
helm repo add snappymail https://bkero.github.io/snappymail-helm-chart
helm repo update
```

Install:

```sh
helm install snappymail snappymail/snappymail \
    --namespace snappymail --create-namespace
```

Or with an ingress, TLS, and a chart-managed admin password:

```sh
helm install snappymail snappymail/snappymail \
    --namespace snappymail --create-namespace \
    --set admin.login=admin \
    --set admin.password='change-me-please' \
    --set ingress.enabled=true \
    --set ingress.className=nginx \
    --set ingress.hosts[0].host=mail.example.com \
    --set ingress.hosts[0].paths[0].path=/ \
    --set ingress.hosts[0].paths[0].pathType=Prefix \
    --set ingress.tls[0].secretName=snappymail-tls \
    --set ingress.tls[0].hosts[0]=mail.example.com
```

For production, prefer keeping the cleartext password out of your values
file by referencing an externally-managed Secret:

```sh
kubectl -n snappymail create secret generic snappymail-admin \
    --from-literal=admin_password='change-me-please'

helm install snappymail snappymail/snappymail \
    --namespace snappymail \
    --set admin.login=admin \
    --set admin.existingSecret=snappymail-admin
```

Then log into `https://mail.example.com/?admin` with `admin` /
`change-me-please` and configure (or override via `config.domains`) your
IMAP/SMTP domains.

## What the chart deploys

| Resource              | Notes                                                                |
|-----------------------|----------------------------------------------------------------------|
| `Deployment`          | Stateless, `/var/lib/snappymail` is an emptyDir, single replica      |
| `Service`             | ClusterIP on port 80 → container port 8888                           |
| `Ingress`             | Optional, standard Helm ingress block                                |
| `ConfigMap`           | Optional, holds `application.ini` + per-domain JSONs                 |
| `Secret`              | Optional, holds the cleartext admin password                         |
| `ServiceAccount`      | No mounted token                                                     |
| `HorizontalPodAutoscaler` | Optional (sticky sessions required)                              |
| `PodDisruptionBudget` | Optional                                                             |

### Security defaults

- `runAsNonRoot: true`, uid 82 (`www-data`)
- `readOnlyRootFilesystem: true` with emptyDirs mounted at every path the
  runtime needs to write to (`/run/snappymail`, `/tmp`, `/var/tmp/nginx`,
  `/var/lib/nginx`, `/var/log/nginx`)
- `capabilities.drop: [ALL]`, `allowPrivilegeEscalation: false`
- `seccompProfile.type: RuntimeDefault`
- `fsGroup: 82` so the PVC is writable by the container user

## Stateless by design

SnappyMail's data tree (`/var/lib/snappymail`) is mounted as an `emptyDir`
and reseeded on every container start. There is **no PVC**. Anything you
want to persist across restarts must be supplied via the chart's
`config.*` and `admin.*` values, which the chart materializes into a
ConfigMap and a Secret and mounts into the pod. The container's entrypoint
overlays them onto the data tree at boot.

This means:

- Per-user state visible only inside SnappyMail (the contacts cache,
  uploaded attachments waiting in /tmp, sieve script drafts) is **not
  preserved** across pod restarts. SnappyMail itself stores user mail in
  the upstream IMAP server, so that part is unaffected.
- Admin-panel changes you make through the UI are **lost** on the next
  restart. To make changes durable, edit `.Values.config.applicationIni`
  / `.Values.config.domains` and re-`helm upgrade`.

If you genuinely need persistence (e.g. for the contacts/attachments
cache), bind your own PVC into the deployment via `extraVolumes` /
`extraVolumeMounts` over the `data` mount path — but for most production
setups, point SnappyMail at a real IMAP server and let the IMAP server
own the state.

Replicas > 1 are not recommended without sticky-session affinity:
SnappyMail's session and file-locking story is best-effort and not
designed for clustered operation.

## Values

The full list lives in
[`charts/snappymail/values.yaml`](charts/snappymail/values.yaml). Notable
sections:

- `image.*` — pulls `ghcr.io/bkero/docker-snappymail:<appVersion>` by default
- `config.applicationIni` — literal contents of `application.ini`
- `config.domains` — map of `<name>` → IMAP/SMTP domain JSON
- `admin.login` / `admin.password` — chart-managed admin credentials
- `admin.existingSecret` / `admin.existingSecretKey` — bring your own Secret
- `env.*` — runtime tunables (`UPLOAD_MAX_SIZE`, `MEMORY_LIMIT`,
  `PHP_FPM_*`, `PHP_TIMEZONE`, `SECURE_COOKIES`)
- `ingress.*` — ingress class, hosts, TLS, annotations (cert-manager friendly)
- `resources.*` — defaults to 100m/256Mi requests, 500m/512Mi limits
- `extraEnv` / `extraEnvFrom` — wire in plugin secrets from Secrets/ConfigMaps
- `extraVolumes` / `extraVolumeMounts` — escape hatch for custom mounts

## Releasing new chart versions

This repo uses
[`helm/chart-releaser-action`](https://github.com/helm/chart-releaser-action).
Bump `version:` in `charts/snappymail/Chart.yaml`, push to `main`, and the
[`Release Helm chart`](.github/workflows/release.yml) workflow will:

1. Lint the chart
2. Package it
3. Create a GitHub Release named `snappymail-<version>` with the `.tgz`
   attached
4. Update `index.yaml` on the `gh-pages` branch

> **One-time setup**: enable GitHub Pages on this repo, set the source to
> the `gh-pages` branch (root), and the chart-releaser will keep `index.yaml`
> up to date from then on.
