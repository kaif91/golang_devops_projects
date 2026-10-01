# DNS Demo

## Overview
This project demonstrates how to build a minimal recursive DNS resolver in Go from scratch, using the low-level `golang.org/x/net/dns/dnsmessage` package instead of Go's standard `net` DNS lookup functions. It listens for UDP DNS queries, then iteratively resolves them by walking the DNS hierarchy starting at the public root servers, following NS/A referrals until an authoritative answer is found.

## Files
- `go.mod` / `go.sum` — module `github.com/kaif/dns-demo` (Go 1.25.0), depends on `golang.org/x/net` for DNS message parsing.
- `cmd/dns-resolver/main.go` — entry point; opens a UDP listener on port `:53` and spawns a goroutine (`dns.HandlePacket`) per incoming packet.
- `pkg/dns/resolver.go` — core resolver logic: `HandlePacket`/`handlePacket` parse an incoming DNS query and write back a response; `dnsQuery` recursively resolves a question by querying servers, following `NS`/`A` referrals (authorities/additionals) up to 3 iterations, and resolving nameservers that lack glue records by recursively querying for their `A` records; `outgoingDnsQuery` sends a single UDP query to a candidate server and parses the raw response; `getRootServers` parses the hardcoded `ROOT_SERVERS` constant (13 well-known root server IPs) into `net.IP` values.
- `pkg/dns/resolver_test.go` — unit tests: `TestHandlePacket` builds and packs sample DNS queries (`www.google.com.`, `www.amazon.com.`) and feeds them through `handlePacket` using a no-op `MockPacketConn`; `TestOutgoingDnsQuery` performs a real outgoing UDP query to a root server for the NS records of `com.` and asserts a successful, non-empty response.

## Prerequisites
- Go 1.25.0 or later (per `go.mod`).
- Network access to UDP port 53 on the internet (root DNS servers) for both running the resolver and running `TestOutgoingDnsQuery`.
- Binding to UDP port `:53` typically requires elevated/administrator privileges (or root on Linux) since it's a privileged port.
- No environment variables or config files are required.

## How to Run
Build and run the resolver (requires privileges to bind port 53):
```
go run ./cmd/dns-resolver
```
Then query it with a DNS client, e.g.:
```
nslookup www.example.com 127.0.0.1
```
Run the test suite (note: `TestOutgoingDnsQuery` makes a live network call to a root DNS server):
```
go test ./...
```

## Notes
- This is an educational/demo implementation, not production-hardened: it has no caching, no TCP fallback for truncated responses, and a fixed retry count of 3 iterations in `dnsQuery`.
- `outgoingDnsQuery` reads only a single 512-byte UDP datagram per attempt, so very large DNS responses could be truncated.
- The root server list is hardcoded in `ROOT_SERVERS` rather than resolved dynamically.
- `TestOutgoingDnsQuery` requires outbound internet/UDP access to succeed and could fail in restricted network environments.
