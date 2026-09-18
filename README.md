# helm-multi-chart

A Fleet prototype that deploys several external Helm charts, one Bundle per
chart, onto a single local k3d cluster. External means the chart is pulled from
an upstream Helm repository at build time (`helm.repo` plus chart name plus a
pinned `helm.version`), not vendored into this repository.

First chart is Traefik. It routes with Gateway API, so the Gateway API CRDs are a
separate Bundle that Traefik depends on.

## Layout

```
charts/                            tenant agnostic bundle definitions, no targets
  gateway-api-crds/                raw manifests, Gateway API v1.6.1 Standard channel
  traefik/                         external chart plus a values override file
tenants/
  local/gitrepos/landing-zone.yaml one GitRepo listing every chart path
```

`charts/` holds what to deploy, `tenants/` holds where to deploy it. The GitRepo
is applied with kubectl or CI, never deployed by another GitRepo.

Ordering is declared, not scripted: `charts/gateway-api-crds/fleet.yaml` carries
`provides: gateway-api-crds`, and `charts/traefik/fleet.yaml` declares a
`dependsOn` selector on that label. Fleet holds the Traefik Bundle until the CRD
Bundle reports Ready.

## Prerequisites

`../check.sh` verifies all of the cluster side items below against the current
kube context. It is read only and exits non zero when something required is
missing.

1. A k3d cluster on Kubernetes 1.30 or newer. The Gateway API Standard bundle
   ships a `ValidatingAdmissionPolicy`, which is only GA from 1.30, and the
   Traefik chart requires 1.25 or newer.
2. The cluster must be created with the k3s bundled Traefik disabled, otherwise
   it competes for host ports 80 and 443, and with the pinned nodePorts mapped:

   ```
   k3d cluster create fleet-dev \
     --k3s-arg "--disable=traefik@server:*" \
     -p "8080:30000@server:0" \
     -p "8443:30001@server:0"
   ```

3. Fleet installed, so that `fleet-local` holds `Cluster/local` and
   `ClusterGroup/default`.
4. Two Secrets in the `traefik` namespace, created out of band. Neither belongs
   in git, see the Sealed Secrets item in `../todo.md`.

   ```
   kubectl create namespace traefik

   kubectl -n traefik create secret tls local-selfsigned-tls \
     --cert=<path to cert> --key=<path to key>

   kubectl -n traefik create secret generic dashboard-auth-secret \
     --type=kubernetes.io/basic-auth \
     --from-literal=username=<user> --from-literal=password=<from your password manager>
   ```

   Without `local-selfsigned-tls` the websecure listener never becomes
   programmed, and since plain HTTP is redirected to HTTPS the cluster then
   serves nothing.

5. If the Gateway API CRDs were ever applied to this cluster by hand, delete them
   before the first Fleet run. Helm refuses to adopt resources that carry no
   ownership metadata of its own.

## Deploying

Set `spec.repo` in `tenants/local/gitrepos/landing-zone.yaml` to the git remote
this repository is pushed to, it is currently `TBD`. `spec.paths` assumes this
directory is the repository root; if the tree is committed as a subdirectory,
prefix every path entry.

```
kubectl apply -f tenants/local/gitrepos/landing-zone.yaml
```

## Validation

Before pushing, build the bundles offline. There is no `--dry-run` on
`fleet apply`, use `-o -` to inspect.

```
fleet apply -n fleet-local -o - gw-api-crds charts/gateway-api-crds/
fleet apply -n fleet-local -o bundle-traefik.yaml traefik charts/traefik/
fleet target --bundle-file bundle-traefik.yaml --dump-input-list > bd.yaml
fleet deploy --input-file bd.yaml --dry-run
```

The rendered Traefik output must actually contain a `GatewayClass` and a
`Gateway`. A chart that guards those behind an API discovery check would omit
them silently rather than fail.

After applying:

```
kubectl -n fleet-local get gitrepo,bundle
kubectl get crd | grep gateway.networking.k8s.io     # expects 10
kubectl -n traefik get pods,svc,gateway
curl -k https://dashboard.docker.localhost:8443      # expects 401 without credentials
```

Watching the Traefik Bundle stay unready while the CRD Bundle is still
reconciling is the proof that the `dependsOn` wiring works. It is worth looking
at once rather than assuming.

## Notes

- `helm.version` is pinned to an exact chart version. An empty value floats to
  whatever is latest at build time, and a semver constraint is re-evaluated on
  every git change.
- The GitRepo polls on the default interval. Webhooks are preferred once a
  repo and branch mapping is stable, but a laptop is not reachable by a git
  provider.
- `charts/gateway-api-crds/fleet.yaml` sets `keepResources: true`, so removing
  the Bundle does not cascade delete every Gateway and HTTPRoute on the cluster.
  Teardown is an explicit `kubectl delete -f` of the vendored manifest.
