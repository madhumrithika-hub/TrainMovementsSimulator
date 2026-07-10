

## Architecture Map — Current State (Single-Train Engine)

### `main.py` — Entry point / bootstrapping

**Owns:** route loading, route selection (interactive input), initial `Train` construction. **Call chain:** `route_loader()` → `route_extractor()` → `get_route_input()` (blocks on user input) → `route_selector()` → `active_routes_initializer()` → `train_initializer()` → constructs `Simulation` → `sim.sim_run()`. **Note (observation, not action):** this still references `from world import tracks` and `simulation(train1, active_routes)` / `sim.sim_run(selected_route)` — doesn't match the current `Simulation` class signature (no-arg `sim_run()`). Likely stale from an earlier merge pass.

### `tracks.py` — Track data model + loading

**Owns:** the `Track` class (geometry, speed limit, distance — computed via Euclidean distance from coordinates), JSON validation (`validate_track_data`), JSON loading (`load_tracks`), and the module-level eager-loaded `tracks` list. **Key design choice:** distance is _derived_ from coordinates, not given directly in JSON. Speed limit is stored in km/h but exposed via `.speed_limit` property as m/s — so downstream consumers always get SI units without needing to know about the conversion. **State ownership:** track identity is `track_id`/`id` (int or str). No mutation after construction except `calculate_distance()`, which is only called once internally. **No dependency on anything else in the system** — lowest-level module, correctly so per RULE-05.2 (downward-only dependencies).

### `train.py` — Train entity / state container

**Owns:** train identity (`train_name`, `train_no`, `train_model`), kinematic state (`current_speed_ms`, `current_position_m`, `acceleration_ms2`), and the reference to its `Physics` engine instance. **Key design choice:** `Train` _constructs and owns_ exactly one `Physics` instance in `__init__`. This is a tight 1:1 coupling — a `Train` cannot exist without a `Physics` engine, and there's no path for a `Physics` instance to be shared or reused across trains. **Call chain per tick:** `movement()` → `sync_to_physics_engine()` (push state into physics) → `update_speed_state()` (delegates to `physics_engine.update_train_dynamics()`) → `sync_from_physics_engine()` (pull results back, including `operating_mode`, notches, `safe_speed_ms`) → `update_position()` (integrates position from speed). **State ownership boundary:** `Train` is the source of truth for position/speed _between_ ticks; `Physics` is where the actual update math happens _during_ a tick. The sync methods are the seam between them. **`change_track()`** resets `current_position_m` to 0 and pushes the new track into both `Train` and `Physics` directly — this is the only place track transitions happen.

### `physics.py` — Train dynamics engine

**Owns:** the physical simulation math — drag, rolling resistance, gradient/curve resistance, traction force limits (power-limited, adhesion-limited), brake deceleration mapping, safe-speed calculation, predictive braking distance, sensor noise application, and final speed-state update. **Owns and constructs a fresh `TrainController` every tick** inside `_create_controller()` — this is the RULE-05.3 concern flagged earlier (allocation inside the per-tick path). **Call chain (`update_train_dynamics`, the orchestrator method):**

1. `_calculate_safe_speed()` → sets `self.safe_speed_ms`, returns `distance_remaining_m`
2. `_calculate_safety_buffer()`
3. `_generate_sensor_noise()`
4. `_predict_braking_distance()` (runs a forward simulation loop using `train_controller.BRAKE_NOTCH_TABLE`)
5. `_log_current_state()` (print-based diagnostics)
6. `_create_controller()` → builds a `TrainController` with all the above as inputs
7. `_apply_controller_decision()` → calls `controller.run_control_cycle()`, pulls `operating_mode`/notches back onto `self`
8. `_update_speed_state()` → smooths acceleration, applies noise, clamps to `[0, max_speed_ms]`

