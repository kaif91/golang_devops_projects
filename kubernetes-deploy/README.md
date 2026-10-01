# Kubernetes Deploy

## Overview
This project demonstrates how to deploy a Kubernetes `Deployment` programmatically from Go using `client-go`, instead of shelling out to `kubectl`. It reads a local YAML manifest, decodes it into a typed Kubernetes object, creates or updates the Deployment via the Kubernetes API, and then polls until the expected number of pods are `Running`.

## Files
- `go.mod` / `go.sum` — module `github.com/kaif/kubernetes-deploy` (Go 1.26.0), depends on `k8s.io/api`, `k8s.io/apimachinery`, and `k8s.io/client-go`.
- `main.go` — the program logic: `getClient` builds a Kubernetes REST client from the local `~/.kube/config` kubeconfig file; `deploy` reads `app.yaml`, decodes it with the client-go scheme deserializer into a `*v1.Deployment`, then calls `Get` on the `default` namespace to check if the deployment exists — creating it if not found, or updating it if it does — and returns the pod template labels and expected replica count; `waitForPods` builds a label selector from those labels and polls the `default` namespace's pods every 5 seconds until the number of `Running` pods matches the expected replica count; `main` wires these together and prints the outcome.
- `app.yaml` — a Kubernetes `Deployment` manifest named `helloworld-deployment` running the `wardviaene/k8s-demo` image with 1 replica, labeled `app: helloworld`, exposing container port 3000 (`nodejs-port`).

## Prerequisites
- Go 1.26.0 or later (per `go.mod`).
- A running Kubernetes cluster with a valid kubeconfig at `~/.kube/config` (the code uses `clientcmd.BuildConfigFromFlags` with the default homedir kubeconfig path — no in-cluster config support here).
- The kubeconfig's current context must have permission to `get`/`create`/`update` Deployments and `list` Pods in the `default` namespace.
- `app.yaml` must exist in the current working directory when running the program (it's read via a relative path).
- No environment variables are required.

## How to Run
Build and run directly against your current kubeconfig context:
```
go run main.go
```
This will create (or update, if already present) the `helloworld-deployment` Deployment described in `app.yaml`, then block and print progress until all expected pods report `Running`.

You can also apply the manifest directly with kubectl for comparison/manual use:
```
kubectl apply -f app.yaml
```

## Notes
- The program always targets the `default` namespace; it is hardcoded and not configurable via flags/env vars.
- `waitForPods` loops forever with no timeout — if pods never reach `Running` (e.g., bad image, insufficient cluster resources), the program will hang indefinitely.
- There are no automated tests in this project (no `_test.go` files).
- `app.yaml` references the public image `wardviaene/k8s-demo`, so the cluster must have outbound access to pull it.
