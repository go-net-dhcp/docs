# Wire format & decision core

The protocol layer is **build-tag-free** — it compiles and is tested on every
platform, because none of it touches a socket.

## `Parse`

```go
func Parse(buf []byte) (*Packet, error)
```

Decodes a DHCPv4 datagram: the fixed 236-byte BOOTP header, the 4-byte magic
cookie (`99 130 83 99`, RFC 2131 §3), then the TLV option block. The parser is
**strict on framing** — it errors on a short buffer, a bad cookie, or an option
whose length runs past the end of the packet — but **tolerant of unknown option
codes**, which land in `Packet.Options` as raw bytes. Pad (0) and End (255) are
handled per spec.

```go
type Packet struct {
	Op, Htype, Hlen, Hops byte
	Xid                   uint32
	Secs, Flags           uint16
	Ciaddr, Yiaddr, Siaddr, Giaddr [4]byte
	Chaddr [16]byte
	Sname  [64]byte
	File   [128]byte
	Options map[byte][]byte
}

func (p *Packet) MessageType() byte     // option 53, or 0
func (p *Packet) RequestedIP() netip.Addr // option 50, or the zero Addr
func (p *Packet) MACString() string       // lowercase colon MAC of the first Hlen bytes
```

## `BuildReply`

```go
func BuildReply(req *Packet, msgType byte, serverID netip.Addr, lease Lease) ([]byte, error)
```

Assembles an OFFER / ACK / NAK from the request plus the server's answer. It
echoes the client's `Xid`, copies the `Chaddr`, writes the magic cookie, and
emits options in a stable order (so packet captures diff cleanly):

- **53** message type, **54** server identifier — always;
- for OFFER / ACK: **1** subnet mask (derived from `SubnetMaskBits`), **3** router
  (when set), **6** DNS (when non-empty), **15** domain (when non-empty), **51**
  lease time (defaulting to 1h).

A NAK carries no `yiaddr` and none of the per-client options. `serverID` must be
a valid IPv4 address.

## `Decide` — the state machine

```go
func Decide(pkt *Packet, opts Options) (Decision, error)

type Decision struct {
	Reply   []byte // wire bytes to send; nil = drop silently
	MsgType byte   // OFFER / ACK / NAK carried in Reply (0 when dropped)
	MAC     string // parsed client MAC, for logging
}
```

The rules, in order:

| Inbound | Result |
| --- | --- |
| not a `BOOTREQUEST` (e.g. our own reply echoed back) | drop |
| message type ≠ DISCOVER/REQUEST (DECLINE/RELEASE/INFORM/…) | drop |
| `Source.Resolve(mac)` returns `false` | drop (unknown MAC, no NAK) |
| resolved `Lease` fails `Validate` | error (Source bug) |
| **DISCOVER** | **OFFER** |
| **REQUEST** with option-50 ≠ the resolved `Yiaddr` | **NAK** |
| **REQUEST** otherwise | **ACK** |

Because `Decide` is pure and socket-free, the exact code path the Linux receive
loop runs is unit-tested on every platform — the OS-specific layer only shuttles
bytes in and out.
