# GNSS Monitor

A personal project to build a full-stack GNSS accuracy and integrity monitoring system from the ground up — starting with raw signal data and ending with a live web dashboard backed by real hardware.

---

## Why This Project Exists

Consumer GPS is everywhere. Every phone, dashcam, and drone uses it constantly, and almost nobody questions what they're actually getting. The position on your screen looks precise. The blue dot moves smoothly. But underneath that polished interface, the receiver is making dozens of assumptions, discarding bad data, and presenting you with a number that could be accurate to two metres or twenty — and it rarely tells you which.

This project exists to answer that question properly.

The goal is to build a system that reads raw GNSS data — the same sentences coming off a real receiver — and characterises the quality of every position fix with honesty: how many satellites were visible, where they were in the sky, how strong their signals were, what the geometric dilution of precision was, and what that means for the expected accuracy of the reported position. Not a black box. Not a number on a map. A full picture.

The secondary goal is learning. This is a deliberate exercise in writing real systems software across a stack that spans low-level Rust, a Kotlin backend, and a web frontend — building each layer from scratch and understanding why it works, not just that it does.

---

## What GNSS Actually Is

GNSS stands for Global Navigation Satellite System — the umbrella term for all satellite positioning systems. GPS is the American one. GLONASS is Russian. Galileo is European. BeiDou is Chinese. A modern receiver uses all of them simultaneously, which is why your phone's position sentences start with `$GN` (multi-constellation) rather than just `$GP` (GPS only).

Every GNSS receiver works the same way at a fundamental level. Satellites broadcast signals containing their precise position and the exact time of transmission. The receiver measures how long each signal took to arrive and uses that to calculate its distance from each satellite. With enough satellites — at least four, ideally more — it can solve for its own position in three dimensions plus a clock correction.

The accuracy of the result depends heavily on geometry. If all visible satellites are clustered in one part of the sky, the position solution is poorly constrained and the error can be large. If satellites are spread evenly across the sky, the geometry is strong and the fix is reliable. This geometric effect is quantified as **Dilution of Precision (DOP)** — a number this project computes directly from the satellite positions.

---

## What This Project Builds

### The Core: Rust Parser

The foundation is a Rust binary that reads NMEA 0183 data — the text-based protocol that virtually every GNSS receiver outputs. NMEA is a stream of sentences, each one a comma-separated ASCII line beginning with `$`. Two sentence types matter most to this project:

- **GGA** — one per second, contains the position fix: latitude, longitude, altitude, fix quality, number of satellites used, and the receiver's own HDOP estimate.
- **GSV** — satellites in view: for each satellite, its PRN identifier, elevation above the horizon, azimuth from north, and signal-to-noise ratio.

The Rust parser reads a log file line by line, validates each sentence's checksum, extracts the relevant fields, and builds a structured representation of the entire session. The output is JSON, consumed by everything else in the system.

Rust is the right tool for this layer. It is fast, precise, and forces you to handle every error case explicitly — which matters when you are parsing a serial stream where any byte can be corrupted. The ownership model makes the data flow clear. There is no ambiguity about what owns the session data or who is responsible for writing it out.

### The Visualiser: Python

While the system is in its early phases, a Python script reads the JSON output and renders two things: a polar sky plot showing every satellite's position in the sky colour-coded by signal strength, and a bar chart of signal strength per satellite. This gives immediate visual feedback on what the receiver was seeing during a logged session.

Python is used here for speed of iteration, not performance. `matplotlib` can produce the polar projection needed for a sky plot in a few lines. Once the Kotlin web frontend exists, this script becomes a development and debugging tool rather than the primary interface.

### The Backend: Kotlin (planned)

The Kotlin layer is a server built with **Ktor** — Kotlin's native asynchronous web framework. It will sit between the Rust core and the web frontend, responsible for:

- Accepting NMEA data — either from uploaded log files or streamed live from a connected hardware receiver
- Invoking the Rust parser and receiving the structured output
- Storing session history in a local database (SQLite initially)
- Exposing a clean REST and WebSocket API for the frontend to consume

Kotlin is chosen deliberately as a learning target. It is a modern, expressive JVM language with an excellent type system, first-class coroutines for async I/O, and a growing ecosystem. For someone with a Java background, Kotlin reads naturally from the first day while rewarding deeper learning over time — null safety, data classes, extension functions, and sealed classes all become tools worth understanding properly rather than novelties.

