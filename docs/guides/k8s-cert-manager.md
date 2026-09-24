# cert-manager

TLS certificates for everything behind the Cilium Gateway, issued by Let's Encrypt
and renewed without anyone touching a manifest.

## How it works

```mermaid
flowchart TD
    subgraph shim["1 · gateway-shim watches each app's ListenerSet"]
        direction LR
        ls["ListenerSet (app namespace)<br/>annotation: cluster-issuer<br/>parentRef: main-gateway"] -->|per HTTPS listener| cert["Certificate<br/>dnsNames = listener hostname"]
        cert -->|secretName| sec[("Secret<br/>kubernetes.io/tls")]
        sec -->|certificateRefs| ls
    end

    subgraph acme["2 · HTTP-01 over the Gateway API"]
        direction LR
        cert2["Certificate"] --> ord["Order / Challenge"]
        ord -->|creates| hr["temporary HTTPRoute<br/>cm-acme-http-solver-xxxxx"]
        hr -->|attaches to :80| gw2["main-gateway"]
        le[("Let's Encrypt")] -->|GET /.well-known/acme-challenge| gw2
        ord -->|validated, then deleted| hr
    end

    shim --> acme
```

Two separate mechanisms do the work.

**gateway-shim** is the automatic part. `main-gateway` itself only has the plain HTTP
port 80 listener and `allowedListeners.namespaces.from: All`. Each app brings its own
HTTPS listener in a `ListenerSet` in its own namespace, attached to `main-gateway`.
Annotate that `ListenerSet` with `cert-manager.io/cluster-issuer` and cert-manager reads
its HTTPS listeners: for each distinct `tls.certificateRefs` secret name it creates and
maintains a `Certificate` in the ListenerSet's namespace, whose DNS names are the
hostnames of the listeners pointing at that secret. You never write a `Certificate` by
hand, and you never renew one.

The important limit: cert-manager reads **listeners** (on a `Gateway` or a
`ListenerSet`), not `HTTPRoute`s. A
hostname that only exists on an `HTTPRoute` gets no certificate. Declaring the listener
is the one manual step per host.

**The HTTP-01 solver** proves ownership. For each challenge cert-manager creates a
throwaway solver pod, service and `HTTPRoute` in the Certificate's namespace, attaches
the route to `main-gateway`'s port 80 listener with the hostname being validated, and
lets Let's Encrypt fetch `/.well-known/acme-challenge/<token>` over plain HTTP. Once the
order is valid the three objects are deleted again.

