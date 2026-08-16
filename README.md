# goit-argo

Desired state of the EKS cluster. Argo CD watches this repository and makes the
cluster match it. Nothing here is applied with kubectl by hand.

```
namespace/
├── application/
│   ├── ns.yaml            the application namespace
│   └── demo-nginx.yaml    Deployment (2 replicas) + ClusterIP Service
└── infra-tools/
    └── ns.yaml            the namespace Argo CD itself runs in
```

## How it is picked up

An ApplicationSet called `namespaces-appset` lives in the cluster and scans
`namespace/*` here. Every directory it finds becomes one Argo CD Application:

| directory | Application | deploys into |
|---|---|---|
| `namespace/application` | `ns-application` | namespace `application` |
| `namespace/infra-tools` | `ns-infra-tools` | namespace `infra-tools` |

The directory name is also the target namespace, so adding a new one is just
`mkdir namespace/<name>`, drop manifests in, push. No Terraform run, no cluster
access needed.

Sync is automated with prune and self-heal, and the reconciliation interval is
60 seconds, so a push shows up in the cluster within about a minute.

The ApplicationSet and Argo CD itself are created by Terraform, in
[goit-mlops-hw-07](https://github.com/Zadorozhnyi/goit-mlops-hw-07).

## Checking it worked

```bash
kubectl get applications -n infra-tools
kubectl get deploy,pods -n application
kubectl -n application port-forward deployment/demo-nginx 8081:80
```

The last one puts the nginx page on http://localhost:8081.

## One thing to be careful with

`namespace/infra-tools/ns.yaml` describes the namespace Argo CD is running in,
and Terraform created that namespace too. The manifest carries

```yaml
argocd.argoproj.io/sync-options: Prune=false,Delete=false
```

so that a prune can never remove the namespace hosting the controller doing the
pruning. Leave that annotation alone.

The repository is public on purpose. Argo CD then needs no credentials, no
deploy key and no repo secret to read it.
