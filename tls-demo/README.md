# TLS Demo

## Overview
This project demonstrates TLS and mutual TLS (mTLS) in Go. It includes a Cobra-based CLI (`tls`) for generating a self-signed CA, RSA private keys, and CA-signed certificates from a YAML config, plus example servers/clients that use those artifacts: a plain TLS server, an mTLS server/client pair that verifies client certificates, and a Let's Encrypt-backed server using `autocert`.

## Files
- `go.mod` / `go.sum` — module `github.com/kaif/tls-demo`, Go 1.25.4, depends on `github.com/spf13/cobra`, `golang.org/x/crypto`, and `gopkg.in/yaml.v2`.
- `tls.yaml` — CLI config file: defines the CA certificate (`serial`, `validForYears: 10`, subject fields, common name `CA Certificate`) and two named certs: `go-demo.localtest.me` (with `dnsNames: [go-demo.localtest.me, go-demo-2.localtest.me]`) and `go-demo-client.localtest.me`, each with its own serial/validity/subject.
- `.gitignore` — ignores generated `*.pem`, `*.crt`, `*.key` files at the root.
- `pkg\cert\pem.go` — `PemToX509` decodes a PEM block and parses it into an `*x509.Certificate`.
- `pkg\cert\types.go` — `CACert`, `Cert`, and `CertSubject` structs that mirror the `tls.yaml` schema (serial, validity, subject fields, DNS names).
- `pkg\cert\x509.go` — `CreateCACert` builds and writes a self-signed CA key/cert; `CreateCert` builds a certificate signed by a given CA key/cert; internal `createCert` generates a 4096-bit RSA key and an X.509 certificate (self-signed if `IsCA`, otherwise signed by the CA), PEM-encoding both.
- `pkg\cmd\root.go` — defines the Cobra root command `tls`, a persistent `--config`/`-c` flag (default `tls.yaml`), and loads/unmarshals the YAML config into a package-level `Config` (with `CACert` and a `Cert` map) on init.
- `pkg\cmd\create.go` — defines the `create` parent subcommand (a grouping command with no `Run`, used to attach `ca`, `cert`, and `key` subcommands).
- `pkg\cmd\ca.go` — `create ca` subcommand: flags `--key-out`/`-k` (default `ca.key`) and `--cert-out`/`-o` (default `ca.crt`); calls `cert.CreateCACert` using the CA config from `tls.yaml`.
- `pkg\cmd\cert.go` — `create cert` subcommand: flags `--key-out`/`-k` (default `server.key`), `--cert-out`/`-o` (default `server.crt`), `--name`/`-n` (required, must match a key under `certs:` in `tls.yaml`), `--ca-key` (default `ca.key`, required) and `--ca-cert` (default `ca.crt`, required); reads the CA key/cert files and calls `cert.CreateCert` to produce a CA-signed certificate.
- `pkg\cmd\key.go` — `create key` subcommand: flags `--key-out`/`-k` (default `key.pem`) and `--key-length`/`-l` (default `4096`); generates and saves a standalone RSA private key.
- `cmd\tls\main.go` — CLI entrypoint; simply calls `cmd.Execute()`.
- `cmd\test-server\main.go` — plain TLS HTTPS server listening on `:443`, serving `it's working` at `/`, using `server.crt`/`server.key`.
- `cmd\mtls-server\main.go` — mTLS HTTPS server on `:443` that loads `ca.crt` into a client CA pool and requires/verifies client certificates (`tls.RequireAndVerifyClientCert`, `MinVersion: tls.VersionTLS13`) using `server.crt`/`server.key`; serves `/` (`it's working`) and `/common-name` (echoes the verified client certificate's `CommonName`).
- `cmd\mtls-client\main.go` — HTTPS client that trusts `ca.crt`, presents its own `client.crt`/`client.key` for mTLS, and issues a GET to `https://go-demo.localtest.me/common-name`, printing the status code and response body.
- `cmd\letsencrypt-server\main.go` — HTTP server using `golang.org/x/crypto/acme/autocert.NewListener("go-demo-test.newtech.academy")` to automatically obtain a real Let's Encrypt certificate for that domain and serve `it's working` at `/`.

## Prerequisites
- Go 1.25.4 or later (per `go.mod`).
- The CA and certificates referenced by the servers/clients (`ca.crt`, `ca.key`, `server.crt`, `server.key`, `client.crt`, `client.key`) must be generated first with the `tls` CLI — none are checked in (`.gitignore` excludes `*.pem`, `*.crt`, `*.key`).
- `test-server`, `mtls-server`, and `letsencrypt-server` bind to port `:443`, which typically requires administrator/root privileges.
- `go-demo.localtest.me` / `go-demo-client.localtest.me` used by the mTLS client rely on the public `localtest.me` wildcard DNS service, which resolves any `*.localtest.me` name to `127.0.0.1` — no `/etc/hosts` edit is needed as long as you have normal DNS resolution.
- `letsencrypt-server` requires a real, publicly reachable domain (as coded: `go-demo-test.newtech.academy`) with DNS pointed at the host and port 80/443 reachable from the internet; it cannot succeed on a local/private machine.

## How to Run
All commands are run from the `tls-demo` directory.

1. Generate the self-signed CA (uses `caCert` section of `tls.yaml`):
   ```
   go run ./cmd/tls create ca --key-out ca.key --cert-out ca.crt
   ```
2. Generate the server certificate, signed by the CA (uses the `go-demo.localtest.me` entry in `tls.yaml`):
   ```
   go run ./cmd/tls create cert --name go-demo.localtest.me --ca-key ca.key --ca-cert ca.crt --key-out server.key --cert-out server.crt
   ```
3. Generate the client certificate for mTLS (uses the `go-demo-client.localtest.me` entry):
   ```
   go run ./cmd/tls create cert --name go-demo-client.localtest.me --ca-key ca.key --ca-cert ca.crt --key-out client.key --cert-out client.crt
   ```
4. Run the plain TLS test server (requires `server.crt`/`server.key`; may require an elevated/administrator shell to bind `:443`):
   ```
   go run ./cmd/test-server
   ```
   Test with: `curl -k https://localhost/`
5. Run the mTLS server instead (requires `ca.crt`, `server.crt`, `server.key`):
   ```
   go run ./cmd/mtls-server
   ```
6. Run the mTLS client from another terminal (requires `ca.crt`, `client.crt`, `client.key`; connects to `https://go-demo.localtest.me/common-name`):
   ```
   go run ./cmd/mtls-client
   ```
   Expected output: `Body (status 200): Your common name: go-demo-client.localtest.me`.
7. (Optional) Generate a standalone RSA key with the `key` subcommand:
   ```
   go run ./cmd/tls create key --key-out key.pem --key-length 4096
   ```
8. (Optional, not runnable locally) `letsencrypt-server` needs a real public domain pointed at the machine:
   ```
   go run ./cmd/letsencrypt-server
   ```

## Notes
- All certs are self-signed by the demo CA; browsers/clients that don't trust `ca.crt` will show certificate warnings for `test-server`/`mtls-server`.
- `mtls-server` only accepts connections presenting a client certificate signed by `ca.crt` (`tls.RequireAndVerifyClientCert`) and negotiates TLS 1.3 minimum.
- The CLI's `--config`/`-c` flag lets you point at a different YAML file; it defaults to `tls.yaml` in the current directory.
- `cmd/letsencrypt-server` is provided as an educational example only — it targets a hardcoded domain (`go-demo-test.newtech.academy`) and requires real public DNS/port access to obtain a certificate from Let's Encrypt.
- Generated key/cert files (`*.pem`, `*.crt`, `*.key`) are gitignored and must be regenerated locally.
