# helm-multi-chart

A Fleet prototype that deploys several external Helm charts, one Bundle per
chart, onto a single local k3d cluster. External means the chart is pulled from
an upstream Helm repository at build time (`helm.repo` plus chart name plus a
pinned `helm.version`), not vendored into this repository.

Traefik routes with Gateway API, so the Gateway API CRDs are a separate Bundle
that Traefik depends on.

## Layout

```
charts/                            tenant agnostic bundle definitions, no targets
  gateway-api-crds/                raw manifests, Gateway API v1.6.1 Standard channel
  cert-manager/                    external chart, CRDs and Gateway API enabled in values
  cert-manager-issuer/             local chart, cluster wide issuers, per tenant ACME contact
  traefik-tls/                     raw manifest, the Certificate for the Traefik listener
  traefik/                         external chart plus a values override file
  external-dns/                    external chart, in-memory provider for now
  sealed-secrets/                  external chart, controller for committed SealedSecrets
tenants/
  local/gitrepos/landing-zone.yaml one GitRepo listing every chart path
```

`charts/` holds what to deploy, `tenants/` holds where to deploy it. The GitRepo
is applied with kubectl or CI, never deployed by another GitRepo.

## Ordering

Ordering is declared, not scripted. Each bundle carries a `provides:` label and
dependent bundles select on it, so Fleet holds a Bundle until the one it depends
on reports Ready:

```
gateway-api-crds ──► cert-manager ──► cert-manager-issuer ──► traefik-tls ──┐
        │                                                                   │
        └───────────────────────────────────────────────────────────────────┤
                                                                            ▼
                                             external-dns  ◄──────────  traefik

sealed-secrets     (independent, nothing waits on it yet)
```

Traefik waits on two things: the CRDs, because its Gateway provider needs them,
and `traefik-tls`, because its websecure listener cannot become Programmed
without the Secret that bundle produces.

`external-dns` depends on Traefik rather than on the CRDs directly, because it
watches the Gateway and HTTPRoutes that Traefik programs, and Traefik already
gates on the CRDs.

cert-manager depends on the CRDs for a less obvious reason: it probes for
Gateway API support only when the controller process starts. If it came up
first, the `gatewayHTTPRoute` solver would stay inactive until something
restarted the pod, with no error to explain why.

## Per tenant values

`charts/cert-manager-issuer` is the one chart defined in this repository rather
than pulled from upstream. It has to be a chart, not raw manifests, because the
ACME contact address differs per tenant and `targetCustomizations` can only
override Helm values, so there must be a template to substitute into.

The shape is one shared chart plus one entry per tenant in `fleet.yaml`:

```yaml
targetCustomizations:
  - name: playground
    helm:
      values:
        acme:
          email: acme-playground@example.internal
    clusterSelector:
      matchLabels:
        tenant: playground
```

Onboarding a tenant is therefore one entry here plus one `tenant: <name>` label
on that tenant's `Cluster`, not a new file and never a second copy of the chart.
The label is data plumbing for picking values, not a security boundary, the
namespace is still what isolates tenants.

To make the local cluster match:

```
kubectl -n fleet-local label clusters.fleet.cattle.io local tenant=playground
```

Two traps worth knowing, both verified with `fleet target` rather than assumed:

- **The catch-all entry is load bearing.** Bundle targets are an exclusive list.
  A cluster that matches no entry gets no BundleDeployment at all, so the bundle
  silently does not deploy there instead of falling back to `values.yaml`. The
  final `clusterSelector: {}` entry is what makes the defaults actually apply,
  and under the default `FirstMatch` mode it has to stay last or it would
  swallow every tenant above it.
- **Never vary `helm.chart`, `helm.repo` or `helm.version` this way.** Values
  only. Two tenants needing genuinely different chart versions is a separate
  path under `charts/`, not a customization.

## Placeholders that need real values

Both are marked TBD in the files:

- `charts/cert-manager-issuer` carries a placeholder ACME contact address, in
  `values.yaml` as the catch-all default and in `fleet.yaml` per tenant.
  cert-manager registers an ACME account as soon as the
  ClusterIssuer exists, not when the first Certificate is requested, and Let's
  Encrypt rejects contact domains without a public suffix. So this issuer will
  report NotReady until a real platform operated address is filled in. Nothing
  depends on the bundle, so it blocks nothing else.

  Note that a working email still would not let this issue a certificate for
  `dashboard.docker.localhost`. Let's Encrypt only issues for names under a
  valid public suffix, and `.localhost` is reserved by RFC 6761, so the order is
  refused before validation is even attempted. The issuer is structural
  boilerplate for a future real environment. For local TLS, a selfSigned or CA
  ClusterIssuer is the option that actually works.
- `charts/external-dns/values.yaml` runs the `inmemory` provider and filters on
  a placeholder zone. It reconciles normally but writes records nowhere, which
  is deliberate: every real provider needs a credential, and that waits for
  Sealed Secrets.

## Prerequisites

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
4. One Secret in the `traefik` namespace, created out of band. It stays manual
   until it is sealed with the controller from `charts/sealed-secrets`.

   ```
   kubectl -n traefik create secret generic dashboard-auth-secret \
     --type=kubernetes.io/basic-auth \
     --from-literal=username=<user> --from-literal=password=<from your password manager>
   ```

   The listener certificate is no longer on this list. `charts/traefik-tls`
   issues it through cert-manager, which replaces the old
   `openssl req -x509` plus `kubectl create secret tls` step. If that manual
   Secret still exists on the cluster, delete it so cert-manager owns the name
   cleanly:

   ```
   kubectl -n traefik delete secret local-selfsigned-tls --ignore-not-found
   ```

   Traffic still fails closed without a certificate: plain HTTP is redirected to
   HTTPS, so until the Certificate is issued the cluster serves nothing.

5. If the Gateway API CRDs were ever applied to this cluster by hand, delete them
   before the first Fleet run. Helm refuses to adopt resources that carry no
   ownership metadata of its own.

## Trusting the local CA

`charts/cert-manager-issuer` bootstraps a self signed root CA and a `local-ca`
ClusterIssuer. Every local certificate is signed by that one root, so trusting it
once removes the `curl -k` and the browser warnings for every hostname:

```
kubectl -n cert-manager get secret local-ca-tls -o jsonpath='{.data.tls\.crt}' \
  | base64 -d > local-ca.crt
```

Import `local-ca.crt` into the OS or browser trust store. Unlike the previous
hand made certificate, the issued certificates also carry proper
subjectAltNames, which browsers require and a bare `CN=` subject does not
provide.

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
