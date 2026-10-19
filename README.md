# homelab

GitOps configuration for my single-node [k3s](https://k3s.io) homelab. Everything running on the cluster is declared here and reconciled by [Flux](https://fluxcd.io). A push to `main` is a deploy.

## Stack

- **k3s**: lightweight Kubernetes
- **Flux**: syncs this repo to the cluster
- **SOPS + age**: secrets are committed encrypted and decrypted by Flux in-cluster
- **Renovate**: keeps image tags and Helm charts up to date through pull requests
- **kube-prometheus-stack**: Prometheus and Grafana for monitoring

## Layout

```
clusters/staging/        Flux entry point: the Kustomizations that point at the folders below
infrastructure/          Cluster-wide controllers (Renovate)
monitoring/              kube-prometheus-stack (controllers + configs)
apps/
  base/<app>/            Plain manifests per app: namespace, deployment, service, ...
  staging/<app>/         Overlay that adds the encrypted secrets and pulls in the base
  staging/kustomization.yaml   Which apps are enabled
```

Flux applies three Kustomizations in order: `infrastructure` → `monitoring` → `apps`.

## Apps

| App | What it is |
| --- | --- |
| [Forgejo](https://forgejo.org) + runner | Git hosting, CI and container registry for my own projects |
| PostgreSQL | Shared database for the apps that need one |
| [Firefly III](https://www.firefly-iii.org) | Personal finance |
| [Audiobookshelf](https://www.audiobookshelf.org) | Audiobooks and podcasts |
| [Navidrome](https://www.navidrome.org) | Music streaming |
| [Linkding](https://linkding.link) | Bookmarks |
| [LubeLogger](https://lubelogger.com) | Vehicle maintenance log |
| [Wallos](https://wallosapp.com) | Subscription tracker |
| [Grocy](https://grocy.info) | Groceries and household |
| timing-tower, ritregistratie, pr-everyday | Small self-built apps, images built on Forgejo |

Some apps have manifests in `apps/base` but are disabled in `apps/staging/kustomization.yaml`.

## Adding an app

1. Create `apps/base/<app>/` with a `namespace.yaml`, the workload and a `kustomization.yaml` listing them.
2. Create `apps/staging/<app>/kustomization.yaml` that references `../../base/<app>` and any secrets.
3. Add `<app>` to `apps/staging/kustomization.yaml`.
4. Commit and push. Flux picks it up within its sync interval, or right away with:

   ```sh
   flux reconcile kustomization apps --with-source
   ```

## Secrets

Secrets are standard Kubernetes `Secret` manifests whose `data`/`stringData` values are encrypted with SOPS using an age key. Only the encrypted form is ever committed. The private key lives in the cluster as the `sops-age` secret in `flux-system`, and Flux uses it to decrypt during reconciliation.

To add or edit a secret, with the age private key available locally:

```sh
# encrypt a new secret in place (rules come from a local .sops.yaml)
sops --encrypt --in-place apps/staging/<app>/secret.yaml

# edit an existing one; it is re-encrypted on save
sops apps/staging/<app>/secret.yaml
```

Check that every value shows as `ENC[...]` before committing. Private keys and decrypted copies (`*.agekey`, `*.dec.yaml`) are git-ignored.
