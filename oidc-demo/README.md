# OIDC Demo

## Overview

This project implements a minimal OpenID Connect (OIDC) identity provider from scratch in Go, together with a small demo relying-party application that authenticates against it. The provider implements the OpenID Connect discovery document (`/.well-known/openid-configuration`), the authorization endpoint (`/authorization`), an HTML login form and POST handler (`/login`), the token endpoint (`/token`), a JWKS endpoint (`/jwks.json`), and a userinfo endpoint (`/userinfo`). The authorization code flow is implemented end-to-end: a client redirects a user to `/authorization`, the user is redirected to `/login` to authenticate with a hardcoded username/password, a short-lived authorization code is issued and redirected back to the client's `redirect_uri`, and the client exchanges that code at `/token` for a signed ID token and access token. Tokens are RS256-signed JWTs (via `github.com/golang-jwt/jwt/v4`) using an RSA private key that is generated once (via the `ssh-demo` package's `GenerateKeys`) and persisted to `enckey.pem`, with the matching public key exposed through the JWKS endpoint so relying parties can verify signatures. There are two entrypoints: `cmd\server` is the actual OIDC provider/authorization server (listens on `:8080`, reads `config.yaml` and `enckey.pem`), and `cmd\appserver` is a demo relying-party application (listens on `:8081`) that performs discovery, redirects the user to the provider, exchanges the returned code for tokens, validates the ID/access tokens against the provider's JWKS, and calls the userinfo endpoint.

## Architecture

- **`pkg\oidc`** — client-side OIDC helper types and functions shared by both server and client code: discovery document parsing (`ParseDiscovery`), cryptographically secure random string generation (used for state, session IDs, and authorization codes), and JSON types for the discovery document, token response, and JWKS.
- **`pkg\server`** — the OIDC provider server implementation. Contains the `server` struct (holding the RSA private key, loaded `Config`, and in-memory maps of pending login requests and issued authorization codes) and one file per HTTP endpoint (`authorization.go`, `login.go`, `token.go`, `jwks.go`, `discovery.go`, `userinfo.go`), plus `config.go`/`types.go` for YAML config parsing and `http.go` which wires up the `net/http` routes and starts the server.
- **`pkg\users`** — a tiny in-memory user store (`Auth` and `GetAllUsers`) used by the login and userinfo handlers; contains a single hardcoded demo user.
- **`cmd\server`** — the OIDC provider entrypoint: loads `config.yaml`, generates/loads an RSA signing key (`enckey.pem`), and starts the HTTP server on port `8080`.
- **`cmd\appserver`** — the relying-party demo app entrypoint: a browser-facing app on port `8081` that drives the OIDC authorization code flow against the provider and displays the userinfo response.

## Files

### Root
- `go.mod` — module `github.com/kaif/oidc-demo` (Go 1.25.4), depends on `github.com/golang-jwt/jwt/v4`, `github.com/kaif/ssh-demo` (via the workspace), and `gopkg.in/yaml.v3`.
- `go.sum` — dependency checksums for the above modules and their transitive dependencies (`golang.org/x/crypto`, `golang.org/x/sys`, `golang.org/x/term`, `gopkg.in/check.v1`).
- `config.yaml` — runtime configuration for the provider: base `url` and a map of registered `apps` (client ID/secret, issuer, allowed redirect URIs).
- `Dockerfile` — multi-stage build that compiles `cmd/server` into a static binary on `golang:1.18-alpine`, then copies the binary and `config.yaml` into a minimal `alpine:latest` runtime image, running it as `/app/oidc-demo-server`.
- `.gitignore` — ignores `*.pem` files (e.g. the generated `enckey.pem` signing key).