The Kotlin backend also serves an architectural purpose: it decouples the signal processing (Rust) from the presentation layer (web), which makes both easier to evolve independently.

### The Frontend: Web (planned)

The web layer replaces the Python visualiser with something live. A TypeScript frontend — likely using a lightweight framework — will connect to the Kotlin backend over WebSockets and render:

- A real-time sky plot updating as satellite positions change
- HDOP and fix quality trends over a session
- Historical session comparison — was accuracy better at this location last week?
- An at-a-glance view of current receiver health when hardware is connected

The frontend is the last layer to be built and the one with the most freedom. The domain knowledge built up in the earlier phases — what makes a good fix, what HDOP actually means, what multipath interference looks like in signal data — will make the design decisions obvious rather than arbitrary.

### The Hardware: u-blox Receiver (planned)

The final concrete addition is a **u-blox NEO-M8N or M9N** receiver module (approximately £30–50). This is the hardware used by serious hobbyists, drone manufacturers, and precision agriculture systems. Connecting it over USB or UART replaces static log files with a live serial stream, turning the monitor into a real-time instrument.

The u-blox hardware also supports the **UBX binary protocol** — a richer, more precise alternative to NMEA that exposes data the text sentences do not include. Phase 4 of this project involves adding UBX parsing alongside NMEA, which is where the signal-level detail needed for Phase 3's error characterisation becomes fully accessible.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│                   Data Sources                      │
│                                                     │
│   NMEA log file          u-blox receiver (live)     │
│   (.nmea / .txt)         (USB/UART serial stream)   │
└────────────────┬────────────────────┬───────────────┘
                 │                    │
                 ▼                    ▼
┌─────────────────────────────────────────────────────┐
│              Rust Core  (gnss-monitor)              │
│                                                     │
│   NMEA parser → checksum validation → structured    │
│   session data → DOP calculation → error bounds     │
└────────────────────────────┬────────────────────────┘
                             │ JSON
                ┌────────────┴────────────┐
                │                         │
                ▼                         ▼
┌───────────────────────┐   ┌─────────────────────────┐
│   Python Visualiser   │   │    Kotlin Backend        │
│   (Phase 1 & dev)     │   │    (Ktor, Phase 4+)      │
│                       │   │                          │
│   Sky plot            │   │   REST API               │
│   SNR chart           │   │   WebSocket stream       │
│   Session summary     │   │   Session storage        │
└───────────────────────┘   └───────────────┬──────────┘
                                            │
                                            ▼
                            ┌─────────────────────────┐
                            │    Web Frontend          │
                            │    (TypeScript, Phase 5) │
                            │                          │
                            │   Live sky plot          │
                            │   HDOP trends            │
                            │   Session history        │
                            └─────────────────────────┘