**Note:** `_create_controller()` currently calls `TrainController(...)` with 10 positional args, but `TrainController.__init__` in the project file takes 11 (it now includes `current_position_m`, which physics.py's call site doesn't pass). That's a live mismatch — worth flagging now since you're about to study this module closely, though not something to fix mid-documentation.

**Dependency direction:** `physics.py` imports `train_controller` (uses `BRAKE_NOTCH_TABLE` directly, plus constructs `TrainController`). That's correctly downward per RULE-05.2 — physics is "higher" than controller in this stack, even though controller is lower-level conceptually in terms of decision-making.

### `train_controller.py` — Decision logic (currently mid-redesign)

**Owns:** `OperatingMode` enum, notch tables (`POWER_NOTCH_TABLE`, `BRAKE_NOTCH_TABLE`), the safety/operational mode-selection logic (`select_operating_mode`), notch target calculation (`calculate_target_notches`), notch stepping (`step_notches_toward_targets`), and final acceleration output (`calculate_acceleration_ms2`). **Per-cycle orchestration:** `run_control_cycle()` → `calculate_stop_error()` → `select_operating_mode()` → `calculate_target_notches()` → `step_notches_toward_targets()` → `calculate_acceleration_ms2()`. **Current inputs accepted in `__init__`:** `max_speed_ms`, `predicted_braking_distance_m`, `distance_remaining_m`, `safety_buffer_m`, `safe_speed_ms`, `current_speed_ms`, `acceleration_notch`, `brake_notch`, `operating_mode`, `speed_limit_ms`, `current_position_m`. This is the "physics-assisted" controller (receives pre-computed `predicted_braking_distance_m` and `safe_speed_ms`) that's currently slated for replacement by the perception-based redesign per memory. **Reserved-but-unused:** `signal_aspect`, `occupancy_ahead`, `speed_restriction_ms` — set to `None`/`False` in `__init__`, not yet wired into `select_operating_mode()` (which still uses local placeholders `signal="GREEN"` and `dwell_time=0`). **Known issues already on your radar** (per memory): naming inconsistency, the BRAKE hysteresis not yet taking prior `stop_reason` as explicit input, the EMERGENCY persistence latch behavior.

### `simulation.py` — Single-train orchestration loop

**Owns:** the tick loop itself, history collection (`history_position_m`, `history_speed_ms`, `history_safe_speed_ms`), and matplotlib visualization at the end. **Validation methods** (`validate_train`, `validate_routes`) run once at construction — boundary validation per RULE-04.2. **Call chain:** `sim_run()` → for each track in `active_routes`: `train.change_track(track)` → while not at end of track: append history, `train.movement()`, increment step counter → after all tracks: plot. **Single-train assumption baked in directly:** `self.train` is one object, `history_*` are flat lists indexed implicitly by tick, not by train. This is the seam Puzzle 2 will need to mark clearly.

### `accuracy_metrics.py` — Post-hoc performance tracking

**Owns:** stop-error recording and averaging, ideal vs. actual braking distance comparison, overshoot/undershoot counting and distance averaging, and a text report generator. **Not currently wired into `simulation.py` or `train.py` anywhere in this set of files** — it exists as a standalone utility, presumably called externally or in tests, but I don't see an integration call site in the modules shown. Worth confirming whether that wiring exists elsewhere or is still pending.

### `noise_model.py` — Sensor noise abstraction

**Owns:** two modes — `DETERMINISTIC_MODE` (always 0.0, used for reproducible tests per RULE-07.2) and `AI_TRAINING_MODE` (uniform random noise). Used by `Physics` via `self.noise_model.generate_noise()`.

---

### Call chain, top to bottom, one full tick

