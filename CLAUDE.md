# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

GitOps definition of a single-node, bare-metal Kubernetes homelab (kubeadm). There is no application code, build or test suite — the repo is plain Kubernetes YAML reconciled by Flux. Nothing is changed by hand on the cluster; a commit to `main` is the deploy.

## How changes reach the cluster

- `infrastructure/flux/flux-system/home-platform.yaml` defines the `GitRepository` (github.com/CodeSugar/home-platform, branch `main`) and a single `Kustomization` with `path: ./infrastructure/flux` and `prune: true`.
- There is **no `kustomization.yaml`**: Flux auto-generates one, so every `.yaml` anywhere under `infrastructure/flux/` is applied. Adding a file deploys it; deleting a file (or moving it out) prunes those resources.
- `infrastructure/disabled/` is outside that path — move an app there to turn it off while keeping its manifest.
- A GitHub webhook (`receiver.yaml`, host `webhook-flux.codesugar.mx`) triggers reconcile on push; polling (GitRepository every 1h) is only a fallback. The Kustomization re-applies every 10m for drift correction.
- Each app is one multi-document YAML file (Namespace, Deployment, Service, ListenerSet, HTTPRoute, …) in `infrastructure/flux/apps/`.

Validating locally (no cluster needed): `kubectl apply --dry-run=client -f <file>`. Checking reconcile on the cluster: `kubectl -n flux-system get gitrepositories,kustomizations` and `kubectl -n flux-system describe kustomization home-platform`. Note the code-server ServiceAccount is cluster read-only (no secrets, no writes).

## Traffic path and the per-app exposure pattern

- **main-gateway** (`infrastructure/flux/cilium/gateway.yaml`, namespace `gateway-system`, Cilium GatewayClass, LB IP `192.168.1.210` via Cilium L2 announcements + LB IPAM). It only defines the HTTP :80 listener and `allowedListeners: from: All`.
- **Each app attaches its own `ListenerSet`** in its own namespace with an HTTPS :443 listener for `<name>.codesugar.mx`, annotated `cert-manager.io/cluster-issuer: letsencrypt-prod` and `acme.cert-manager.io/http01-parentreffallback: "true"`, with a unique `certificateRefs` secret name per host. The app's `HTTPRoute` uses `parentRefs` → that `ListenerSet` with `sectionName: https`. Copy an existing app (e.g. `apps/code-server.yaml`) when adding one. Raw TCP (e.g. Forgejo SSH) uses an extra TCP listener + `TCPRoute`.
- **cert-manager** issues via HTTP-01 through main-gateway (`cert-manager/cluster-issuer.yaml`; `letsencrypt-staging` exists for testing — LE rate limits apply).
- **Access control** — `cilium/lan-only-policy.yaml` is a `CiliumClusterwideNetworkPolicy` on the gateway's `reserved:ingress` identity: LAN (`192.168.0.0/16`) and one trusted IP get everything; the internet gets only ACME challenge paths plus an explicit allow-list of public hostnames. **A new host is LAN-only unless added to that list.**
- **SSO (optional per app)** — see `docs/guides/k8s-auth.md` and `test/whoami-private.yaml` as the reference. The main-gateway HTTPRoute points at `envoy-internal` (Envoy Gateway, ClusterIP-only tier in `cilium/envoy-gateway/`), a second HTTPRoute on `internal-gateway` points at the app, and a `SecurityPolicy` ext-auths against Authelia (`auth/`, users in LLDAP). Also requires adding the app namespace to the `allow-routes-to-envoy-internal` ReferenceGrant in `envoy-gateway.yaml` and a SecurityPolicy→authelia ReferenceGrant in `auth`. If the host is public, `auth.codesugar.mx` must be public too (it already is).
- Namespaces with Flux's `networkPolicy: true` (flux-system) need explicit `CiliumNetworkPolicy` allowing `fromEntities: ingress`, otherwise gateway traffic is dropped (see `flux-web.yaml`, `receiver.yaml`).

## Conventions

- **Storage is ZFS via `hostPath`**, not PVCs: `/zpool-ssd/k8s/<app>/<dir>` with `type: Directory` (the directory must already exist on the host, so creating it is a manual step to mention). HDD pool holds media. Deployments with hostPath use `strategy: Recreate`.
- **Secrets are never committed.** They're created out-of-band with `kubectl create secret`; the manifest that consumes one carries a comment with the command (e.g. `auth/authelia.yaml`, `flux-system/receiver.yaml`).
- **Helm charts go through Flux** (`HelmRelease` + `HelmRepository`/`OCIRepository`), pinned to versions. CRDs for Gateway API (`api-gateway/`) and Envoy Gateway (`cilium/envoy-gateway/crds/`) are vendored as rendered YAML, and the Envoy Gateway HelmRelease uses `crds: Skip` — keep the chart version and vendored CRDs in sync (render command is in the header of `envoy-gateway.yaml`).
- Explanatory comments in manifests explain *why* (network-policy workarounds, header trust, etc.); keep that style.
- `docs/guides/` holds the install/bootstrap runbooks (kubeadm, ZFS, Cilium, cert-manager, auth, Flux Operator); update the relevant guide when changing how a component works.
