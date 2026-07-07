# Contributing

Contributions are welcome. The bar is a green CI run, which means:

- **`gofmt` + `go vet` clean.**
- **100% test coverage**, error branches included. The gate runs on every OS:

    ```sh
    COVERPKG=$(go list ./... | paste -sd, -)
    go test -race -coverpkg="$COVERPKG" -coverprofile=cover.out ./...
    go tool cover -func=cover.out | tail -1   # must be 100.0%
    ```

- **Green on all six 64-bit Go targets** — amd64, arm64, riscv64, loong64,
  ppc64le, s390x. The non-native arches run under qemu-user in CI.

## Testing the socket paths

The Linux server's socket calls sit behind an injectable seam, so new error
branches can and should be covered without root — inject a fake that returns the
error and assert the outcome. See [The Linux server](server.md#testing-to-the-socket--the-syscall-seam).
Tests that bind a real socket must use an **ephemeral loopback** port so they run
unprivileged in CI and under qemu.

## Keeping the core dependency-free

The `github.com/go-net-dhcp/dhcp` package must not gain a third-party import.
Anything that needs an external dependency (like the Prometheus adapter) goes in
a **sub-package**, wired to the core only through the small `Metrics` interface.

## License

By contributing you agree your work is licensed under BSD-3-Clause, copyright the
go-net-dhcp/dhcp authors.