```
main.py
  └─ Simulation.sim_run()
       └─ Train.movement()
            ├─ sync_to_physics_engine()
            ├─ Physics.update_train_dynamics()
            │     ├─ _calculate_safe_speed()
            │     ├─ _calculate_safety_buffer()
            │     ├─ _generate_sensor_noise() → NoiseModel.generate_noise()
            │     ├─ _predict_braking_distance()  [uses train_controller.BRAKE_NOTCH_TABLE]
            │     ├─ _create_controller() → new TrainController(...)
            │     ├─ _apply_controller_decision() → TrainController.run_control_cycle()
            │     │     ├─ calculate_stop_error()
            │     │     ├─ select_operating_mode()
            │     │     ├─ calculate_target_notches()
            │     │     ├─ step_notches_toward_targets()
            │     │     └─ calculate_acceleration_ms2()
            │     └─ _update_speed_state()
            ├─ sync_from_physics_engine()
            └─ update_position()
```



##  Marking the Single-Train Seams


### `simulation.py` — the biggest concentration of seams

- **`self.train` (singular)** — set once in `__init__`, referenced everywhere in `sim_run()` as _the_ train. There's no list, no dict, no identity-keyed lookup. To support N trains, every reference to `self.train` becomes a question of "which train."
- **`validate_train(train)`** — validates one `Train` instance. Would need to become "validate each train in a collection."
- **History lists are flat and global to the simulation, not scoped per train:**
    
    ```python
    self.history_position_m=[]self.history_speed_ms=[]self.history_safe_speed_ms=[]
    ```
    
    These are appended to once per tick, implicitly assuming "the one train's" state at that tick. With multiple trains, a single flat list can't tell you _whose_ speed/position it's recording without an external index correlation — this needs to become per-train (e.g. keyed by `train_no` or `train_id`).
- **`global_pos` tracking in `sim_run()`** — accumulates distance across tracks for _one_ train's route. If trains run different routes (or the same route at different times), this single accumulator can't serve more than one train.
- **The track-traversal loop itself:**
    
    ```python
    for track in self.active_routes:    self.train.change_track(track)    while self.train.current_position_m < track.distance_m or self.train.current_speed_ms > 0:
    ```
    
    This loop runs _one train to completion on one route_ before considering anything else. It's fundamentally a "drive train A from start to finish" loop, not a "advance all active trains by one tick" loop. This is the structural inversion flagged earlier — the loop nesting itself (outer: tracks: inner: ticks) would need to become (outer: ticks; inner: trains) for trains to move _together_ in real time rather than sequentially.
- **The plotting block at the end** assumes one speed series, one safe-speed series, one position series — `plt.plot(self.history_position_m, self.history_speed_ms, ...)` called once. With N trains you'd want either N lines on one plot or a legend distinguishing trains — currently there's no concept of "which line belongs to which train" since there's only one line.

### `train.py` — fewer seams than you'd expect, but one matters

- **The tight 1:1 `Train` ↔ `Physics` construction in `__init__`** isn't itself a problem for multi-train — each `Train` correctly owns its own `Physics` instance, so trains _are_ already somewhat self-contained. This is actually a point in your favor: `Train` doesn't assume it's the only train, it just doesn't yet have an identity that the _outer_ system uses to distinguish it from others in a registry.
- **No unique, system-recognized `train_id`** — `train_no` exists as a field, but nothing currently enforces uniqueness or uses it as a lookup key anywhere. For a registry to work, this needs to become the canonical identity key.

### `physics.py` — essentially seam-free for multi-train, with one caveat

- Each `Physics` instance is already scoped to the one `Train` that owns it — no shared/global state, no list assumptions. Good.
- **Caveat:** `_create_controller()` constructs a fresh `TrainController` every tick (the RULE-05.3 allocation concern from before). That's not a _correctness_ seam for multi-train — it'll still work per-train — but it's a _scaling_ seam: 10 trains × fresh controller object per tick, per train, every tick, is 10x the allocation cost. Worth keeping on the list for Puzzle 4/5 timing, not now.

### `train_controller.py` — no train-count seam, but a _signal/occupancy_ seam that will matter later

