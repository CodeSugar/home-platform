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