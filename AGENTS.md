# AGENTS.md

## Cursor Cloud specific instructions

### Project overview

quic-go is a pure Go implementation of the QUIC protocol (RFC 9000) with HTTP/3 support. It is a **library**, not a standalone application. There are no external service dependencies (databases, caches, etc.) — all tests are self-contained.

### Go version

The project requires **Go 1.25+** (see `go.mod`). The VM snapshot has Go 1.25.9 installed at `/usr/local/go/bin`. If Go is not on `PATH`, run `export PATH=/usr/local/go/bin:$PATH`.

### Key commands

| Task | Command |
|---|---|
| Download deps | `go mod download` |
| Build | `go build ./...` |
| Vet/lint | `go vet ./...` |
| Unit tests | `TIMESCALE_FACTOR=10 go test -shuffle=on ./...` (includes integration tests) |
| Unit tests only | Remove `integrationtests/` first (CI does `git rm -r --cached integrationtests`), then `go test ./...` |
| Integration (self) | `go test -timeout 5m -shuffle=on ./integrationtests/self` |
| Integration (version negotiation) | `go test -timeout 30s -shuffle=on ./integrationtests/versionnegotiation` |
| Echo demo | `go run ./example/echo/` |

### Non-obvious notes

- `TIMESCALE_FACTOR` environment variable adjusts timing tolerances in tests. CI uses `10` for unit tests and `3` for integration tests. Set it higher (e.g. `20`) if running with the race detector.
- The UDP buffer size warning (`failed to sufficiently increase receive buffer size`) is benign in dev/test environments and does not affect functionality.
- Git hooks are available in `.githooks/` — install with `git config core.hooksPath .githooks` (optional, contains a pre-commit hook).
- The `integrationtests/gomodvendor/` subdirectory is a separate Go module used to test vendoring; it replaces quic-go with the local checkout.
