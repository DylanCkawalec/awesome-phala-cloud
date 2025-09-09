# dStack Remote Attestation Dashboard (Phala Cloud)

[![Deploy to Phala Cloud](https://cloud.phala.network/deploy-button.svg)](https://cloud.phala.network/templates/ra-dashboard)

An interactive, production-grade remote attestation dashboard for Phala Network TEEs. It visualizes quote generation, verification, and secure session setup, and includes a full API testing suite. Built from the dstack-visualizer and the Remote Attestation Template.

Repo: https://github.com/DylanCkawalec/dstack-visualizer

---

Why this matters
- Hardware-backed trust: Intel TDX + dStack SDK provides verifiable compute, identity, and networking.
- Zero Trust by design: “Never trust, always verify” — continuously verify the runtime, not just at startup.
- Developer experience: Next.js dashboard + Python FastAPI + optional Bun server for benchmarking.

---

Features
- TEE Attestation: Generate quotes with event logs and measurements (RTMRs) and verify them
- Live Security: Security status, capabilities, key/quote functions, and measurements
- API Test Suite: One-click test against all implemented endpoints with latency metrics
- Explorer Integration: Submit quotes and verify on external explorers (t16z)
- Blog-Style Guide: Understand how remote attestation works end to end

Architecture

  +-----------------------------------------------------------+
  |                Remote Attestation Dashboard               |
  |  (Next.js 15 UI + Python FastAPI + optional Bun server)  |
  +-------------------------+---------------------------------+
            |               |                     |
            |               |                     |
     Frontend (3000)    Python API (8000)     Bun (8001, optional)
            \               |                     /
             \              v                    /
              +--------- dStack SDK (0.5.x) -----+
                             |
                       Phala TEE (Intel TDX)

Live endpoints (example)
- Dashboard: https://<appId>-3000.<gateway-domain>/
- API: https://<appId>-8000.<gateway-domain>/
- API Docs (FastAPI): https://<appId>-8000.<gateway-domain>/docs

Quick start

Prerequisites
- Docker 20.10+
- phala CLI (npm i -g @phala/cli), login: phala auth login
- A Phala API key and optionally a dStack API key

Deploy with Phala CLI
1) Bootstrap env

  cp .env.example .env
  # Fill in PHALA_API_KEY (and optionally DSTACK_API_KEY, endpoints)

2) Create CVM

  phala cvms create \
    --name ra-dashboard \
    --compose ./docker-compose.yml \
    --env-file ./.env \
    --vcpu 2 \
    --memory 2048 \
    --disk-size 20

3) Open URLs from the dashboard. If base URL 80 doesn’t respond, use port-qualified form: https://<appId>-3000.… or -8000.…

Configuration

Environment variables (.env)
Required
- PHALA_API_KEY=your-phala-api-key

Recommended/optional
- DSTACK_API_KEY=your-dstack-api-key
- DSTACK_ENDPOINT=https://api.dstack.network
- PHALA_ENDPOINT=https://api.phala.network (or your gateway if applicable)
- APP_NAME=remote-attestation-dashboard
- APP_VERSION=1.0.0
- DEVELOPER_NAME=Your Name
- ORGANIZATION=Your Org
- NODE_ENV=production
- ATTESTATION_SEED=phala-attestation-seed
- ATTESTATION_SALT=phala-attestation-salt
- PHALA_CLUSTER_ID=…
- PHALA_CONTRACT_ID=…

Ports
- UI: 3000 (exposed as 80 or 3000 externally depending on mapping)
- API: 8000
- Bun: 8001 (optional)

Docker Compose (Phala-ready)
Includes two services (UI and API) and mounts TEE sockets when present.

Remote Attestation — the mental model
1) Identity and measurements
   - The TEE exposes measurements (RTMR0–3, MRTD) that summarize the boot/runtime state.
   - A quote packages measurements + runtime info + event log into a signed structure.