### cmd\appserver
- `main.go` — relying-party demo app listening on `:8081`; `index` starts the login flow (discovery + state generation + redirect to the provider's authorization endpoint), `callback` handles the redirect back, exchanges the code for tokens, and calls the userinfo endpoint.
- `jwt.go` — `getTokenFromCode` posts the authorization code to the provider's token endpoint, parses the returned ID/access tokens as JWTs, and verifies their signatures using the provider's JWKS (`getPublicKeyFromJwks` builds an `rsa.PublicKey` from the JWKS `n`/`e` values by `kid`).
- `jwt_test.go` — spins up an `httptest` server that mimics the token and JWKS endpoints (signing tokens with a freshly generated RSA key) and verifies `getTokenFromCode` correctly exchanges a code and validates the resulting claims.

### cmd\server
- `main.go` — OIDC provider entrypoint: verifies `config.yaml` exists and reads it, generates an RSA key pair via `ssh.GenerateKeys()` and persists it to `enckey.pem` on first run (or reads it back on subsequent runs), then calls `server.Start` on `:8080` with the parsed config.

### pkg\oidc
- `discovery.go` — `ParseDiscovery` fetches and JSON-decodes an OIDC discovery document from a given URL.
- `rand.go` — `GetRandomString` generates a URL-safe, base64-encoded cryptographically random string of a given byte length (used for `state`, session IDs, and authorization codes).
- `types.go` — shared JSON types: `Discovery` (the `.well-known/openid-configuration` document), `Token` (token endpoint response), and `Jwks`/`JwksKey` (JWKS document).

### pkg\server
- `authorization.go` — implements `/authorization`: validates `client_id`, `redirect_uri`, `scope`, `response_type=code`, and `state`, checks the client is registered and the redirect URI is whitelisted, stores a pending `LoginRequest` keyed by a random session ID, and redirects to `/login`.
- `authorization_test.go` — tests that `/authorization` returns a redirect (`Location` header) to the login page for a valid request.
- `config.go` — `ReadConfig` unmarshals the YAML config bytes into a `Config` struct, recording any parse error on `Config.LoadError`.
- `discovery.go` — implements `/.well-known/openid-configuration`, returning an `oidc.Discovery` document describing the provider's endpoints and supported scopes/response types/signing algorithms/claims.
- `http.go` — defines the `server` struct (private key, config, pending login requests, issued codes), `newServer`, and `Start`, which registers all HTTP routes (`/authorization`, `/token`, `/login`, `/jwks.json`, `/.well-known/openid-configuration`, `/userinfo`) and starts the given `*http.Server`; also defines the shared `returnError` helper.
- `http_test.go` — global test setup (`TestMain`) that generates an in-memory RSA key and a `testConfig` with one registered app used by all other `pkg\server` tests; `TestStart` boots the real server and checks every route responds without a 404/5xx.
- `jwks.go` — implements `/jwks.json`, parsing the server's RSA private key and returning its public key (`n`, `e`) as a JWKS document with `kid: "0-0-0-1"`.
- `login.go` — implements `/login`: on GET, serves the embedded `templates/login.html` form (with the session ID injected); on POST, authenticates the submitted credentials via `pkg/users.Auth`, issues a random authorization code tied to the login request, stores it, and redirects to the client's `redirect_uri` with `code` and `state`.
- `login_test.go` — drives the authorization → login (GET then POST) flow and asserts a valid redirect with an authorization `code` is returned after a successful login.
- `jwks.go`/`token.go` share the `kid` value `"0-0-0-1"` used to sign and identify keys — noted for consistency, not a separate file.
- `token.go` — implements `/token`: validates `grant_type=authorization_code`, looks up the stored code (checking it hasn't expired after 10 minutes, and that `client_id`/`client_secret`/`redirect_uri` match), then mints and RS256-signs an ID token and an access token (JWTs with `iss`/`sub`/`aud`/`exp`/`nbf`/`iat` claims) and returns them as a JSON `oidc.Token`.
- `token_test.go` — drives the full authorization → login → token exchange flow and verifies the returned ID token is a valid, verifiable JWT.
- `types.go` — `Config`/`AppConfig` (parsed YAML config) and `LoginRequest` (in-memory state tracked between the authorization, login, and token steps).
- `userinfo.go` — implements `/userinfo`: validates the `Authorization: Bearer` JWT against the provider's public key, checks the token's audience matches `<issuer>/userinfo`, and returns the matching `users.User` record as JSON.
- `userinfo_test.go` — drives the full authorization → login → token → userinfo flow and verifies a valid userinfo JSON response is returned for a valid access token.

### pkg\server\templates
- `login.html` — Bootstrap-styled HTML login form (username/password) posted to `/login`, carrying the session ID via a hidden `sessionID` field; embedded into the server binary via `//go:embed`.

## Configuration

`config.yaml` is loaded by `cmd\server\main.go` and parsed by `pkg\server\config.go` (`ReadConfig`, using `gopkg.in/yaml.v3`) into a `server.Config` struct (defined in `pkg\server\types.go`):

```yaml
url: "http://localhost:8080"          # the provider's base URL, used as `iss` and to build endpoint URLs
apps:
  app1:                               # arbitrary key naming a registered relying-party app
    clientID: "1-2-3-4"
    clientSecret: "secret"
    issuer: "http://localhost:8080"
    redirectURIs: ["http://localhost:8081/callback"]
```

Each entry under `apps` becomes an `AppConfig` (`ClientID`, `ClientSecret`, `Issuer`, `RedirectURIs`); the authorization endpoint looks up the app by `client_id` and only allows `redirect_uri` values present in `RedirectURIs`, and the token endpoint validates `client_id`/`client_secret`/`redirect_uri` against the same entry.

## Prerequisites

- Go **1.25.4** or later (as declared in `go.mod`).
- This module is part of the repository's Go workspace (`go.work`) and depends on the sibling module `github.com/kaif/ssh-demo`, resolved locally via the workspace — no extra setup needed if building from the workspace root.
- A `config.yaml` file must exist in the working directory when starting `cmd\server` (the process exits if it's missing).
- No environment variables are required for `cmd\server`. `cmd\appserver` reads `OIDC_ENDPOINT`, `CLIENT_ID`, and `CLIENT_SECRET` from the environment.
- Docker (optional), if building/running via the provided `Dockerfile`.

## How to Run

1. **Start the OIDC provider (`cmd\server`)** — run from the `oidc-demo` directory so it can find `config.yaml` (an `enckey.pem` RSA key is auto-generated on first run):
   ```powershell
   cd oidc-demo
   go run .\cmd\server
   ```
   The provider listens on `http://localhost:8080` (per `config.yaml`'s `url` and the hardcoded `:8080` address in `cmd\server\main.go`). Discovery document: `http://localhost:8080/.well-known/openid-configuration`.

2. **Start the demo relying-party app (`cmd\appserver`)** in a second terminal, setting the env vars to match `config.yaml`'s `app1` entry:
   ```powershell
   cd oidc-demo
   $env:OIDC_ENDPOINT = "http://localhost:8080"
   $env:CLIENT_ID = "1-2-3-4"
   $env:CLIENT_SECRET = "secret"
   go run .\cmd\appserver
   ```
   This listens on `http://localhost:8081`.

3. **Log in via the browser**: open `http://localhost:8081`, click **Login**, which redirects through `/authorization` → `/login` on the provider. Sign in with the hardcoded demo user credentials `edward` / `password` (see `pkg\users\auth.go`). You'll be redirected back to `http://localhost:8081/callback`, which exchanges the code for tokens and displays the userinfo JSON.

4. **Run tests**:
   ```powershell
   go test ./...
   ```

## Notes

- This is a teaching/demo project — it is **not production-ready** (in-memory session/code storage, a single hardcoded user, no HTTPS, no key rotation, permissive CORS/security handling absent).
- JWTs (ID tokens and access tokens) are signed with RS256 using an RSA private key generated once via the `ssh-demo` module's `GenerateKeys()` and persisted unencrypted to `enckey.pem` (gitignored); the corresponding public key is exposed at `/jwks.json` under a single fixed `kid` (`"0-0-0-1"`) for relying parties to verify signatures.
- The `pkg\server\templates` directory (embedded at build time with `//go:embed templates/*`) currently contains only `login.html`, a simple Bootstrap-based login form whose `$SESSIONID` placeholder is substituted server-side before being served.
