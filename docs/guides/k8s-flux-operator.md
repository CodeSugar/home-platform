# Flux Operator

This service is in charge of GitOps and Flux mantainance.

## How it works

    ```mermaid
flowchart TD
    subgraph install["1 · Install Flux"]
        direction LR
        helm(["helm install"]) --> op["Flux Operator"]
        fi["FluxInstance"] -->|desired setup| op
        op -->|deploys| ctrl["Flux controllers"]
    end

    subgraph sync["2 · Sync from Git"]
        direction LR
        gh[("GitHub<br/>home-platform")] -->|clone| gr["GitRepository<br/>every 1m"]
        gr -->|artifact| ks["Kustomization<br/>./infrastructure/flux<br/>every 5m"]
        ks -->|apply and prune| k8s["Cluster"]
    end

    install -->|controllers run| sync
```

The **Flux Operator** installs and manages Flux itself. You describe the Flux you want in a `FluxInstance`; the operator pulls the matching manifests from `ghcr.io`, deploys the controllers, and upgrades them automatically when a new `2.x` release is published.

From there it's regular Flux. **source-controller** clones `CodeSugar/home-platform` (branch `main`) every minute and stores the commit as an artifact. **kustomize-controller** takes that artifact, builds `./infrastructure/flux`, and applies it to the cluster. New commits are picked up as soon as the source sees them; the `5m` interval is how often Flux re-applies everything to undo manual drift. With `prune: true`, anything you delete from Git is deleted from the cluster.

**helm-controller**, **notification-controller** and **source-watcher** stay idle until you add `HelmRelease`, `Alert`/`Provider` or `ArtifactGenerator` objects.

## Prerequisites

