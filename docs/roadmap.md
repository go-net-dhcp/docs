# Roadmap

## Done

- **RFC 2131 / 2132 wire codec** — `Parse` + `BuildReply` (OFFER / ACK / NAK).
- **Decision core** — the stateless `Decide` state machine, build-tag-free.
- **Linux UDP/67 server** — `SO_BINDTODEVICE` bind, receive loop, broadcast reply.
- **Cross-platform surface** — non-Linux build stub + socket-free `StubServer`.
- **Optional metrics** — the `Metrics` hook + a Prometheus adapter in `prom`.
- **100% coverage** — error branches included, socket paths via syscall seams
  (no root), across the six 64-bit Go targets.

## Deliberately out of scope (for now)

- **Unicast replies.** The server always broadcasts. The spec allows a unicast
  reply to `chaddr` when the client's BROADCAST flag is clear; that is a
  candidate addition once there is a concrete client that needs it.
- **Lease persistence / an address pool.** The library is stateless by design —
  you own the address plan through `Source`. Dynamic allocation from a pool, with
  conflict detection and a lease database, is a layer that belongs above this one.
- **DHCPv6.** This is a DHCPv4 server. IPv6 leasing (RFC 8415) is a separate
  protocol and out of scope for this module.
- **Relay / `giaddr` handling.** The current target is a single directly-attached
  segment; relayed requests are parsed but not specially routed.

## Design guarantees that will not change

- **Pure Go, zero cgo** in the core.
- **No third-party dependency** in the core module — only the opt-in `prom`
  sub-package imports Prometheus.
- **The decision core stays portable** and socket-free, so it is testable
  everywhere.
