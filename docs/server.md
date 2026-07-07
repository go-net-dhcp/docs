# The Linux server

`LinuxServer` is the real UDP/67 DHCPv4 server. It only builds on Linux
(`//go:build linux`); other platforms get a stub that returns a "linux-only"
error from the constructor.

## Binding

The socket is constructed by hand (rather than via `net.ListenUDP`) so the server
can set three socket options before the kernel commits the binding:

- **`SO_REUSEADDR`** — restart without a `TIME_WAIT` bind conflict.
- **`SO_BROADCAST`** — the reply goes to `255.255.255.255:68`.
- **`SO_BINDTODEVICE`** (the `Interface` in `Options`) — only frames arriving on
  that NIC / bridge / VLAN reach the socket, and replies leave the same path.
  This is what makes it safe to serve DHCP on one segment of a multi-homed host.
  It requires `CAP_NET_RAW` (run as root or grant the capability via systemd).

The raw fd is then handed to the `os`/`net` stack (`os.NewFile` +
`net.FilePacketConn`) so the receive loop gets `ReadFromUDP` / `WriteToUDP`
ergonomics and the runtime poller.

## The receive loop

```go
srv, _ := dhcp.NewLinuxServer(opts)
err := srv.Run(ctx)
```

`Run` binds, then loops: `ReadFromUDP` → `Parse` → `Decide` → send. Transient
problems (a malformed packet, an unknown MAC) are logged and the loop continues;
only a socket-fatal error breaks out. Cancelling `ctx` closes the socket, which
unblocks the read and returns `ctx.Err()`.

Replies are always broadcast in this version — it reaches every switch and does
not depend on the client already having ARP'd the server. `SO_BINDTODEVICE`
keeps the broadcast on the right NIC.

Logging goes through `log/slog`; swap the logger with `SetLogger`.

## Testing to the socket — the syscall seam

Reaching 100% coverage of a socket server usually means "skip in CI without
root". This library avoids that: the six raw calls `listen` makes — `socket`,
`setsockopt` (int and string), `bind`, `close`, and fd-adoption — sit behind an
injectable seam. Tests substitute fakes to drive **every** error branch (each
`setsockopt` failure, the bind failure, the fd-adoption failure, the
unexpected-conn-type branch) and the full receive → decide → send loop, all
without `CAP_NET_RAW`. The production seam wires straight to `golang.org/x/sys/unix`
and is itself exercised against a real ephemeral loopback socket.

The result: the coverage gate holds at 100% on the ordinary GitHub `ubuntu`
runner, including the socket code.