- The controller itself operates on a single train's kinematic inputs — nothing here assumes "I am the only train." It's blind to other trains entirely.
- **But:** the placeholder `signal="GREEN"` and `dwell_time=0` inside `select_operating_mode()` are stand-ins for exactly the kind of cross-train information (is the block ahead occupied by another train? what does the signal say because of that?) that multi-train introduces. Not a structural seam in the sense of "this breaks with N trains" — more accurately, this is the _socket_ where multi-train awareness will eventually plug in, currently capped off with placeholders.

### `tracks.py` — no seam at all

- Tracks are shared, read-only, immutable-after-construction infrastructure. Multiple trains referencing the same `Track` objects is already fine — nothing here assumes a 1:1 train-to-track relationship. This module needs zero changes for multi-train.

### `accuracy_metrics.py` — a seam, but a quiet one

- `AccuracyMetrics` as written holds a single running set of stats (`stop_error_list`, `overshoot_count`, etc.) with no train identity attached. If you want per-train accuracy reporting (which you likely will, to compare driving behavior across trains per the driver-archetype goal in memory), this needs either one `AccuracyMetrics` instance per train, or per-train keying added internally. Currently it has neither.

### `main.py` — seam by omission, not by design

- `train_initializer()` hardcodes constructing exactly one `Train` with one hardcoded name/number/type. Not "broken" for multi-train so much as simply not yet doing the job — there's no loop, no list of train configs, no concept of staggered entry times.

---

## Summary — seams ranked by how much they'll cost to address

|Seam|Module|Severity|
|---|---|---|
|Tick loop structure (route-completion loop vs. tick-all-trains loop)|`simulation.py`|**High** — structural inversion needed|
|Flat, non-keyed history lists|`simulation.py`|**High** — but mechanical to fix once identity exists|
|No canonical train identity used as a lookup key|`train.py`|**Medium** — `train_no` exists, just unused as a key|
|Single plot series|`simulation.py`|**Low** — cosmetic, fix last|
|Fresh `TrainController` per tick|`physics.py`|**Low priority now, matters at scale**|
|Accuracy metrics not train-scoped|`accuracy_metrics.py`|**Medium** — depends how soon you want per-train comparison|
|Signal/occupancy placeholders|`train_controller.py`|**Deferred** — intentionally, pending controller redesign|
|Hardcoded single-train construction|`main.py`|**Low** — straightforward once registry exists|

Everything in `tracks.py` and `physics.py`'s core dynamics math needs no rework at all — that's good news, it means a meaningful chunk of your stack is already multi-train-ready by virtue of being properly scoped per-instance.


Here's the text — copy into your own document as you like.

---
# MULTI-TRAIN ARCHITECTURE:

### Hybrid Simulation Model — Fixed-Tick Physics with Event-Triggered Coordination

#### Rationale

Pure procedural (fixed-tick) simulation is the more physically realistic model, since train motion is continuous — speed, position, and acceleration must be sampled at a steady rate to produce credible braking/acceleration curves, valid noise application, and accurate stop-error measurement. However, a purely procedural model is operationally inefficient for railway coordination concerns (signals, dispatch, scheduling, delays), where discrete-event simulation is the standard and more useful approach. The hybrid model is designed to capture the realism of procedural physics while gaining the operational clarity of event-driven coordination, rather than treating these as mutually exclusive choices.

#### Core Structure

There are two coordinated mechanisms operating against the same time axis:

**1. The Physics Clock (Tick Loop)**  
Advances every active train's position, speed, and acceleration by a fixed `TIME_STEP_S` every tick, unconditionally. This layer is unchanged from the current single-train implementation in principle — `Train.movement()` and `Physics.update_train_dynamics()` continue to run every tick, for every active train. This preserves physical realism and is non-negotiable for credible kinematic behaviour.

**2. The Event Queue (Event Engine)**  
A separate, time-ordered queue of discrete, schedulable state changes, processed against the same clock the physics layer uses. The event queue does not replace ticking — it rides alongside it.

