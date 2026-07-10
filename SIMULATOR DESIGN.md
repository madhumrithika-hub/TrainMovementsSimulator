
# SIMULATOR DESIGN

Last Updated: 2026-05-07

## Purpose

The simulator is a controlled railway environment used to:
- test the AI delay predictor
- test the AI propagation algorithm
- generate synthetic training data
- visualize railway behavior for operators
- verify decisions before real-world use

This is a core system for Phase 2 of Project-001. 

---

## Main Goals

1. Simulate train movement across a railway network
2. Simulate delays, congestion, signal changes, and route conflicts
3. Provide a realistic environment for AI training
4. Show how a delay spreads through the network
5. Support future dashboard visualization

---

## Simulator Scope

### Included
- train locations
- station nodes
- route paths
- travel times
- delays
- congestion
- platform occupancy
- signal states
- basic event triggers

### Not Included Yet
- real hardware integration
- full passenger flow system
- maintenance system
- crew management
- energy monitoring
- live production railway control

---

## Core Components

### 1. Railway Network Model
Represents the railway system as:
- nodes = stations, junctions, depots
- edges = tracks/routes
- weights = travel time, delay risk, congestion level

### 2. Train Entity Model
Each train should have:
- train ID
- current station or track position
- departure time
- arrival time
- delay value
- route path
- priority level

### 3. Event Engine
Handles events such as:
- train departure
- train arrival
- delay occurrence
- route blockage
- signal change
- congestion buildup

### 4. Time Engine
Moves the simulation forward in fixed steps or event-based steps.

### 5. AI Feedback Layer
Produces data for:
- delay prediction
- propagation prediction
- route decisions

### 6. Visualization Layer
Shows:
- train movement
- route status
- delayed trains
- congestion zones
- AI decisions

---

## Simulation Flow

1. Load railway map
2. Load trains and schedules
3. Start time progression
4. Trigger events
5. Update train states
6. Calculate delays and propagation
7. Send outputs to dashboard and AI modules

---

## Data Needed

The simulator should support both:
- dummy data for early testing
- real data later for validation

Inputs may include:
- station list
- route graph
- scheduled times
- delay history
- weather impact
- congestion levels
- peak-hour patterns

---

## Output Data

The simulator should produce:
- current train positions
- delay history
- route conflicts
- propagation effects
- congestion maps
- event logs
- training data for AI
----
## CURRENTLY FINISHED:





---

## Priority Order

### Phase 2 Priority
1. basic network model
2. train movement
3. delay events
4. propagation logic
5. visualization

### Later Priority
1. reinforcement learning environment
2. advanced route optimization
3. operator-grade dashboard integration

---

## Success Criteria

The simulator is useful if it can:
- reproduce railway movement correctly
- generate repeatable test cases
- show delay spread clearly
- support AI training
- help debug the propagation engine