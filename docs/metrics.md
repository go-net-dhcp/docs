# Metrics

Metrics are an **optional hook**. The core library depends on nothing
third-party; if you want Prometheus, you opt in via a separate sub-package.

## The `Metrics` interface

```go
type Metrics interface {
	RecordPacket(outcome string)         // one call per inbound packet
	RecordHandleDuration(seconds float64) // parse+decide+send latency
}
```

Pass any implementation as `Options.Metrics`. A `nil` value is replaced by a
no-op sink, so metrics cost nothing until you ask for them. The `outcome` is one
of the exported constants:

| Constant | Value | Meaning |
| --- | --- | --- |
| `OutcomeOffer` | `offer` | sent an OFFER |
| `OutcomeAck` | `ack` | sent an ACK |
| `OutcomeNak` | `nak` | sent a NAK |
| `OutcomeDropParseErr` | `drop_parse_err` | malformed wire packet |
| `OutcomeDropUnknownMAC` | `drop_unknown_mac` | `Source.Resolve` returned false |
| `OutcomeDropDecideErr` | `drop_decide_err` | lease validation / build failed |
| `OutcomeDropUnsupported` | `drop_unsupported` | message type we don't answer |
| `OutcomeSendErr` | `send_err` | wire-side write failed |

## The Prometheus adapter

```go
import (
	"github.com/go-net-dhcp/dhcp"
	dhcpprom "github.com/go-net-dhcp/dhcp/prom"
	"github.com/prometheus/client_golang/prometheus"
)

m, err := dhcpprom.New(prometheus.DefaultRegisterer)
if err != nil {
	// a collector with the same name is already registered
}
opts.Metrics = m
```

`prom.New` registers two collectors (bounded cardinality — no per-MAC labels):

- `dhcpv4_packets_total{outcome}` — a counter, labelled by the outcomes above;
- `dhcpv4_handle_duration_seconds` — a histogram of per-packet latency.

Because this lives in `github.com/go-net-dhcp/dhcp/prom`, importing the core
`dhcp` package never pulls in the Prometheus client. Bring your own `Metrics`
implementation just as easily — the interface is two methods.
