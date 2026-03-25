# bubbaloop-industrial-edge

> **GSoC 2026 — kornia / Bubbaloop**  
> A real-world, production-grade Bubbaloop application running on live industrial hardware integrating a Industrial smart camera, Siemens PLC, and a full observability dashboard contributed upstream to Bubbaloop.

## About Me

I am Vishal Kumar Mahto, just graduated in Production and Industrial Engineering from Birla Institute of Technology, Mesra and currently working as a Robotics/Automation Engineer

My work focuses on building real world industrial systems that combine robotics, computer vision, and low latency control. I have hands on experience designing and deploying production systems using PLCs, industrial cameras, and real time pipelines.

---

## What this project is

This project focuses on implementation of my Google Summer of Code 2026 proposal for the [kornia](https://github.com/kornia) organization under the project idea **"5.Bubbaloop application for robotics or IoT"**

The application is an **Industrial Conveyor Belt Automation System** for warehouse inward goods processing. It is not a simulation,the hardware is real, running, and already has a working Python prototype. This GSoC project ports that system to a high-performance, event-driven Bubbaloop pipeline written in Rust, and contributes node observability tooling back to the Bubbaloop runtime.

---

## Hardware I own and am actively using

| Device | Model | Protocol | Role |
|--------|-------|----------|------|
| Smart code reader | Industrial Camera | TCP socket + FTP | Captures images, decodes QR codes at 60/sec |
| PLC | Siemens PLC | Modbus TCP  | Controls physical conveyor belt motor |
| Conveyor belt | Industrial grade | — | Physical goods transport |
| Edge host | Ubuntu PC (GTX 1650Ti) | — | Runs all Bubbaloop nodes |

The Modbus TCP control logic, FTP image receiver, QR TCP stream parser, and full React dashboard are **already implemented and working** in Python/React. The GSoC work is the architectural migration to Bubbaloop + the upstream observability contribution.


## Architecture

The system consists of three main nodes:

#### Camera node
Reads QR codes from the Hikrobot camera and publishes scan data

#### Validation agent
Checks if scanned QR count matches expected stack size and detects duplicates

#### PLC node
Controls the conveyor by sending commands to the Siemens PLC

If there is any mismatch or duplicate QR, the conveyor is stopped immediately.



## End-to-end data flow

1. Worker places boxes on conveyor belt

2. Photoelectric sensor triggers Hikrobot camera hardware capture

3. Camera node (Rust):
   - Reads QR strings from TCP stream 
   - Receives JPEG image via FTP server
   - Publishes to Zenoh: bubbaloop/local/camera/qr-data

4. Validation agent (Rust) wakes on Zenoh message:
   - Checks for duplicates against session_seen_set
   - Compares codes.count vs expected_stack_size

5a. COUNT MATCHES:
   - Adds QRs to session accumulator
   - POSTs each code to cloud webhook
   - Publishes SUCCESS to agent/scan-result
   - Belt continues running ,PLC untouched

5b. COUNT MISMATCH:
   - Publishes STOP to plc/command immediately
   - Publishes MISMATCH event to agent/scan-result

6. PLC node (Rust) on STOP command:
   - Acquires plc_lock
   - Writes Modbus FC6 value=1 to register 40001
   - Belt stops physically within milliseconds
   - Publishes plc/status 

7. Dashboard:
   - Receives scan-result event → shows error popup or green tick
   - Receives plc/status → updates belt status badge
   - Operator dismisses error, fixes stack, clicks Rescan

---
## Project timeline — 350 hours over 12 weeks

 
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
- Week 8: Implement cloud webhook posting (`request` HTTP client), wire MCP tool for `set_stack_size`, implement agent's five endpoints
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

## Communication & Availability Plan

Open source development is collaborative, and I believe consistent communication is as important as writing good code.

### Timezone & Availability

I am based in India (Asia/Kolkata, UTC+5:30).  
I work full time from 10:00 AM to 7:30 PM, and I plan to dedicate focused hours outside of work for this project.

I will commit approximately 30 hours per week during the GSoC period, ensuring a total contribution of around 350 hours across 12 weeks. My working hours will primarily be early mornings, late evenings, and weekends.

---
## Mentors

Edgar Riba · Jian Shi · Luis Ferraz · Miquel Farré

---