Per-tick loop shape:

```
each tick:
    advance simulation_clock by TIME_STEP_S
    process any events scheduled for this tick (may change train/world state)
    for each active train:
        train.movement()   ← physics runs every tick, unconditionally
    check whether any state changes this tick should generate new events
```

#### What Qualifies as an Event

**Treated as events** (discrete, schedulable, infrequent):

- A train's scheduled departure time arriving
- A train completing its route / being retired from the registry
- A signal changing aspect (e.g. RED → GREEN) due to occupancy change ahead
- A station dwell-time timer expiring
- A scheduled delay or disruption firing

**Not events — remain purely tick-driven:**

- Speed/position/acceleration updates (continuous, every tick)
- Controller notch stepping (inherently incremental; only meaningful when evaluated every tick)

#### Why This Matters at Multi-Train Scale

The principal benefit is not computational efficiency — ticking 10+ trains is cheap regardless — but **decoupling cross-train coordination from the tick loop's inner logic**:

- Occupancy and signal state only need to be recomputed when a train enters or exits a block, not polled every tick for every train against every block.
- Staggered train entry (trains departing at different scheduled times) is naturally expressed as queued events rather than special-cased tick-loop conditionals.
- Delay propagation (a core Phase 3 goal) maps directly onto event-to-event causality — a late arrival is an event, and its downstream effect on the next train's schedule is a new event triggered by it — rather than requiring delay state to be re-derived implicitly from raw kinematics every tick.

#### Boundary of Responsibility

The event engine's role ends at producing updated world state (e.g., "signal at block 7 is now RED," "train 4 has arrived at station X"). It does not determine how a train's controller reacts to that state — that responsibility remains entirely with the train controller's safety-critical decision hierarchy. This boundary is intentional: it keeps the multi-train coordination infrastructure and the train controller redesign as separable, independently reviewable workstreams.

#### Structural Implications

- **Clock** must be built first; both the tick loop and the event queue key off it.
- **Event Engine** depends only on Clock, and should be built as a pair with it.
- **TrackBlock, Station, Junction** (topology/occupancy layer) sit above Clock/Event Engine conceptually, though they exist as data structures independent of them; they become event _sources and targets_ once wired in.
- **Signal** depends on TrackBlock/Junction and the Event Engine, since signal state changes are themselves events.
- **Dispatcher** depends on all of the above, plus the train registry.
- The **train registry** has no dependency on Clock/Event Engine and can be built in parallel with them.

## NEW MODULES TO BE ADDED:



- Train registry
- Clock
- Event Engine
- Track Block
- Station
- Junctions
- Signal
- Dispatcher


## Simulation components:





### Clock:

Maintains the global simulation time. Every simulation tick advances the clock by a fixed interval and synchronizes all time-dependent components including train physics, event processing, scheduling, and logging. Returns time in "HH:MM" format for display, but from programs it gives the time in minutes from midnight.


### Event Engine:

Maintains a time-ordered queue of simulation events. It schedules, stores, executes, and removes discrete events as the simulation clock advances. Events include departures, arrivals, signal aspect changes, dwell-time expiration, delays, and other operational state changes.

## Topology:

### Track Block:

Divides the railway network into occupancy blocks. Each block stores its geographical limits, connected tracks, current occupancy, and reservation status. Track Blocks form the primary safety layer for signaling by ensuring that only one train may occupy a block at a time.

> Note: There will be a `json` file on the locations of track blocks and `.py` file on the operations of track blocks


### Station:

Contains all stations within the operational area of the simulator. The station manages platform occupancy, scheduled arrivals and departures, dwell times, and station-specific operational rules. It coordinates with the Dispatcher and Signal System to receive and dispatch trains while maintaining safe and efficient station operations.

> Note: This will be a `json` file on the locations of stations and it's procedures and a `.py` file