2) Generating attestation
   - Frontend asks the API to generate a quote (optionally with user data and nonce).
   - The API tries the real dStack SDK first; if unavailable, it falls back to unix-socket protocol; if that fails, it returns a safe mock with environment hints.
3) Verifying attestation
   - Verifier compares expected data, checks measurements/TCB info, and validates the signature chain.
   - You can cross-check quotes against an external explorer like t16z.
4) Trusting the dashboard
   - The UI never blindly trusts the API; it displays evidence (quotes, measurements, capabilities) and exposes a testing suite.
   - All secrets are injected at runtime; private keys should never be baked into images.

How the template works

Python FastAPI (api/main.py)
- dStack integration precedence:
  1. AsyncDstackClient (preferred) — real SDK usage for info(), get_quote(), etc.
  2. Unix socket protocol (/var/run/dstack.sock, /var/run/tappd.sock) — direct calls
  3. Mock mode — still returns useful environment info so you can introspect the TEE
- Implemented endpoints:
  - GET /api/health — health + TEE readiness
  - POST /api/attestation/generate — returns attestation with quote/event log/RTMRs if available
  - POST /api/attestation/verify — verifies attestation (delegates to SDK/socket)
  - GET /api/tee/info — returns TEE info (TDX, versions, device_id)
  - GET /api/tee/measurements — MRTD and RTMRs
  - POST /api/tee/execute — invoke a named function in the TEE
  - GET /api/security/status — overall status + sockets availability
  - GET /api/test/all — runs a mini test suite with timings

Node/Bun (bun-server/index.ts)
- Optional high-performance server with helpful routes:
  - GET / — service status
  - GET /api/benchmark — measures latency across several SDK-like operations
  - POST /api/attestation/batch — generateAttestation N times and summarize
  - GET /api/tee/monitor — fetch info + measurements + status together
- Uses PHALA_API_KEY or DSTACK_API_KEY if present

Frontend (src/*)
- app/page.tsx — main layout with tabs: Attestation, TEE Info, Security, API Test; right-side visualization panel
- components/AttestationDashboard.tsx — triggers backend attestation, verification, and TEE info calls
- components/APITester.tsx — runs an end-to-end suite against all available API endpoints, aggregates latency and success/failure
- components/SecurityDashboard.tsx — security-oriented functions (status, certificate/key paths, quotes)
- components/AttestationExplorer.tsx — submit quotes to explorer and fetch node info; links to t16z proof explorer
- src/attestation-service.ts — blog-worthy reference implementation for generating/verifying local attestation payloads, hashing code/config, and obtaining Phala-backed quotes

Using t16z explorer (https://proof.t16z.com/)
- Paste or upload your TEE quote to cross-check measurements, TCB status, and signature chain.
- Use alongside /api/tee/measurements and /api/attestation/generate outputs to validate end-to-end.

dStack SDK usage notes
- Python: AsyncDstackClient provides info(), get_quote(data), and helpers such as replay_rtmrs() from the quote.
- Node/Bun: a thin client abstraction can invoke generateAttestation(), getTEEInfo(), getMeasurements(), and getSecurityStatus().
- Both environments prefer sockets inside the CVM when SDK bindings aren’t available, ensuring you can still introspect.

Security and operations
- Secrets via env: Always pass credentials through the CLI’s --env-file; never commit keys.
- Ports and gateways: Externally, Phala maps ports as subdomains (<appId>-PORT.gateway). Use /docs on the API to explore.
- Resource sizing: Default 2 vCPU / 2GB / 20GB is sufficient; bump for heavier workloads.

FAQ
- The API shows mock=true — Your environment may not have sockets available or SDK bindings installed. The API still returns helpful data to verify the environment.
- Can I run both UI and API in one image? You can, but we recommend separate services for clarity and scaling. The start.sh in the template demonstrates a dev-mode tri-service launcher.

License
MIT

