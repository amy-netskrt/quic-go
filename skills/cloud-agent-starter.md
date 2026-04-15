# Cloud Agent Starter Skill: quic-go

Minimal runbook for Cloud agents to get productive quickly in this repo.

## 1) First 5 minutes (setup + login reality check)

### Auth / login
- No product or web login is required for local development in this repository.
- `gh` is useful for reading CI / PR data when needed, but local build and tests do not require additional auth.
- Docker login is only needed when pushing interop images; local interop checks do not require pushing.

### Baseline setup
Run these from repo root:
- `go version` (CI currently runs on Go `1.25.x` and `1.26.x`).
- `go env GOMOD` (sanity-check that you are in the module).
- `go test -v ./integrationtests/tools/...` (quick dependency + toolchain smoke test).

## 2) Runtime toggles ("feature flags" in this codebase)

This project mostly uses environment variables as runtime toggles. There is no remote flag service, so "mocking a flag" means setting env vars inline on test commands.

- `TIMESCALE_FACTOR=3|10|20`: increases timing allowances in tests.
- `QUIC_GO_DISABLE_GSO=true`: disable Generic Segmentation Offload code paths.
- `QUIC_GO_DISABLE_ECN=true`: disable Explicit Congestion Notification code paths.
- `QLOGDIR=/tmp/qlog`: enables qlog file output when tracer / qlog mode is used.
- `QUIC_GO_DISABLE_RECEIVE_BUFFER_WARNING=true`: suppresses receive-buffer warnings during constrained test runs.
- `TESTCASE=<name>`: selects interop behavior in `interop/client` and `interop/server`.

## 3) Workflows by codebase area

### A) Core QUIC library (`/` and `/internal`)
Use when changing transport, stream, congestion, packet handling, or shared internals.

Concrete workflow:
1. Fast local package check:
   - `go test -v -shuffle=on .`
2. Internal package coverage:
   - `go test -v -shuffle=on ./internal/...`
3. If timing-sensitive tests are flaky:
   - `TIMESCALE_FACTOR=10 go test -v -shuffle=on . ./internal/...`

### B) HTTP/3 stack (`/http3`)
Use when changing HTTP/3 transport behavior, datagrams, qpack, or server/client integration.

Concrete workflow:
1. Package tests:
   - `go test -v -shuffle=on ./http3/...`
2. Optional qlog-enabled rerun for debugging:
   - `mkdir -p /tmp/qlog && QLOGDIR=/tmp/qlog go test -v -shuffle=on ./http3/...`
3. Quick executable smoke path:
   - Terminal A: `go run ./example -bind localhost:6121`
   - Terminal B: `go run ./example/client -insecure https://localhost:6121/demo/echo`

### C) Integration suites (`/integrationtests`)
Use when changing handshake, version negotiation, migration, retries, HTTP/3 end-to-end, or behavior that needs networked fixtures.

Concrete workflow (mirrors CI order):
1. Tools:
   - `go test -v -timeout 30s -shuffle=on ./integrationtests/tools/...`
2. Version negotiation:
   - `go test -v -timeout 30s -shuffle=on ./integrationtests/versionnegotiation`
3. Self suite, QUIC v1:
   - `go test -v -timeout 5m -shuffle=on ./integrationtests/self -version=1`
4. Self suite, QUIC v2:
   - `go test -v -timeout 5m -shuffle=on ./integrationtests/self -version=2`
5. Toggle-based scenarios (feature-flag style):
   - `QUIC_GO_DISABLE_GSO=true go test -v -timeout 5m -shuffle=on ./integrationtests/self -version=1`
   - `QUIC_GO_DISABLE_ECN=true go test -v -timeout 5m -shuffle=on ./integrationtests/self -version=1`

### D) Examples and "start the app" smoke tests (`/example`)
Use to quickly prove an end-to-end run without full integration suite cost.

Concrete workflow:
1. Single-command echo smoke test:
   - `go run ./example/echo`
2. Demo HTTP/3 server + client:
   - Terminal A: `go run ./example -bind localhost:6121`
   - Terminal B: `go run ./example/client -insecure https://localhost:6121/1024`

### E) Interop harness (`/interop`)
Use when working on interoperability behavior or endpoint behavior expected by external interop runners.

Concrete workflow:
1. Local package checks:
   - `go test -v ./interop/...`
2. Understand endpoint script contract before debugging:
   - `ROLE`, `TESTCASE`, `CLIENT_PARAMS`, and mounted dirs (for `/certs`, `/www`, `/logs`) drive behavior in `interop/run_endpoint.sh`.
3. Optional local image build smoke test:
   - `docker build -f interop/Dockerfile -t quic-go-interop:local .`

## 4) Common troubleshooting

- If timing-related failures appear, retry with a higher `TIMESCALE_FACTOR`.
- If you need protocol traces, create and set `QLOGDIR`, then rerun with qlog-enabled paths.
- If Linux networking behavior is suspect, run one pass each with `QUIC_GO_DISABLE_GSO=true` and `QUIC_GO_DISABLE_ECN=true` to isolate kernel-offload-related effects.

## 5) Keeping this skill current

When you discover a new runbook trick (from CI failures, flaky test mitigation, or reproducible local debug steps), update this file in the same PR if possible:
1. Add the note under the relevant area section (A-E), not in a random appendix.
2. Include one concrete command and one sentence saying when to use it.
3. Mark caveats clearly (platform-specific, slow, requires Docker, etc.).
4. Remove or rewrite stale steps when CI workflows or flags change.
