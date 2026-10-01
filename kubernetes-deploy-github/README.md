# GitHub Deploy

## Overview
This project implements a GitOps-style webhook receiver that runs inside a Kubernetes cluster: it exposes an HTTP `/webhook` endpoint for a GitHub repository webhook, validates the payload signature, and — on a `push` event — downloads any added/modified files from the repo via the GitHub API, decodes them as Kubernetes `Deployment` manifests, and applies them (create or update) to the cluster. It also polls pods until the expected replica count is `Running`, closing the loop between a `git push` and a live deployment.

## Files
- `go.mod` / `go.sum` — module `github.com/kaif/kubernetes-deploy-github` (Go 1.26.0), depends on `github.com/google/go-github/v45`, `golang.org/x/oauth2`, and the `k8s.io/api`, `k8s.io/apimachinery`, `k8s.io/client-go` Kubernetes client libraries.
- `main.go` — program entry point: `getClient` builds a Kubernetes client either via in-cluster config (`rest.InClusterConfig`, used in real deployments) or from the local `~/.kube/config` kubeconfig (used when `inCluster` is `false`, e.g. local testing); `getGitHubClient` builds an authenticated `github.Client` using an OAuth2 static token from the `GITHUB_TOKEN` env var (or an unauthenticated client if empty); `main` wires up a `server` struct with the Kubernetes client, `WEBHOOK_SECRET` env var, and GitHub client, registers its `webhook` handler on `/webhook`, and starts an HTTP server on `:8080`; `deploy` decodes a byte slice as a Kubernetes `Deployment` manifest and creates it if missing or updates it if it already exists in the `default` namespace; `waitForPods` polls the `default` namespace every 5 seconds until the deployment's pods are all `Running` and match the expected replica count.
- `server.go` — defines the `server` type (`client`, `githubClient`, `webhookSecretKey`) and its `webhook` HTTP handler: it validates the incoming GitHub webhook payload signature with `github.ValidatePayload` using `webhookSecretKey`, parses the event type, and on a `*github.PushEvent` collects added/modified filenames via `getFiles`, downloads each file's contents from the repo via `githubClient.Repositories.DownloadContents`, then calls `deploy` to apply it to the cluster; other event types (e.g. `*github.Hook`) are acknowledged or rejected with a 500. `getFiles` extracts a de-duplicated list of added/modified filenames from a slice of `*github.HeadCommit`.
- `server_test.go` — unit test `TestGetFiles` verifies that `getFiles` correctly extracts a single added filename (`test.txt`) from a mock `HeadCommit`.
- `Dockerfile` — multi-stage build: stage 1 (`golang:1.18-alpine`) compiles a static (`CGO_ENABLED=0`) Linux binary named `github-deploy`; stage 2 (`alpine:latest`) copies the binary into a minimal runtime image with `ca-certificates`/`bash`/`curl` installed, and sets it as the `ENTRYPOINT`.
- `.dockerignore` — excludes the `.git` directory and all `*.yaml` files from the Docker build context.
- `deploy.yaml` — Kubernetes manifests: a `Deployment` (`github-deploy`, 1 replica) running image `wardviaene/github-deploy`, exposing container port 8080, injecting `GITHUB_TOKEN` and `WEBHOOK_SECRET` from a Secret named `github-deploy` (keys `github-token` and `webhook-secret`); a `LoadBalancer` `Service` exposing port 8080; a `ServiceAccount` named `github-deploy`; and a `Role`/`RoleBinding` in the `default` namespace granting that service account `get/list/watch/create/update/patch/delete` on `apps` `deployments`.

## Prerequisites
- Go 1.26.0 or later (per `go.mod`) if building/testing locally; Docker's build stage uses `golang:1.18-alpine`.
- A running Kubernetes cluster. In-cluster (pod) execution uses the pod's service account (`rest.InClusterConfig`); local execution falls back to `~/.kube/config`.
- The `github-deploy` `ServiceAccount`/`Role`/`RoleBinding` from `deploy.yaml` applied so the pod can manage Deployments in the `default` namespace.
- Environment variables required at runtime:
  - `WEBHOOK_SECRET` — shared secret used to validate incoming GitHub webhook signatures.
  - `GITHUB_TOKEN` — a GitHub personal access token used to authenticate API calls to download repo contents (if empty, an unauthenticated/rate-limited client is used).
- A GitHub repository configured with a webhook pointing at this service's `/webhook` endpoint (content type `application/json`, secret matching `WEBHOOK_SECRET`), sending `push` events.
- Docker, if building/deploying the container image.

## How to Run
Run locally against your current kubeconfig context (uses local `~/.kube/config`, not in-cluster config):
```
$env:WEBHOOK_SECRET = "mysecret"
$env:GITHUB_TOKEN = "your-github-token"
go run .
```
This starts an HTTP server on port `:8080` with a `/webhook` endpoint.

Run the tests:
```
go test ./...
```

Build the Docker image:
```
docker build -t github-deploy .
```

Deploy to Kubernetes (after creating the secret below):
```
kubectl apply -f deploy.yaml
```

## Create secret
```
kubectl create secret generic github-deploy --from-literal=webhook-secret=mysecret --from-literal=github-token=token
```

## Notes
- `webhookSecretKey` and the GitHub token are read from environment variables (`WEBHOOK_SECRET`, `GITHUB_TOKEN`), never hardcoded — ensure the `github-deploy` Secret values match what you configure on the GitHub webhook.
- The server only reacts meaningfully to `push` events; other event types return a 500 with a log message rather than a graceful no-op.
- Deployments are always applied to the `default` namespace, matching the RBAC `Role` scope in `deploy.yaml`.
- `waitForPods`/`deploy` logic is shared conceptually with the `kubernetes-deploy` project but here manifests come from files fetched dynamically from GitHub instead of a local `app.yaml`.
- Only one automated test exists (`TestGetFiles` in `server_test.go`); the HTTP webhook handler and Kubernetes deploy logic are not covered by automated tests.
- The `.dockerignore` excludes `*.yaml` files from the build context, so `deploy.yaml` (and any other YAML) is not baked into the image — it's applied separately via `kubectl apply`.
