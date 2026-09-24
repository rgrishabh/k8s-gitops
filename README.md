# k8s-gitops

Argo CD source of truth for the single-node Kubernetes cluster on `rgrishabh.in`.

## Layout

    apps/jenkins/       live manifests for the `jenkins` namespace
    apps/dashboard/     live manifests for the `kubernetes-dashboard` namespace
    argocd/             the Argo CD Application definitions themselves

## How this was created

These manifests were **exported from the running cluster**, not taken from the
upstream chart or addon defaults. That is deliberate. Both workloads had been
customised by hand, and pointing Argo CD at upstream sources would have reverted:

- `apps/dashboard/deployment-kubernetes-dashboard.yaml` — `--enable-skip-login`
  and `--disable-settings-authorizer` are **removed**. Restoring them would make
  the internet-facing Dashboard a one-click cluster-admin takeover.
- `apps/dashboard/service-kubernetes-dashboard.yaml` — `NodePort 32000`, which the
  host nginx proxies. Reverting to ClusterIP breaks `k8s.rgrishabh.in`.
- `apps/jenkins/configmap-jenkins.yaml` — `apply_config.sh` ends with
  `... /var/jenkins_plugins/ || true`. Without that, the init container is not
  idempotent and Jenkins hangs on every pod restart.
- `apps/jenkins/configmap-jenkins-jenkins-jcasc-config.yaml` — `location.url` is
  `https://jenkins.rgrishabh.in/`.

## Deliberately NOT in this repo

Secrets (Jenkins admin credentials, the Dashboard's cluster-admin service-account
token, dashboard certs/csrf/key-holder) and the `jenkins` PVC. The PVC is excluded
so Argo CD can never prune it — its PV has `reclaimPolicy: Delete`, so pruning it
would destroy `JENKINS_HOME`.

## Safety rails on the Applications

Manual sync only — no `automated`, no `selfHeal`. `Prune=false`. No
`resources-finalizer`, so deleting an Application orphans resources rather than
cascade-deleting them.

## InfraSight (apps/infrasight)

Kustomize base plus a `prod` overlay. `main` deploys to namespace `infrasight`
and is published at `https://infrasight.rgrishabh.in`.

The `images:` block in `overlays/prod/kustomization.yaml` is the interface
between CI and CD: Jenkins rewrites `newTag` after a successful build, commits,
and Argo CD syncs the result. Nothing in the pipeline touches the cluster
directly.

`infrasight-secrets` (POSTGRES_PASSWORD, JWT_SECRET, ADMIN_PASSWORD) is created
out-of-band with kubectl and is deliberately absent from this repo.

Two optional features are disabled in `base/configmap.yaml`: Tor vantage points
and Chromium screenshots. Both are optional by the application's own design and
both are the likeliest way to exhaust a 7.5 GB single node.