- `kubectl` pointing at the right cluster (`kubectl config current-context`)
- Helm 3.8+ (needed for OCI charts)
- Optional: the [Flux CLI](https://fluxcd.io/flux/installation/#install-the-flux-cli) for nicer status output


## Step 1 — Install the Flux Operator

```bash
helm install flux-operator oci://ghcr.io/controlplaneio-fluxcd/charts/flux-operator \
  --namespace flux-system \
  --create-namespace
```

Check that it's running:

```bash
kubectl -n flux-system get deploy flux-operator
```

## Step 2 — Create the FluxInstance

Save as `flux-instance.yaml`:

```yaml
apiVersion: fluxcd.controlplane.io/v1
kind: FluxInstance
metadata:
  name: flux
  namespace: flux-system
spec:
  distribution:
    version: "2.x"
    registry: "ghcr.io/fluxcd"
    artifact: "oci://ghcr.io/controlplaneio-fluxcd/flux-operator-manifests"
  components:
    - source-controller
    - source-watcher
    - kustomize-controller
    - helm-controller
    - notification-controller
  cluster:
    type: kubernetes
    size: medium
    multitenant: false
    networkPolicy: true
    domain: "cluster.local"
```

The less obvious fields: `version: "2.x"` tracks the latest Flux 2 release, `size: medium` tunes controller concurrency and resource limits, and `networkPolicy: true` adds NetworkPolicies that restrict traffic to the Flux pods.

Apply it and wait until it's ready:

```bash
kubectl apply -f flux-instance.yaml
kubectl -n flux-system wait fluxinstance/flux --for=condition=Ready --timeout=5m
kubectl -n flux-system get fluxinstance
kubectl -n flux-system get deploy
```

You should see `flux-operator` plus the five controllers.

## Step 3 — Connect the Git repository

Save as `home-platform-git.yaml`:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: home-platform
  namespace: flux-system
spec:
  interval: 1m0s
  url: https://github.com/CodeSugar/home-platform.git
  ref:
    branch: main
```

```bash
kubectl apply -f home-platform-git.yaml
kubectl -n flux-system get gitrepositories
```

`READY` should be `True` and `STATUS` should show the latest commit SHA. This works without credentials because the repo is public; if you make it private, see [Private repository](#private-repository).

## Step 4 — Apply the Kustomization

Save as `home-platform-kustomize.yaml`:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: home-platform
  namespace: flux-system
spec:
  interval: 5m0s
  path: ./infrastructure/flux
  prune: true
  sourceRef:
    kind: GitRepository
    name: home-platform
```

```bash
kubectl apply -f home-platform-kustomize.yaml
kubectl -n flux-system get kustomizations
```


## Troubleshooting cheat sheet

```bash
# Overall status reported by the operator
kubectl -n flux-system get fluxreport flux -o yaml

# Sources and Kustomizations
kubectl -n flux-system get gitrepositories,kustomizations

# Why is it failing?
kubectl -n flux-system describe kustomization home-platform
kubectl -n flux-system logs deploy/kustomize-controller
```

## Force a reconciliation with kubectl

`flux reconcile` is only an annotation write, so you never actually need the CLI.
Every Flux resource watches `reconcile.fluxcd.io/requestedAt`: when the value
differs from the `.status.lastHandledReconcileAt` the controller already recorded,
the object is queued immediately instead of waiting for its `interval`. The value
itself is meaningless — it just has to change — and `$(date +%s)` is the convention.

`--field-manager=flux-client-side-apply` is what the CLI uses. Keeping the same
field manager stops kustomize-controller from fighting you over ownership of the
annotation on the next apply.

### Pull the latest commit and apply it

Annotating the Kustomization alone re-applies whatever revision the source has
already fetched. To pick up a commit you just pushed, poke the GitRepository
first and let it report the new SHA before poking the Kustomization:

```bash
# flux reconcile source git home-platform
kubectl -n flux-system annotate --field-manager=flux-client-side-apply --overwrite \
  gitrepository/home-platform reconcile.fluxcd.io/requestedAt="$(date +%s)"

# wait until REVISION shows the new commit, then Ctrl-C
kubectl -n flux-system get gitrepository home-platform -w

# flux reconcile kustomization home-platform
kubectl -n flux-system annotate --field-manager=flux-client-side-apply --overwrite \
  kustomization/home-platform reconcile.fluxcd.io/requestedAt="$(date +%s)"
```

### Helm releases

Same annotation, but the HelmRelease lives in the namespace it was declared in
(`cilium` in `kube-system`, `flux-web` in `flux-system`):

```bash
kubectl -n kube-system annotate --field-manager=flux-client-side-apply --overwrite \
  helmrelease/cilium reconcile.fluxcd.io/requestedAt="$(date +%s)"
```

That is a no-op when the release is already up to date. To push a Helm upgrade
through anyway (`flux reconcile helmrelease --force`), add `forceAt` — it only
counts when `requestedAt` carries the *same* value:

```bash
TOKEN="$(date +%s)"
kubectl -n flux-system annotate --field-manager=flux-client-side-apply --overwrite \
  helmrelease/flux-web \
  reconcile.fluxcd.io/requestedAt="$TOKEN" \
  reconcile.fluxcd.io/forceAt="$TOKEN"
```

If a release burned through its `remediation.retries` and stopped trying,
`resetAt` clears the failure counters — again paired with `requestedAt`:

```bash
TOKEN="$(date +%s)"
kubectl -n kube-system annotate --field-manager=flux-client-side-apply --overwrite \
  helmrelease/cilium \
  reconcile.fluxcd.io/requestedAt="$TOKEN" \
  reconcile.fluxcd.io/resetAt="$TOKEN"
```

### The FluxInstance itself

The operator honours the same annotation, so you can make it re-check for a new
Flux 2.x release without waiting an hour:

```bash
kubectl -n flux-system annotate --overwrite \
  fluxinstance/flux reconcile.fluxcd.io/requestedAt="$(date +%s)"
```

Here `reconcile.fluxcd.io/forceAt` means something different than it does for a
HelmRelease: it migrates every Flux resource in the cluster to its latest API
version. Useful after a major Flux upgrade, not something to run casually.

### Suspend and resume

`flux suspend` / `flux resume` are just a field in the spec:

```bash
kubectl -n flux-system patch kustomization home-platform \
  --type=merge -p '{"spec":{"suspend":true}}'

kubectl -n flux-system patch kustomization home-platform \
  --type=merge -p '{"spec":{"suspend":false}}'
```

The FluxInstance is the exception — it pauses via an operator annotation:

```bash
kubectl -n flux-system annotate --overwrite \
  fluxinstance/flux fluxcd.controlplane.io/reconcile=disabled
```

### Did it actually run?

The controller copies your token into `lastHandledReconcileAt` once it picks the
request up, so comparing the two tells you whether it was seen:

```bash
kubectl -n flux-system get kustomization home-platform \
  -o jsonpath='{.status.lastHandledReconcileAt}{"\n"}'

kubectl -n flux-system wait kustomization/home-platform \
  --for=condition=Ready --timeout=2m
```