```

---

## Project Phases

### Phase 1 — Parse & Plot
**Status: In progress**

Build the Rust NMEA parser and Python visualiser. At the end of this phase the project can ingest any NMEA log file and produce a sky plot showing which satellites were visible, where they were, and how strong their signals were — plus a session summary with HDOP statistics.

Tasks in this phase are tracked separately in `phase1-todo.md`.

Key deliverables:
- GGA and GSV sentence parsing in Rust
- NMEA checksum validation
- JSON session output
- Polar sky plot and SNR bar chart in Python

---

### Phase 2 — Geometry & Quality
**Status: Not started**

Implement DOP calculation from scratch. Rather than trusting the HDOP value the receiver reports, this phase derives it independently from the satellite positions using the geometry matrix. This is the core mathematics of the project: build the unit vector from receiver to each satellite, form the geometry matrix H, compute `(HᵀH)⁻¹`, and take the square roots of the diagonal elements to get GDOP, PDOP, HDOP, and VDOP.

This phase also adds position scatter analysis — plotting the spread of fixes over a session to visualise real-world accuracy.

Key deliverables:
- Geometry matrix construction from satellite azimuths and elevations
- DOP computation and comparison against receiver-reported values
- Position scatter plot

---

### Phase 3 — Error Characterisation
**Status: Not started**

Add multipath detection heuristics, expected error bounds based on satellite geometry and signal quality, and session logging over time. This phase is where the monitor starts producing genuinely novel analysis rather than replicating what the receiver already tells you.

Multipath interference — signals reflecting off buildings or terrain before reaching the antenna — causes characteristic SNR instability and position scatter. Detecting it from the data patterns is an interesting signal processing problem with no perfect solution.

Key deliverables:
- Multipath detection heuristics based on SNR variance and elevation-dependent thresholds
- Expected positional error bounds
- Session logging to persistent storage
- Session replay

---

### Phase 4 — Hardware & Kotlin Backend
**Status: Not started**

Connect a physical u-blox receiver and build the Kotlin backend. This phase transforms the project from an offline analysis tool into a live monitoring system. The Kotlin server manages incoming data, stores sessions, and exposes the API that the frontend will consume.

This is also where UBX binary protocol parsing is added — giving access to raw pseudorange and carrier phase data that NMEA does not expose, and opening the door to more sophisticated accuracy estimation in later work.

Key deliverables:
- Live serial stream ingestion from u-blox hardware
- UBX binary protocol parser (alongside existing NMEA parser)
- Ktor backend with REST API and WebSocket session streaming
- SQLite session storage

---

### Phase 5 — Web Frontend
**Status: Not started**

Build the TypeScript web frontend that replaces the Python visualiser as the primary interface. The frontend connects to the Kotlin backend over WebSockets and renders a live sky plot, HDOP trends, and session history in the browser.

Key deliverables:
- Live sky plot updating in real time
- HDOP and fix quality trend charts
- Historical session browser
- Receiver health dashboard

---

## Technology Choices

| Layer | Technology | Why |
|---|---|---|
| Parser & signal processing | Rust | Performance, precision, explicit error handling |
| Development visualiser | Python + matplotlib | Fast iteration, polar plot support |
| Backend | Kotlin + Ktor | Modern JVM language, coroutines, good learning target |
| Frontend | TypeScript | Type safety, broad ecosystem, pairs well with Kotlin |
| Database | SQLite | Zero infrastructure, sufficient for single-user monitoring |
| Hardware | u-blox NEO-M8N/M9N | Industry standard, supports both NMEA and UBX |

---

## Repository Structure

```
gnss-monitor/
├── src/
│   ├── main.rs         # Entry point and CLI argument handling
│   ├── nmea.rs         # NMEA 0183 sentence parser (GGA, GSV)
│   ├── types.rs        # Core data types (Fix, SatelliteObservation, Session)
│   └── output.rs       # JSON serialisation
├── visualiser/
│   └── skyplot.py      # Python sky plot and SNR visualiser
├── data/               # NMEA log files (not committed)
├── Cargo.toml
├── phase1-todo.md      # Granular task list for Phase 1
└── README.md
```

---

## Getting Started

**Prerequisites**

```bash
# Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Python dependencies
pip install matplotlib numpy
```

**Running the parser**

```bash
cargo run -- --file data/your_log.nmea --output output.json
```

**Running the visualiser**

```bash
python3 visualiser/skyplot.py --input output.json
```

---

## NMEA Sample Data

Good sources for NMEA log files to test with:

- **GitHub — gps-nmea-log-files**: `github.com/esutton/gps-nmea-log-files` — a collection of real device logs specifically for parser testing
- **OpenStreetMap GPS traces**: `openstreetmap.org/traces` — real-world logs, filter for NMEA format (not GPX)
- **Your own Android phone**: the app *NMEA Tools* records raw sentences from the device receiver

When evaluating a file, open it in a text editor and confirm the lines begin with `$GP`, `$GN`, or `$GL`. If the file is XML it is GPX format, not NMEA, and will not work with this parser.

---

## HDOP Reference

The receiver's reported HDOP (Horizontal Dilution of Precision) gives a rough guide to fix quality. Phase 2 of this project computes this independently.

| HDOP | Interpretation |
|---|---|
| < 1.0 | Ideal — excellent satellite geometry |
| 1 – 2 | Excellent — suitable for precision applications |
| 2 – 5 | Good — suitable for most navigation |
| 5 – 10 | Moderate — position accuracy is degraded |
| > 10 | Poor — geometry is weak, large errors likely |