### Junction:

Contains all railway junctions within the operational area of the simulator. A junction manages track branching, route selection, point (switch) alignment, route locking, and conflict prevention. It determines which outgoing track a train may enter and coordinates with the Signal System and Dispatcher to ensure trains are routed safely without conflicting movements. 

> Junctions coordinate with the Dispatcher and Signal System to safely establish and release routes through the junction.
> 
> Note: This will be a `json` file on the locations of junctions and a `.py` file


## Operational components:

## Train Registry (`json` file):

Contains the details of all trains that would be simulated in the operational region of the simulator. It should be easily scalable, it should have three components, Scheduled train, active trains, passed trains. These three reflect whether the train is on the operational field of the simulator or not.

> Note: The three components aren't written in train registry. Rather Dispatcher checks the schedules and make it active or let it stay on scheduled. Passed train is when the train completes the journey, not the scheduled journey time.

### Signals:

Contains all signals on the operational area of the simulator. Controls train movement authority by displaying signal aspects based on track occupancy, junction state, dispatcher commands, and operational rules. Signals prevent conflicting train movements and enforce safe separation between trains.

> Note: There will be a `json` file on the locations of signal and `.py` file on the operations of signal
### Dispatcher: 

Coordinates all operational decisions within the simulator. The Dispatcher manages train routing, movement authority requests, conflict resolution, route reservation, train prioritization, and scheduled departures. It communicates with the Train Registry, Signals, Track Blocks, Stations, and Junctions to maintain safe and efficient railway operations.

`Dispatcher` reads`train_registry's` data, compares scheduled times against `Clock.current_time()` (the time-derived logic from before), and is the thing that actually **flags and holds** which trains are currently active. So the in-memory "currently active trains" collection lives on/in `Dispatcher`, not as a separate registry object.


### Simulation flow:

`simulation.py` calls the `dispatcher` for the number of scheduled trains. Unless the `dispatcher` returns zero scheduled trains and active train the simulation continues. 
``` text
Simulation Engine

↓

Clock.tick()

↓

Dispatcher.update()

↓

Event Engine.process()

↓

Signals.update()

↓

for every active train

↓

Train.movement()

↓

Occupancy.update()

↓

Metrics.update()

↓

Visualization
```


> Note: Max loop ceiling will exists to avoid infinite looping.



## Module dependency order:

Note: This does not include the existing modules.


| Module         | Dependency                                       |
| -------------- | ------------------------------------------------ |
| Clock          | None                                             |
| Train registry | None                                             |
| Track Block    | tracks.py (existing)                             |
| Event Engine   | Clock,Signal,Dispatcher                          |
| Station        | Track Block                                      |
| Junction       | Track Block                                      |
| Signal         | Track Block, Junction, Station                   |
| Dispatcher     | Train registry, clock, station, signal, junction |


### Dispatcher Position:


Dispatcher is more like train controller, but instead of calculating momentum and for a single train. It decides which train goes where for multiple trains. It uses train registry to get the locations a train is supposed to go and coordinates the multiple trains so there won't be any conflicts. Dispatcher shall later evolved to add a feature to be a human dispatcher control board, so human dispatchers can simulate their decision. 

This module sits between the simulation.py and train.py. 

Simulation calls the Dispatcher --> Dispatcher checks the schedule and current positions of the train --> Dispatcher resolves if any conflict arises--> Dispatcher decides the priority and route selection --> Dispatcher separates every train and feed it to train.py --> train.py gets the movement for each train.


### Clock:


Clock follows the IST zone. This will make it easy to follow and for train train handovers. 

There will be a fast forward, pause and rewind option. But that is for the next phase.


### File components:

#### Track Block:

`json File`:

``` text
track_block_start
track_block_end
track_block_type 
track_block_track_number
track_length
```

`.py File`:

``` python
occupied_by_train  
  
is_occupied  
  
reserved_by_train
```



