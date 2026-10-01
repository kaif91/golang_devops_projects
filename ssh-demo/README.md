# SSH Demo

## Overview
This project demonstrates how to build SSH tooling in Go using `golang.org/x/crypto/ssh`. It shows how to generate RSA key pairs in SSH format, run a minimal SSH server that authenticates clients by public key and serves both one-off commands and an interactive shell, and run an SSH client that connects with a private key and executes a remote command.

## Files
- `go.mod` / `go.sum` — module `github.com/kaif/ssh-demo`, Go 1.25.4, depends on `golang.org/x/crypto` and `golang.org/x/term`.
- `.gitignore` — ignores generated `*.pem` and `*.pub` key files.
- `keygen.go` (package `ssh`) — `GenerateKeys()` creates a 4096-bit RSA key pair and returns the private key as a PEM block and the public key in SSH "authorized_keys" format.
- `server.go` (package `ssh`) — `StartServer(privateKey, authorizedKeys)` parses the authorized public keys, configures an `ssh.ServerConfig` with a `PublicKeyCallback` that only accepts known keys, adds the server's host private key, and listens on `0.0.0.0:2022`. For each connection it services the `session` channel: an `exec` request runs `whoami` and returns the result; a `shell`/`pty-req` request opens an interactive terminal (via `golang.org/x/term`) supporting the commands `whoami` and `quit`.
- `cmd\keygen\main.go` — generates a key pair via `ssh.GenerateKeys()` and writes the private key to `mykey.pem` (mode `0600`) and the public key to `mykey.pub` (mode `0644`).
- `cmd\server\main.go` — reads the server's host private key from `server.pem` and the authorized client public key from `mykey.pub`, then calls `ssh.StartServer`.
- `cmd\client\main.go` — reads the client's private key from `mykey.pem` and the expected server public key from `server.pub`, dials `tcp` `localhost:2022` as user `"username"` using public-key authentication and a fixed host key check, opens a session, and runs `whoami`, printing the output.

## Prerequisites
- Go 1.25.4 or later (per `go.mod`).
- Key files must be generated (and correctly named) before the server or client can run — see below. The `.gitignore` confirms no `*.pem`/`*.pub` files are checked in, so they must be created locally.

## How to Run
All commands are run from the `ssh-demo` directory. The keygen tool always writes `mykey.pem`/`mykey.pub`, but the server expects a host key named `server.pem` and the client expects the host's public key as `server.pub`, so you must generate keys twice and rename the first set:

1. Generate the server's host key pair, then rename it:
   ```
   go run ./cmd/keygen
   ren mykey.pem server.pem
   ren mykey.pub server.pub
   ```
2. Generate the client's key pair (this run's `mykey.pem`/`mykey.pub` are used as-is — `mykey.pub` doubles as the server's authorized key):
   ```
   go run ./cmd/keygen
   ```
3. Start the server (reads `server.pem` and `mykey.pub`, listens on `0.0.0.0:2022`):
   ```
   go run ./cmd/server
   ```
4. In another terminal, run the client (reads `mykey.pem` and `server.pub`, connects to `localhost:2022`, runs `whoami`):
   ```
   go run ./cmd/client
   ```
   Expected output: `Output is: You are: username`.

## Notes
- The server binds to all interfaces (`0.0.0.0:2022`); the client always connects to `localhost:2022`.
- Client authentication is public-key only, restricted to whatever key is present in `mykey.pub` at server start.
- Host key verification on the client uses `ssh.FixedHostKey`, so `server.pub` must exactly match the server's host key or the connection will be rejected.
- The interactive terminal (triggered by `pty-req`) only understands `whoami` and `quit`; anything else prints "Command not found".
- This is a teaching demo, not hardened for production use (e.g., the client always authenticates as the hardcoded user `"username"`).
