# bubbaloop-industrial-edge

> **GSoC 2025 — kornia / Bubbaloop**  
> A real-world, production-grade Bubbaloop application running on live industrial hardware — integrating a Hikrobot smart camera, Siemens S7-1200 PLC, and a full observability dashboard contributed upstream to Bubbaloop.

---

## What this project is

This repository is the working implementation of my Google Summer of Code 2025 proposal for the [kornia](https://github.com/kornia) organization under the project idea **"Build a real-world Bubbaloop application on a robot or IoT device."**

The application is an **Industrial Conveyor Belt Automation System** for warehouse inward goods processing. It is not a simulation — the hardware is real, running, and already has a working Python prototype. This GSoC project ports that system to a high-performance, event-driven Bubbaloop pipeline written in Rust, and contributes node observability tooling back to the Bubbaloop runtime.

---

## Hardware I own and am actively using

| Device | Model | Protocol | Role |
|--------|-------|----------|------|
| Smart code reader | Hikrobot MV-ID6200M-00C-NNG | TCP socket + FTP | Captures 20MP images, decodes QR codes at 60/sec |
| PLC | Siemens S7-1200 | Modbus TCP (FC6, register 40001) | Controls physical conveyor belt motor |
| Conveyor belt | Industrial grade | — | Physical goods transport |
| Edge host | Ubuntu PC (192.168.0.xx) | — | Runs all Bubbaloop nodes |

The Modbus TCP control logic, FTP image receiver, QR TCP stream parser, and full React dashboard are **already implemented and working** in Python/React. The GSoC work is the architectural migration to Bubbaloop + the upstream observability contribution.

---

## Why Bubbaloop — the problem with the current system

The existing Python implementation runs four separate processes that communicate via HTTP polling:

```
React UI  →  polls Flask every 200ms  →  "any new QR codes?"
tcp_client  →  polls Flask every 200ms  →  "any manual commands?"
```

This creates real problems in a time-sensitive industrial environment:

- **200ms polling latency** between a QR mismatch and the PLC stop command — the belt moves further than it should
- **4 separate processes** with no unified health monitoring — any silent crash goes undetected
- **No observability** — operators have zero visibility into which service is alive, message throughput, or timing

The Bubbaloop migration replaces HTTP polling with **Zenoh pub/sub** (microsecond latency, zero-copy), unifies processes into standard nodes with health endpoints, and enables the observability dashboard this project contributes upstream.

---

## Proposed architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Hardware layer                               │
│                                                                 │
│  Photoelectric  →  Hikrobot camera      Siemens S7-1200 PLC    │
│  sensor            192.168.0.xx         192.168.0.xx:502        │
│                    TCP:2002 + FTP:2005  Modbus TCP              │
└──────────────────────────┬──────────────────┬───────────────────┘
                           │                  │
                    ┌──────▼──────────────────▼──────┐
                    │     Zenoh pub/sub message bus   │
                    │  (zero-copy, microsecond latency)│
                    └──────┬──────────┬───────────────┘
                           │          │
           ┌───────────────▼──┐  ┌────▼──────────────┐
           │  Camera node     │  │  PLC node          │
           │  hikrobot-trigger│  │  siemens-s7-1200   │
           │  (Rust)          │  │  (Rust)            │
           └───────┬──────────┘  └────────────────────┘
                   │                      ▲
           ┌───────▼──────────────────────┤
           │  Validation agent            │
           │  QR count vs stack_size      │
           │  publishes STOP on mismatch ─┘
           └───────┬──────────────────────┘
                   │
           ┌───────▼──────────────────────────────────┐
           │  React observability dashboard            │
           │  (upstream contribution to Bubbaloop)    │
           │  Node DAG · live QR log · belt controls  │
           └──────────────────────────────────────────┘
```

### Zenoh topic map

| Publisher | Topic | Subscriber(s) |
|-----------|-------|---------------|
| Camera node | `bubbaloop/local/camera/qr-data` | Validation agent |
| Validation agent | `bubbaloop/local/plc/command` | PLC node |
| Validation agent | `bubbaloop/local/agent/scan-result` | Dashboard |
| PLC node | `bubbaloop/local/plc/status` | Dashboard |

---

## Node specifications

Every Bubbaloop node exposes the standard five HTTP endpoints alongside its core logic. All three nodes below follow this contract.

### Node 1 — Camera node (`hikrobot-trigger`)

**Responsibility:** Connect to the Hikrobot QR scanner TCP stream, receive FTP image uploads, batch QR codes, and publish structured data to Zenoh.

**Inputs:**
- Raw TCP stream on `192.168.0.xx:2002` — semicolon-delimited QR strings
- FTP push from camera on `0.0.0.0:2005` — JPEG image files

**Zenoh output — `camera/qr-data`:**
```json
{
  "codes": ["QR-BOX-001", "QR-BOX-002", "QR-BOX-003"],
  "count": 3,
  "timestamp": 1711360000,
  "image_path": "/data/hik/capture_001.jpg",
  "trigger_id": "scan-0042",
  "scanner_ip": "192.168.0.78",
  "duration_ms": 47
}
```

**Rust crates:** `tokio` (async runtime), `tokio::net::TcpStream`, `libunftp` or `unftp-sbe-fs` (embedded FTP), `zenoh`

**Standard endpoints:**
```
GET  /health    →  { "status": "ok", "last_scan": "2025-04-01T10:22:01Z", "codes_today": 1240 }
GET  /manifest  →  { "name": "hikrobot-trigger", "version": "0.1.0", "type": "camera" }
GET  /schema    →  Protobuf definition of camera/qr-data message
GET  /config    →  { "scanner_ip": "192.168.0.xx", "scanner_port": 2002, "ftp_port": 2005 }
POST /command   →  { "command": "start_scan" | "stop_scan" | "configure" }
```

---

### Node 2 — Validation agent

**Responsibility:** Subscribe to `camera/qr-data`, compare scanned count against operator-set `stack_size`, publish STOP command on mismatch, post to cloud webhook on success. This is a direct port of the validation logic in `tcp_client.py`.

**Internal state:**
- `expected_stack_size: AtomicU32` — updated by operator via `/command` or MCP tool
- `session_seen_set: HashSet<String>` — duplicate detection across the full session

**Decision logic:**
```
ON camera/qr-data received:
  IF any code in session_seen_set → publish STOP (reason: duplicate_qr)
  ELSE IF codes.count == expected_stack_size:
    → add codes to session accumulator
    → POST each code to cloud webhook (leader.salaryslip.co)
    → publish SUCCESS to agent/scan-result
  ELSE:
    → publish STOP command to plc/command
    → publish MISMATCH to agent/scan-result (missing/extra counts)
```

**Zenoh outputs:**
```json
// plc/command (on mismatch)
{ "command": "stop", "reason": "qr_mismatch", "expected": 4, "received": 3 }

// agent/scan-result (on success)
{ "status": "ok", "codes": [...], "trigger_id": "scan-0042", "matched": true }
```

**Standard endpoints:**
```
GET  /health    →  { "status": "ok", "stack_size": 4, "session_scans": 12 }
GET  /config    →  { "stack_size": 4, "webhook_url": "..." }
POST /command   →  { "command": "set_stack_size", "value": 4 }
```

---

### Node 3 — PLC node (`siemens-s7-1200`)

**Responsibility:** Subscribe to `plc/command`, maintain a persistent Modbus TCP socket to the Siemens S7-1200, write Function Code 6 to register 40001 on command. The Modbus packet structure is identical to the working Python implementation.

**Modbus packet (unchanged from existing system):**
```rust
// Function Code 6: Write Single Register
// Register 40001, offset 0
// value 0 = START belt, value 1 = STOP belt
let request = WriteSingleRegister { address: 0, value };
ctx.write_single_register(address, value).await?;
```

**Socket management:**
- Persistent TCP connection to `192.168.0.xx:xx`
- `plc_lock: Mutex` serializes all writes (thread-safe)
- Auto-reconnect on drop with 2 retry attempts, exponential backoff

**Rust crates:** `tokio-modbus`, `zenoh`, `tokio`

**Zenoh output — `plc/status` (after each write):**
```json
{
  "belt_state": "stopped",
  "last_command": "stop",
  "success": true,
  "timestamp": 1711360001,
  "trigger_id": "scan-0042"
}
```

**Standard endpoints:**
```
GET  /health    →  { "status": "ok", "socket": "connected", "belt_state": "running" }
GET  /config    →  { "plc_ip": "192.168.0.xx", "plc_port": xxx, "register": 40001 }
POST /command   →  { "command": "start" | "stop" | "emergency_stop" }
```

---

## Upstream contribution — observability dashboard

The most impactful contribution of this project to the Bubbaloop community is the **web-based node observability dashboard** — currently Bubbaloop has CLI-only visibility into node state.

This dashboard will be built in React and contributed directly to the Bubbaloop repository. It consumes the standard node endpoints (`/health`, `/manifest`, `/config`) uniformly across all nodes.

**Dashboard panels:**

| Panel | Data source | What it shows |
|-------|-------------|---------------|
| Node DAG view | `/manifest` from all nodes | Live graph of nodes + Zenoh topic connections |
| Node health grid | `/health` (1s poll) | Green/red status, last seen, message count |
| Live QR log | `agent/scan-result` Zenoh sub | Real-time scan events, PASS/MISMATCH badges |
| Belt controls | `POST /command` on PLC node | Start/Stop with live belt status from `plc/status` |
| Camera feed | `/latest-image` (FastAPI port 8000) | Last captured image from Hikrobot camera |
| Throughput chart | Aggregated from node `/health` | Messages/sec per Zenoh topic over time |

The dashboard is intentionally framework-agnostic in its data layer — it talks only to the standard node contract endpoints, meaning any future Bubbaloop node automatically appears in it.

---

## End-to-end data flow

```
1. Worker places boxes on conveyor belt

2. Photoelectric sensor triggers Hikrobot camera hardware capture

3. Camera node (Rust):
   - Reads QR strings from TCP stream (192.168.0.78:2002)
   - Receives JPEG image via embedded FTP server (port 2005)
   - Publishes to Zenoh: bubbaloop/local/camera/qr-data

4. Validation agent (Rust) wakes on Zenoh message:
   - Checks for duplicates against session_seen_set
   - Compares codes.count vs expected_stack_size

5a. COUNT MATCHES:
   - Adds QRs to session accumulator
   - POSTs each code to cloud webhook (leader.salaryslip.co)
   - Publishes SUCCESS to agent/scan-result
   - Belt continues running — PLC untouched

5b. COUNT MISMATCH:
   - Publishes STOP to plc/command immediately
   - Publishes MISMATCH event to agent/scan-result

6. PLC node (Rust) on STOP command:
   - Acquires plc_lock
   - Writes Modbus FC6 value=1 to register 40001
   - Belt stops physically within milliseconds
   - Publishes plc/status (belt_state: stopped)

7. Dashboard:
   - Receives scan-result event → shows error popup or green tick
   - Receives plc/status → updates belt status badge
   - Operator dismisses error, fixes stack, clicks Rescan

8. Operator syncs accumulated QR codes:
   - Clicks Sync in dashboard
   - POST to Lux WMS API: wmsbeta.luxkutumb.info/api/sap/inward
   - QR queue cleared, ready for next batch
```

---

## Comparing current Python system vs Bubbaloop target

| Dimension | Current Python | Bubbaloop (target) |
|-----------|---------------|---------------------|
| Inter-process comms | HTTP polling every 200ms | Zenoh pub/sub, ~10µs latency |
| PLC stop latency | Up to 200ms after mismatch | <5ms (event-driven) |
| Process count | 4 independent Python processes | 3 Rust nodes + dashboard |
| Health monitoring | None — silent crashes go undetected | `/health` endpoint on every node |
| Observability | Zero | Full web dashboard (this project) |
| Memory footprint | ~180MB (4× Python runtimes) | ~15MB (Rust binary) |
| Thread safety | Python GIL + manual locking | Rust borrow checker at compile time |
| AI agent control | Not possible | MCP tools on every node |

---

## Project timeline — 350 hours over 12 weeks

### Community bonding (pre-coding)
- Deep dive into Bubbaloop's node architecture and `ARCHITECTURE.md`
- Build Bubbaloop from source, run quickstart examples
- Discuss node schema and API contracts with mentors
- Begin Rust fundamentals — ownership, async, tokio basics
- Set up local development environment with all hardware connected

---

### Phase 1 — Rust foundation + PLC node (Weeks 1–3) · ~90 hours

**Goal:** First working Rust node that physically controls the conveyor belt.

- Week 1: Learn `tokio` async runtime, understand Zenoh pub/sub API, study `tokio-modbus` crate documentation
- Week 2: Implement PLC node — persistent Modbus TCP socket, START/STOP writes, `plc_lock` Mutex, auto-reconnect logic
- Week 3: Implement all five node endpoints (`/health`, `/manifest`, `/schema`, `/config`, `/command`), wire Zenoh subscriber for `plc/command`, test with live S7-1200

**Milestone:** Push button in terminal → Modbus packet fires → physical belt starts/stops. All five endpoints return correct JSON.

---

### Phase 2 — Camera node (Weeks 4–6) · ~90 hours

**Goal:** Camera node reads live QR data and publishes structured Zenoh messages.

- Week 4: Implement async TCP stream reader for Hikrobot port 2002, parse semicolon-delimited QR strings, implement `extract_qr_value()` URL stripping in Rust
- Week 5: Embed FTP server (`libunftp`) inside the node process, save incoming images to disk, wire image path into the Zenoh payload
- Week 6: Integrate both streams into single structured `camera/qr-data` publish, test with live camera, implement five standard endpoints

**Milestone:** Place a box on the conveyor → Zenoh subscriber on laptop receives `camera/qr-data` message with real QR codes and image path within 100ms.

---

### Phase 3 — Validation agent + end-to-end integration (Weeks 7–9) · ~80 hours

**Goal:** Complete the Zenoh pipeline — camera → agent → PLC — tested on real hardware end-to-end.

- Week 7: Port validation logic from `tcp_client.py` to Rust agent, implement `session_seen_set` (HashSet), `expected_stack_size` (AtomicU32), duplicate detection
- Week 8: Implement cloud webhook posting (`reqwest` HTTP client), wire MCP tool for `set_stack_size`, implement agent's five endpoints
- Week 9: Full end-to-end hardware test — place correct stack → SUCCESS flow, place wrong count → STOP flow, duplicate → STOP flow. Fix edge cases.

**Milestone:** Complete scan cycle works on real hardware without any Python processes running.

---

### Phase 4 — Observability dashboard (Weeks 10–11) · ~60 hours

**Goal:** Web dashboard contributed upstream to Bubbaloop — usable by any Bubbaloop node, not just this project.

- Week 10: Node DAG visualisation (React + D3/SVG), node health grid polling `/health` from all nodes, live belt status from `plc/status` Zenoh subscription via WebSocket bridge
- Week 11: Live QR log from `agent/scan-result`, throughput metrics chart, camera feed panel, start/stop controls, error popup on mismatch event. PR to Bubbaloop repo.

**Milestone:** Dashboard PR submitted to kornia/bubbaloop. Shows any Bubbaloop node's health without configuration.

---

### Phase 5 — Benchmarking, documentation, demo (Week 12) · ~30 hours

- Performance benchmarks: latency from QR scan to PLC stop command (Python vs Bubbaloop), CPU and memory comparison, message throughput under sustained load (60 codes/sec)
- Comprehensive README and node documentation
- Video demo: hardware running, dashboard live, mismatch → belt stop in real time
- Final PR cleanup and mentor review

---

## Repository structure (planned)

```
bubbaloop-industrial-edge/
├── nodes/
│   ├── hikrobot-trigger/     # Camera node (Rust)
│   │   ├── src/
│   │   │   ├── main.rs
│   │   │   ├── tcp_reader.rs
│   │   │   ├── ftp_server.rs
│   │   │   └── endpoints.rs
│   │   └── Cargo.toml
│   ├── siemens-s7-1200/      # PLC node (Rust)
│   │   ├── src/
│   │   │   ├── main.rs
│   │   │   ├── modbus.rs
│   │   │   └── endpoints.rs
│   │   └── Cargo.toml
│   └── validation-agent/     # Agent (Rust)
│       ├── src/
│       │   ├── main.rs
│       │   ├── validator.rs
│       │   ├── webhook.rs
│       │   └── endpoints.rs
│       └── Cargo.toml
├── dashboard/                # React observability UI (upstream contribution)
│   ├── src/
│   │   ├── components/
│   │   │   ├── NodeDag.jsx
│   │   │   ├── HealthGrid.jsx
│   │   │   ├── QRLog.jsx
│   │   │   ├── BeltControls.jsx
│   │   │   └── CameraFeed.jsx
│   │   └── App.jsx
│   └── package.json
├── proto/
│   └── camera_data.proto     # schema for qr-data message
├── config/
│   └── .env.example          # PLC IP, camera IP, stack size defaults
├── benchmarks/
│   └── latency_comparison.md # Python vs Bubbaloop timing results
└── README.md
```

---

## Existing Python prototype

The full working Python + React system lives at the root of this repo under `legacy/`. It serves as:

1. **Proof of hardware integration** — every protocol, packet format, and API call is proven working on the real devices
2. **Reference implementation** — each Python file maps 1:1 to a Bubbaloop node
3. **Regression test baseline** — benchmark comparisons run against this

| Python file | Maps to Bubbaloop node |
|-------------|------------------------|
| `legacy/backend/tcp_client.py` | `nodes/hikrobot-trigger/` + `nodes/validation-agent/` |
| `legacy/backend/ftp_server.py` | Embedded in `nodes/hikrobot-trigger/` |
| `legacy/backend/server.py` | Replaced by Zenoh (no equivalent needed) |
| `legacy/backend/api_server.py` | `/latest-image` endpoint in camera node |
| `legacy/src/` (React) | `dashboard/` (extended + contributed upstream) |

---

## Setup and running

> Full setup guide will be added as development progresses. Outline below.

### Prerequisites

```bash
# Rust toolchain
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Bubbaloop runtime
curl -sSL https://github.com/kornia/bubbaloop/releases/latest/download/install.sh | bash

# Node.js 20+ for dashboard
```



### Starting all nodes

```bash
# Start PLC node
cd nodes/siemens-s7-1200 && cargo run --release

# Start camera node
cd nodes/hikrobot-trigger && cargo run --release

# Start validation agent
cd nodes/validation-agent && cargo run --release

# Start observability dashboard
cd dashboard && npm run dev
```

---

## Skills I bring

- **Working industrial automation system** — already deployed on real hardware, not a simulation or prototype
- **Full-stack implementation** — Python (Flask, FastAPI, asyncio, pyftpdlib, Modbus TCP), React, Vite, TailwindCSS
- **Industrial protocol experience** — Modbus TCP packet construction, Hikrobot TCP stream parsing, FTP server integration
- **Hardware I own** — Hikrobot MV-ID6200M-00C-NNG, Siemens S7-1200, industrial conveyor — ready on day one

---

## What I will learn through this project

- **Rust** — ownership, lifetimes, async/await, tokio runtime
- **Zenoh** — pub/sub messaging, session management, publisher/subscriber APIs
- **Bubbaloop node contract** — schema, manifest, command API design
- **Protobuf** — message schema definition for node outputs
- **Open source contribution** — PR workflow, upstream API design, community review

---

## Related links

| Resource | URL |
|----------|-----|
| Bubbaloop repository | https://github.com/kornia/bubbaloop |
| kornia-rs repository | https://github.com/kornia/kornia-rs |
| GSoC project idea | https://summerofcode.withgoogle.com/programs/2025/organizations/kornia |
| Zenoh documentation | https://zenoh.io/docs/ |
| tokio-modbus | https://github.com/slowtec/tokio-modbus |

---

## Mentors

Edgar Riba · Jian Shi · Luis Ferraz · Miquel Farré

---

*This repository is part of a Google Summer of Code 2025 proposal. The hardware integration described here is real — I built and maintain the conveyor automation system this project is based on.*