#### Station:

`json File`:

``` text
station_name
station_code
station_type (terminus/terminal, central, hill, junction)

station_coordinated_x
station_coordinated_y 
station_coordinated_z (elevation)

number_of_platforms
number_of_tracks

can_train_reverse
can_train_overshoot_long (false for terminals, emergency brakes)

connected_track_blocks

```


`.py File`:

``` python
occupied_platforms

waiting_trains

arrived_trains

departed_trains

platform_status
```

Methods:

``` python
assign_platform()

release_platform()

arrival()

departure()
```


#### Junctions:

`json file`:

``` text
junction_id

junction_name

junction_type
    - simple
    - crossover
    - diamond
    - wye
    - scissors

coordinates
    x
    y
    z

incoming_tracks

outgoing_tracks

connected_track_blocks

connected_signals

default_route

maximum_speed_kmh

switch_count
```

`.py File`:

``` python
Data
-----
current_route
is_locked

Methods
-------
set_route()
lock()
unlock()
is_available()
```

#### Signals

`json File`:

```text 

signal_id

signal_name

signal_type
    (home, starter, distant, shunt)

signal_direction

signal_location

protected_track_block

associated_junction (optional)

default_aspect

maximum_authorized_speed_kmh (optional)
```

``` python
current_aspect  
  
previous_aspect  
  
last_change_time  
  
is_override_enabled

Methode:

update_aspect()

get_current_aspect()

has_aspect_changed()

reset()
```

#### Train Registry:

`json File`:

``` text
train_id
train_name
train_loco_type
train_type (express,local,premimum, ARME, goods)
emergency_mission_enabled (when ARME is chosen)
traction_type (ELECTRIC, DIESEL,HYDROGEN,STEAM,DUAL)


total_number_of_rolling_stocks
number_of_general_coach
number_of_1_tier_ac_coach
number_of_2_tier_ac_coach
number_of_3_tier_ac_coach

mass_of_train_kg
train_length_m
max_speed_of_train_ms
max_deceleration_of_train_ms

scheduled_dep_time
scheduled_arr_time
origin_station
destination_station

route_id
```
## Plan to Execute:

> Note: For this phase only 10 trains will be simulated. Not 50, not 500.

## STEP 1 — Create the infrastructure modules

Create the modules with only their interfaces, validations, and basic data structures.

- Clock
- Event Engine
- Track Block
- Station
- Junction
- Signal
- Dispatcher

Don't integrate them yet. Just make sure each module is self-contained.

---

## STEP 2 — Refactor the simulation core

Refactor of modules with seams which are stated above. The order of doing that depends on the severity of the seam. High severity gets first priority.

Priority:

1. Simulation loop
2. Per-train history
3. Train identity
4. Main initialization
5. Accuracy metrics
6. Performance improvements

At the end of this step, the simulator should still work exactly as before—but its architecture should be ready for multiple trains.

---

## STEP 3 — Implement the Train Registry

Now introduce the registry. A separate to contain all trains of the operational range.

Once it exists, replace:

```
self.train
```

with something conceptually like:

```
active_trains
```

This is where the simulator truly becomes multi-train.

---

## STEP 4 — Wire everything together


Now connect:

- Dispatcher ↔ Train Registry
- Signals ↔ Track Blocks
- Stations ↔ Dispatcher
- Junctions ↔ Signals
- Event Engine ↔ Clock
- Simulation ↔ All modules

Only after this point should any module begin influencing another.

---

## STEP 5 — Testing

Testing. Tons and tons of testing and debugging.

```
Stage 1
Single train(Regression)
↓
Stage 2
2 trains
↓
Stage 3
5 trains
↓
Stage 4
10 trains
↓
Stage 5
Stress tests
↓
Stage 6
Failure scenarios
```

This way, if something breaks at 5 trains, you know the issue is in scaling rather than in the fundamental architecture.

---