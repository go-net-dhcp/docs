# go-net-dhcp documentation

**A small, dependency-free DHCPv4 server library in pure Go** — the RFC 2131 /
2132 wire codec, a stateless decision core, and a real Linux UDP/67 server, built
with **zero cgo**. The module path is `github.com/go-net-dhcp/dhcp`.

`go-net-dhcp/dhcp` targets the *"one subnet, hand out a lease per known MAC"*
case — a host that owns a bridge / VLAN and needs to answer DHCP for the guests
it spawns on it, **without running an external dnsmasq**. You supply a `Source`
that resolves a client MAC into a `Lease`; the library never persists anything
itself.

!!! success "Status: server library complete"
    RFC 2131 / 2132 **parse + build** (OFFER / ACK / NAK), the **`Decide`** state
    machine (DISCOVER→OFFER, REQUEST→ACK, mismatched requested-IP→NAK, unknown
    MAC / other message types→drop), a real Linux **`SO_BINDTODEVICE`** UDP/67
    server, a cross-platform **`StubServer`**, and an optional **Prometheus**
    metrics adapter. 100% coverage — socket error branches included, via
    injectable syscall seams so nothing needs root — `gofmt` + `go vet` clean, CI
    green across the six 64-bit Go targets.

## Design in one paragraph

The DHCPv4 wire format is a fixed BOOTP header + a 4-byte magic cookie + a block
of TLV options terminated by code 255 — small enough to hand-roll, so the core
module has **zero third-party dependencies**. Everything that does not touch a
socket (`Parse`, `BuildReply`, `Decide`) is **build-tag-free** and portable, and
is therefore unit-tested on every platform. Only the UDP/67 bind lives behind
`//go:build linux`. Metrics are an **optional hook**: pass any `Metrics`
implementation in `Options`; a Prometheus adapter lives in a separate sub-package
so the core stays lean.

## Quick taste

```go
opts := dhcp.Options{
    Interface: "br0",
    ServerIP:  netip.MustParseAddr("10.0.0.1"),
    Source: dhcp.SourceFn(func(mac string) (dhcp.Lease, bool) {
        return dhcp.Lease{
            Yiaddr:         netip.MustParseAddr("10.0.0.42"),
            SubnetMaskBits: 24,
            Router:         netip.MustParseAddr("10.0.0.1"),
        }, true
    }),
}
srv, _ := dhcp.NewLinuxServer(opts)
_ = srv.Run(ctx) // blocks until ctx is cancelled
```

## Repositories

| Repo | What it is |
| --- | --- |
| [`dhcp`](https://github.com/go-net-dhcp/dhcp) | the library — wire codec, `Decide`, the Linux server, and the `prom` metrics adapter |
| [`docs`](https://github.com/go-net-dhcp/docs) | this documentation site (MkDocs Material, versioned with mike) |
| [`go-net-dhcp.github.io`](https://github.com/go-net-dhcp/go-net-dhcp.github.io) | the organization landing page (Hugo) |
| [`brand`](https://github.com/go-net-dhcp/brand) | logo and brand assets |

## Principles

- **Pure Go, `CGO_ENABLED=0`** — trivial cross-compilation, a single static
  binary, no C toolchain.
- **Dependency-free core.** The wire codec is hand-rolled; the Prometheus adapter
  is quarantined in its own sub-package.
- **Stateless & caller-driven.** You own the address plan via a `Source`.
- **Testable to the socket.** The Linux socket path is covered without root via
  injectable syscall seams.
- **100% test coverage** is the target, enforced as a CI gate.

## Where to go next

- [Usage & API](api.md) — `Options`, `Lease`, `Source`, `NewLinuxServer`, `NewStub`.
- [Wire format & decision core](wire-format.md) — `Parse`, `BuildReply`, `Decide`.
- [The Linux server](server.md) — `SO_BINDTODEVICE`, the receive loop, the syscall seams.
- [Metrics](metrics.md) — the `Metrics` hook and the Prometheus adapter.
- [Roadmap](roadmap.md) — what is done and what is deliberately out of scope.

Source lives at
[github.com/go-net-dhcp/dhcp](https://github.com/go-net-dhcp/dhcp).
