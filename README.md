# Railway Simulator

A physics-driven railway simulation engine that models realistic train dynamics — acceleration, braking, sensor noise, and safety-critical control decisions — with the long-term goal of scaling from a single train to a full multi-train operational network with signals, junctions, stations, and dispatch logic.

> **Note:** This is a public showcase repository. It documents the project's design, progress, and roadmap. Source code lives in a private repository and is not included here.

---

## Overview

The simulator models a train moving along a track under realistic physical constraints — drag, rolling resistance, gradient/curve resistance, traction limits, and braking behavior — while a controller layer makes real-time decisions (accelerate, coast, brake, emergency-stop) based on speed limits, safety buffers, and predicted braking distance.

The current build supports a **single-train architecture** and has been the focus of an **accuracy improvement sprint**, aimed at making the simulated motion and stopping behavior match real-world train dynamics as closely as possible. The project is architected from the outset with multi-train scaling in mind — the codebase has already been reviewed for structural "seams" that would need to change to support several trains operating at once.

---

## Features

**Physics & Dynamics**
- Realistic force modeling: drag, rolling resistance, gradient/curve resistance
- Power-limited and adhesion-limited traction calculations
- Predictive braking distance calculation using a forward-simulation loop
- Safe-speed and safety-buffer computation per tick

**Train Control**
- Operating-mode decision logic (Power / Coast / Brake / Emergency)
- Notch-based acceleration and braking, stepped toward target values each tick
- Stop-error tracking for overshoot/undershoot analysis

**Sensor Realism**
- Configurable noise model with a deterministic mode (for reproducible testing) and a randomized mode (for AI-training scenarios)

**Accuracy Metrics**
- Post-hoc reporting on stop-error, braking distance accuracy, and overshoot/undershoot statistics

**Track & Route Data**
- JSON-driven track and route definitions, with distance derived from coordinates and speed limits normalized to SI units internally

**Testing**
- A dedicated test suite covering physics, controller behavior, edge cases, routing, and full simulation runs

---

## Architecture (Current — Single Train)

```
engine/
├── main.py                # Entry point, route loading, train construction
├── train.py                # Train entity: identity + kinematic state
├── physics.py               # Dynamics engine: forces, braking, noise, safe speed
├── train_controller.py       # Decision logic: operating mode, notches
├── simulation.py             # Tick loop, history tracking, visualization
├── accuracy_metrics.py        # Stop-error and braking-accuracy reporting
├── noise_model.py             # Sensor noise abstraction
└── route_info.json            # Route data

world/
├── tracks.py                # Track geometry, speed limits, distance calculation
├── route.py                  # Route composition
├── station.py                 # Station data
├── tracks_info.json
├── schedules.json
└── trains.json

tests/
└── Full coverage across physics, controller, simulation, tracks, and routes
```

**Per-tick call flow:**

```
Simulation.sim_run()
  └─ Train.movement()
       ├─ sync_to_physics_engine()
       ├─ Physics.update_train_dynamics()
       │     ├─ calculate safe speed & safety buffer
       │     ├─ generate sensor noise
       │     ├─ predict braking distance
       │     ├─ run controller decision cycle
       │     └─ update speed state
       ├─ sync_from_physics_engine()
       └─ update_position()
```

Tracks and core physics math are already scoped per-instance with no shared global state, which means a meaningful part of the current architecture is already well-positioned for multi-train expansion.

---

## Roadmap

The project is being developed in stages, moving from a single-train simulation to a full operational railway network.

### ✅ Stage 0 — Single-Train Engine (current)
Core physics, controller, noise model, accuracy metrics, and test suite for one train on a defined route.

### 🔜 Stage 1 — Multi-Train Simulation
- **Infrastructure modules:** Clock, Event Engine, Track Block, Station, Junction, Signal, Dispatcher
- **Core refactor:** convert the single-train tick loop into a tick-all-trains loop, move history tracking to per-train, introduce canonical train identity
- **Train Registry:** replace the single `self.train` reference with a managed collection of active trains
- **Full wiring:** connect Dispatcher, Signals, Track Blocks, Stations, and Junctions into one coordinated system
- **Staged testing:** 1 → 2 → 5 → 10 trains, then stress tests and failure scenarios, so scaling issues are caught independently of architectural ones

Planned model: a **hybrid simulation** combining fixed-tick physics (for continuous, physically realistic motion) with an event-driven layer (for discrete railway coordination like signal changes, dwell timers, and scheduled departures) — giving both physical realism and operational efficiency.

### 🔜 Stage 2 — Global Time Engine
A shared simulation clock (IST-based) synchronizing all trains, events, and scheduling, with planned fast-forward, pause, and rewind support.

### 🔜 Stage 3 — Signal Systems
Track blocks, junctions, and signal aspects enforcing safe train separation and conflict-free routing.

### 🔜 Stage 4 — Dispatcher & Scheduling
Automated train dispatch based on schedules, priorities, and real-time conflict resolution — with a future path toward a human-operable dispatcher control board.

### 🔜 Stage 5 — Delay Propagation
Modeling how a delay in one train cascades into schedule impacts for others, using the event engine's causality model.

### 🔜 Stage 6 — AI Training Environment
Using the randomized noise mode and multi-train environment as a training ground for AI-driven train control policies, with support for comparing different "driver archetypes."

### 🔜 Stage 7 — Human Dispatcher Interface
An interactive control layer allowing a human to make live dispatch decisions within the simulated network.

---

## Tech Stack

- **Python** — core simulation engine
- **Matplotlib** — simulation visualization
- **Pytest** — test suite
- **JSON** — track, route, station, and schedule data

---

## Status

Actively in development. The single-train engine is functional and under an ongoing accuracy-improvement sprint; multi-train architecture planning is complete and implementation is beginning.

See the summary report in this repository for a detailed account of work completed so far.
