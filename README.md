# py-bay-fast-api-gitops

ArgoCD app-of-apps GitOps repo for [py-bay-fast-api](https://github.com/BoeingPo/py-bay-fast-api). Mirrors the
`go-ledger-x-gitops` pattern: ArgoCD watches this repo's `apps/` directory and self-manages everything else.

## What's deployed

| Application | Source | Namespace |
|---|---|---|
| `py-bay-fast-api-app-of-apps` | this repo, `apps/` | `argocd` |
| `py-bay-fast-api` | [`py-bay-fast-api`](https://github.com/BoeingPo/py-bay-fast-api) repo, `k8s/` | `py-bay-fast-api` |
| `ts-preact-bay` | [`ts-preact-bay`](https://github.com/BoeingPo/ts-preact-bay) repo, `k8s/` | `py-bay-fast-api` |
| `postgres` | this repo, `postgres/` | `py-bay-fast-api` |
| `dynamodb-local` | this repo, `dynamodb-local/` | `py-bay-fast-api` |

`ts-preact-bay` (the frontend) shares the `py-bay-fast-api` namespace rather than getting its own — it's the
same logical product, and the cluster is tight on resources, so there's no benefit to splitting it out.

The root Application is named `py-bay-fast-api-app-of-apps`, not `app-of-apps` — this cluster also runs
go-ledger-x, whose own root Application already uses the plain name `app-of-apps` in the same `argocd`
namespace.

Redis, Kafka, and any other future services get added here the same way: a new `apps/<name>.yaml` Application
pointing either at a path in this repo (raw manifests) or at another service repo's `k8s/` directory.

Postgres + DynamoDB Local add their own resource requests on top of whatever else is already running on this
VPS (it also hosts go-ledger-x) — see the headroom note in
[`py-bay-fast-api-infra`'s README](../py-bay-fast-api-infra/README.md#resource-headroom) before syncing.

## Bootstrap (one-time)

1. Push this repo (public), then run `terraform apply` in
   [`py-bay-fast-api-infra`](../py-bay-fast-api-infra) — default vars (`join_existing_cluster = true`) just
   verify the cluster and register `py-bay-fast-api-app-of-apps`, no other changes. Or skip Terraform
   entirely and apply it by hand on a cluster that already has ArgoCD installed:
   ```bash
   kubectl apply -f apps/app-of-apps.yaml -n argocd
   ```
2. Create the app secret once ArgoCD has created the `py-bay-fast-api` namespace (ArgoCD does not manage
   secrets in this repo — they're never committed in plaintext):

   ```bash
   kubectl create secret generic py-bay-fast-api-secrets -n py-bay-fast-api \
     --from-literal=postgres-password=<choose-a-password> \
     --from-literal=jwt-secret=<choose-a-jwt-secret> \
     --from-literal=aws-access-key-id=local \
     --from-literal=aws-secret-access-key=local
   ```

   (`aws-access-key-id`/`aws-secret-access-key` only need to be real AWS credentials if you swap
   `dynamodb-local` for real AWS DynamoDB later.)
3. Watch sync status: `kubectl get applications -n argocd`

## Local minikube testing

This repo is only needed for the ArgoCD/server path. For a quick local loop against minikube without
installing ArgoCD at all, use `py-bay-fast-api`'s own `make dev-up` / `make dev-deploy` — it deploys
self-contained copies of Postgres + DynamoDB Local directly, no dependency on this repo.
