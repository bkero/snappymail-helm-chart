# snappymail-helm-chart

A Helm chart for deploying [SnappyMail](https://snappymail.eu) on Kubernetes
with sane production defaults: a non-root container, a read-only root
filesystem, a PVC for the data directory, and HTTP probes against a cheap
`/ping` endpoint.

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

Or with an ingress and TLS:

```sh
helm install snappymail snappymail/snappymail \
    --namespace snappymail --create-namespace \
    --set ingress.enabled=true \
    --set ingress.className=nginx \
    --set ingress.hosts[0].host=mail.example.com \
    --set ingress.hosts[0].paths[0].path=/ \
    --set ingress.hosts[0].paths[0].pathType=Prefix \
    --set ingress.tls[0].secretName=snappymail-tls \
    --set ingress.tls[0].hosts[0]=mail.example.com
```

After install, retrieve the auto-generated admin password:

```sh
POD=$(kubectl -n snappymail get pod -l app.kubernetes.io/instance=snappymail \
        -o jsonpath='{.items[0].metadata.name}')
kubectl -n snappymail exec -it "$POD" -- \
    cat /var/lib/snappymail/_data_/_default_/admin_password.txt
```

Then log into `https://mail.example.com/?admin` and configure your IMAP/SMTP
domains, optional contacts database, and any plugins you need.

## What the chart deploys

| Resource              | Notes                                                          |
|-----------------------|----------------------------------------------------------------|
| `Deployment`          | Single replica by default, `Recreate` strategy                 |
| `Service`             | ClusterIP on port 80 → container port 8888                     |
| `Ingress`             | Optional, standard Helm ingress block                          |
| `PersistentVolumeClaim` | RWO PVC mounted at `/var/lib/snappymail`                     |
| `ServiceAccount`      | No mounted token                                               |
| `HorizontalPodAutoscaler` | Optional (only useful with RWX storage)                    |
| `PodDisruptionBudget` | Optional                                                       |

### Security defaults

- `runAsNonRoot: true`, uid 82 (`www-data`)
- `readOnlyRootFilesystem: true` with emptyDirs mounted at every path the
  runtime needs to write to (`/run/snappymail`, `/tmp`, `/var/tmp/nginx`,
  `/var/lib/nginx`, `/var/log/nginx`)
- `capabilities.drop: [ALL]`, `allowPrivilegeEscalation: false`
- `seccompProfile.type: RuntimeDefault`
- `fsGroup: 82` so the PVC is writable by the container user

## Replication

SnappyMail keeps user-visible state on disk under `/var/lib/snappymail`.
With `ReadWriteOnce` storage you must keep `replicaCount: 1`. To run
multiple replicas you need `ReadWriteMany` storage, *and* you should be
aware that SnappyMail is not designed for clustered operation — sessions
and file locking are best-effort.

## Values

The full list lives in
[`charts/snappymail/values.yaml`](charts/snappymail/values.yaml). Notable
sections:

- `image.*` — pulls `ghcr.io/bkero/docker-snappymail:<appVersion>` by default
- `env.*` — runtime tunables (`UPLOAD_MAX_SIZE`, `MEMORY_LIMIT`,
  `PHP_FPM_*`, `PHP_TIMEZONE`, `SECURE_COOKIES`)
- `persistence.*` — PVC sizing and storage class
- `ingress.*` — ingress class, hosts, TLS, annotations (cert-manager friendly)
- `resources.*` — defaults to 100m/256Mi requests, 500m/512Mi limits
- `extraEnv` / `extraEnvFrom` — wire in plugin secrets from Secrets/ConfigMaps

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
