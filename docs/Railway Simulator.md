

Progress Summary Report — Single-Train Engine & Accuracy Improvement Sprint

# 1. Executive Summary

This report summarizes the work completed to date on the railway simulator project. The team has built and stabilized a single-train simulation engine covering physics, control logic, sensor noise, and accuracy tracking, and has completed a structural review to prepare the codebase for multi-train expansion. A dedicated accuracy improvement sprint addressed a control-logic oscillation bug and validated stopping and braking behavior against expected physical outcomes.

# 2. Project Background

The railway simulator models a train's motion along a track under realistic physical constraints, with a controller layer that makes real-time driving decisions (power, coast, brake, emergency stop). The long-term goal is to scale this into a full multi-train network with signals, junctions, stations, and centralized dispatch. The current phase of work has focused on getting the single-train engine correct and reliable before that expansion begins.

# 3. Work Completed

## 3.1 Core Single-Train Architecture

The engine was built out across a clear separation of concerns:

●        train.py — train identity and kinematic state, with a 1:1 owned Physics engine instance

●        physics.py — force modeling (drag, rolling resistance, gradient/curve resistance), traction limits, predictive braking distance, and safe-speed calculation

●        train_controller.py — operating-mode decisions and notch-based acceleration/braking control

●        simulation.py — the tick loop, state history tracking, and visualization

●        accuracy_metrics.py — post-run reporting on stop-error and braking accuracy

●        noise_model.py — sensor noise abstraction, separated into its own dedicated file for clarity and reuse

●        tracks.py — track geometry and route data, loaded and validated from JSON

## 3.2 Accuracy Improvement Sprint

A focused sprint was run to verify and improve the physical and structural accuracy of the simulation, including:

●        Diagnosing and fixing a BRAKE ↔ COAST oscillation bug, traced to mismatched target values between safe_speed_ms and speed_limit_ms in the controller's mode-selection logic

●        Reviewing merged files to confirm accuracy improvements held both quantitatively (stop-error, overshoot/undershoot metrics) and structurally (no regressions introduced to module boundaries)

●        Validating predictive braking distance calculations against expected stopping behavior

## 3.3 Test Suite Stabilization

The test suite (covering physics, controller behavior, edge cases, tracks, routes, and full simulation runs) was audited and repaired, including correcting widespread mock patch path errors so that unit tests correctly target the engine's actual module structure rather than stale references.

## 3.4 Multi-Train Readiness Review

Ahead of the next development phase, the entire single-train codebase was reviewed module-by-module to identify structural "seams" — places where the current design assumes exactly one train and would need to change to support several trains running concurrently. Key findings:

●        simulation.py holds the highest-severity seams: a route-completion tick loop (rather than a tick-all-trains loop) and flat, non-keyed history lists

●        train.py and physics.py are largely multi-train-ready already, since each Train correctly owns its own Physics instance with no shared global state

●        tracks.py requires no changes at all, since tracks are shared, read-only, and already support being referenced by multiple trains

●        train_controller.py has reserved-but-unused placeholders for signal/occupancy awareness, intentionally deferred pending its planned redesign

This review produced a severity-ranked list of seams and a five-step execution plan (infrastructure modules → core refactor → train registry → full wiring → staged multi-train testing) to guide the next phase of development.

# 4. Current Module Status

|   |   |   |
|---|---|---|
|**Module**|**Status**|**Notes**|
|train.py|Stable|Owns identity + kinematic state; already multi-train-friendly|
|physics.py|Stable|Core dynamics math verified; no shared state|
|train_controller.py|Mid-redesign|Physics-assisted controller; perception-based redesign planned|
|simulation.py|Needs refactor|Single-train tick loop; largest concentration of multi-train seams|
|accuracy_metrics.py|Functional|Stop-error and braking-accuracy reporting; not yet per-train scoped|
|noise_model.py|Complete|Separated into its own module; deterministic + randomized modes|
|tracks.py|Stable|No changes needed for multi-train|
|Test suite|Stabilized|Mock patch paths corrected across test files|

# 5. Next Steps

With the single-train engine stabilized and the multi-train seam analysis complete, the next phase of work begins implementation of the new infrastructure modules (Clock, Event Engine, Track Block, Station, Junction, Signal, Dispatcher) as self-contained units, followed by the core refactor of the simulation loop and the introduction of a train registry. Full details are tracked in the project roadmap.