The app's `ListenerSet` only has an HTTPS listener, so it has nowhere to attach the
solver route. The `acme.cert-manager.io/http01-parentreffallback: "true"` annotation on
the `ListenerSet` makes cert-manager fall back to the issuer's `parentRefs`, which point
at `main-gateway` (see [Issuers](#issuers)). Without it, challenges never become
reachable.

## Prerequisites

- Public `A` record for every hostname pointing at the WAN IP.
- Router forwarding **80** → `192.168.1.210` (required by HTTP-01) and **443** for real
  traffic.
- Gateway API CRDs (v1.6, which includes `ListenerSet`) installed before cert-manager
  starts. They already are, in `infrastructure/flux/api-gateway/`.
- `lan-only-policy` (`infrastructure/flux/cilium/lan-only-policy.yaml`) lets anyone
  on the internet reach `/.well-known/acme-challenge/` on port 80 for **any** hostname.
  That way, LAN-only hosts still get certificates. Keep that rule when you edit the policy.

Wildcards are not possible here: ACME refuses HTTP-01 for `*.codesugar.mx`, so a
wildcard SAN would need a DNS-01 solver instead.

## What is configured

### Gateway API support in cert-manager

`infrastructure/flux/cert-manager/helm-release.yaml`:

```yaml
  values:
    config:
      apiVersion: controller.config.cert-manager.io/v1alpha1
      kind: ControllerConfiguration
      gatewayAPI:
        enabled: true
```

`apiVersion` and `kind` are required by the chart. This is the modern spelling — most
blog posts still show `--enable-gateway-api` or the `ExperimentalGatewayAPISupport`
feature gate, both of which are unnecessary since the gate went beta/on-by-default.

No RBAC work is needed: the `cert-manager-controller-challenges` and
`-ingress-shim` ClusterRoles already carry `gateway.networking.k8s.io` rules for
`gateways`, `httproutes` and `listenersets`, unconditionally.

Setting `config` for the first time adds `--config`, a volume and a volumeMount to the
Deployment, so the pod rolls by itself. **Later** edits only rewrite the ConfigMap — the
chart has no `checksum/config` pod annotation — so those need a nudge:

```bash
kubectl -n cert-manager rollout restart deploy/cert-manager
```

### Issuers

`infrastructure/flux/cert-manager/cluster-issuer.yaml` defines `letsencrypt-staging` and
`letsencrypt-prod`, identical apart from the ACME directory URL and account key secret
name. Both solve HTTP-01 through `main-gateway`:

```yaml
    solvers:
    - http01:
        gatewayHTTPRoute:
          parentRefs:
          - name: main-gateway
            namespace: gateway-system
            kind: Gateway
```

The solver `HTTPRoute` is created in the Certificate's namespace and attaches
cross-namespace, which works because the port 80 listener has
`allowedRoutes.namespaces.from: All`.

Always prove a new setup on staging first. Let's Encrypt allows 50 certificates per
registered domain per week and failed orders still count against you.

## Adding a host

Put a `ListenerSet` in the app's own manifest, next to its `Service`, and commit. The
example below is for an app `grafana` in namespace `monitoring`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: ListenerSet
metadata:
  name: grafana
  namespace: monitoring
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    acme.cert-manager.io/http01-parentreffallback: "true"
spec:
  parentRef:
    name: main-gateway
    namespace: gateway-system
  listeners:
  - name: https
    protocol: HTTPS
    port: 443
    hostname: grafana.codesugar.mx
    tls:
      mode: Terminate
      certificateRefs:
      - name: grafana-codesugar-mx-tls
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: grafana
  namespace: monitoring
spec:
  parentRefs:
  - group: gateway.networking.k8s.io
    kind: ListenerSet
    name: grafana
    sectionName: https
  hostnames:
  - grafana.codesugar.mx
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: grafana
      port: 80
```

That is the whole job. cert-manager creates `Certificate/grafana-codesugar-mx-tls` in
`monitoring`, orders the cert, writes the secret, and renews it from then on. The
secret lives in the same namespace as the `ListenerSet` that references it, so no
`ReferenceGrant` is involved. Adding a host touches only the app's own file, never
`gateway.yaml`.

Give each listener its own secret name. Sharing one secret across listeners merges the
hostnames into a single SAN certificate, which means every new host re-issues the whole
thing; separate names keep the blast radius at one host.

The app's `HTTPRoute` attaches to its `ListenerSet` with `sectionName: https`, not to
`main-gateway` directly. Any `.yaml` in `infrastructure/flux/apps/` works as a template.
The new host is LAN-only until you add it to the public list in `lan-only-policy`.

## Verifying

```bash
# cert-manager picked up the config
kubectl -n cert-manager get deploy cert-manager \
  -o jsonpath='{.spec.template.spec.containers[0].args}'   # expect --config=...
kubectl -n cert-manager logs deploy/cert-manager | grep -i gateway

# issuers registered with ACME
kubectl get clusterissuer                                   # READY=True

# the ListenerSet was accepted by main-gateway
kubectl -n <app-ns> get listenerset <name> -o yaml          # Accepted + Programmed

# the shim turned listeners into Certificates (in the app's namespace)
kubectl -n <app-ns> get certificate,certificaterequest,order,challenge

# watch the temporary solver route come and go
kubectl -n <app-ns> get httproute -w                        # cm-acme-http-solver-xxxxx

# from OUTSIDE the network, while a challenge is pending
curl -sv http://test.codesugar.mx/.well-known/acme-challenge/anything

# Cilium actually programmed the listeners
kubectl -n gateway-system get gateway main-gateway -o yaml  # Programmed + ResolvedRefs
kubectl get ciliumenvoyconfig -A                            # a CEC must exist

# end to end
curl -vI https://test.codesugar.mx                          # "(STAGING) Let's Encrypt"
```

## Switching staging → prod

Change the `cert-manager.io/cluster-issuer` annotation on the app's `ListenerSet` to
`letsencrypt-prod` and commit. cert-manager notices the issuer changed but will not
discard a still-valid certificate, so force a fresh order:

```bash
kubectl -n <app-ns> delete certificate test-codesugar-mx-tls
kubectl -n <app-ns> delete secret test-codesugar-mx-tls
```

The shim recreates the Certificate from the listener within seconds. Re-run the `curl
-vI` check — the issuer should now read `Let's Encrypt` and a browser should trust it.

## Troubleshooting

```bash
# why is a cert stuck?
kubectl -n <app-ns> describe certificate <name>
kubectl -n <app-ns> describe challenge              # the ACME error is here
kubectl -n cert-manager logs deploy/cert-manager -f
```

**Challenge pending forever** — Let's Encrypt cannot reach port 80. Check the A record
resolves to the WAN IP and the router forwards 80 to `192.168.1.210`; test the solver URL
from outside the LAN, not from a machine that resolves the name internally. If the solver
route exists but is not attached to `main-gateway`, the `ListenerSet` is missing the
`http01-parentreffallback` annotation.

**Listener stuck on `ResolvedRefs: False`** — the secret does not exist yet. Expected
until the first order completes; it resolves on its own.

**Port 80 suddenly 404s after adding a ListenerSet** — check
`kubectl get ciliumenvoyconfig -A` still shows a CEC for the gateway. Cilium
[#44123](https://github.com/cilium/cilium/issues/44123) had a wildcard-HTTP-listener
plus specific-hostname-HTTPS-listener combination generate no Envoy config at all, which
breaks HTTP-01 exactly. Fixed before 1.20, but this is the topology that triggered it.

**Secret exists but is `Opaque` and cert-manager refuses to write it** — Cilium
[#45705](https://github.com/cilium/cilium/issues/45705), 1.19.3.x only, fixed before
1.20. Not applicable here, noted so it isn't re-diagnosed from scratch.
