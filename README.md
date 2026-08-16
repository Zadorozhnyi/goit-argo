# goit-argo

This is what I want the cluster to look like. Argo CD reads it and makes the
cluster match. I never apply anything here with kubectl.

```
namespace/
├── application/
│   ├── ns.yaml            the application namespace
│   └── demo-nginx.yaml    Deployment (3 replicas) + ClusterIP Service
└── infra-tools/
    └── ns.yaml            the namespace Argo CD itself runs in
```

## How it gets picked up

An ApplicationSet named `namespaces-appset` lives in the cluster and scans
`namespace/*` here. Each directory becomes one Application: `namespace/application`
turns into `ns-application` and deploys into the `application` namespace,
`namespace/infra-tools` turns into `ns-infra-tools`. The directory name is the
target namespace, so adding another one means `mkdir namespace/<name>`, drop
manifests in, push. No Terraform, no cluster access.

Sync is automated with prune and self-heal, and Argo CD re-reads this repo every
60 seconds. My last push took 32 seconds to show up as a running pod.

Argo CD itself and the ApplicationSet come from Terraform, in
[goit-mlops-hw-07](https://github.com/Zadorozhnyi/goit-mlops-hw-07).

## Checking it

```bash
kubectl get applications -n infra-tools
kubectl get deploy,pods -n application
kubectl -n application port-forward deployment/demo-nginx 8081:80
```

The last one puts nginx on http://localhost:8081.

## Careful with infra-tools

`namespace/infra-tools/ns.yaml` describes the namespace Argo CD runs in, and
Terraform created that namespace first. Two owners for one object, so the
manifest carries

```yaml
argocd.argoproj.io/sync-options: Prune=false,Delete=false
```

Without it a prune could delete the namespace that hosts the controller doing
the pruning. Leave the annotation alone.

The repo is public so Argo CD needs no credentials, no deploy key, no secret.
