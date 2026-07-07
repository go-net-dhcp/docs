# Usage & API

Install:

```sh
go get github.com/go-net-dhcp/dhcp
```

## Running the Linux server

```go
package main

import (
	"context"
	"net/netip"

	"github.com/go-net-dhcp/dhcp"
)

func main() {
	opts := dhcp.Options{
		Interface: "br0",
		ServerIP:  netip.MustParseAddr("10.0.0.1"),
		Source: dhcp.SourceFn(func(mac string) (dhcp.Lease, bool) {
			if mac != "52:54:00:00:00:01" {
				return dhcp.Lease{}, false // unknown MAC → silently dropped
			}
			return dhcp.Lease{
				Yiaddr:         netip.MustParseAddr("10.0.0.42"),
				SubnetMaskBits: 24,
				Router:         netip.MustParseAddr("10.0.0.1"),
				DNSServers:     []netip.Addr{netip.MustParseAddr("9.9.9.9")},
			}, true
		}),
	}

	srv, err := dhcp.NewLinuxServer(opts)
	if err != nil {
		panic(err)
	}
	// Run blocks until ctx is cancelled (or the socket errors out).
	_ = srv.Run(context.Background())
}
```

`NewLinuxServer` binds only on Linux. On other platforms it validates `Options`
and then returns a "linux-only" error, so cross-platform code compiles unchanged;
use [`NewStub`](#the-stubserver) where you only need the `Source` pipeline.

## `Options`

```go
type Options struct {
	Interface string      // kernel interface to bind (SO_BINDTODEVICE); required
	ServerIP  netip.Addr  // announced as option 54 (server identifier); required IPv4
	Source    Source      // resolves a client MAC → Lease; required
	Metrics   Metrics     // optional per-packet telemetry sink (nil = discard)
}

func (o Options) Validate() error
```

## `Lease`

The per-MAC answer your `Source` returns. Only `Yiaddr` and `SubnetMaskBits` are
required; the rest are optional and skipped when zero.

```go
type Lease struct {
	Yiaddr         netip.Addr    // handed to the client
	SubnetMaskBits int           // prefix length (1–32) → option 1
	Router         netip.Addr    // option 3 (optional)
	DNSServers     []netip.Addr  // option 6 (optional)
	Domain         string        // option 15 (optional)
	LeaseTime      time.Duration // option 51 (0 = 1h default)
}

func (l Lease) Validate() error
```

`Validate` is called for you inside `Decide`; a lease that fails it is dropped
(and, on the real server, counted as `drop_decide_err`).

## `Source`

```go
type Source interface {
	Resolve(mac string) (Lease, bool)
}

// SourceFn adapts a plain function.
type SourceFn func(mac string) (Lease, bool)
```

`Resolve` returns `(Lease{}, false)` to decline — an unknown MAC is dropped
silently, with **no NAK**. The MAC is the lowercase colon-separated form, e.g.
`"52:54:00:00:00:01"`.

## The `StubServer`

Usable on every platform, no socket:

```go
s, _ := dhcp.NewStub(opts)
lease, issued := s.SimulateRequest("52:54:00:11:22:33")
for _, h := range s.Hits() { /* h.MAC, h.Lease, h.Issued */ }
```

`Run(ctx)` blocks until `ctx` is cancelled, matching the real server's lifecycle,
so the same start/stop plumbing works in tests.

## Lower-level primitives

If you want to drive the protocol yourself:

```go
func Parse(buf []byte) (*Packet, error)
func BuildReply(req *Packet, msgType byte, serverID netip.Addr, lease Lease) ([]byte, error)
func Decide(pkt *Packet, opts Options) (Decision, error)
```

See [Wire format & decision core](wire-format.md).
