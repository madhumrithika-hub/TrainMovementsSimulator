
# SYSTEM MAP

## Core Vision

Project-001 aims to become an intelligent railway operating ecosystem capable of:
- traffic management
- Automation
- scheduling optimization
- simulation
- safety monitoring
- resource management
- predictive systems

Eventually evolving into RailOS.

---

# ACTIVE SYSTEMS

## 1. Simulation Engine
Status: IN DEVELOPMENT

Purpose:
- Simulate railway operations
- Train AI models
- Verify propagation logic
- Visualize AI decisions

Dependencies:
- Railway topology
- Signal logic
- Event system

Connected Systems:
- AI Delay Predictor
- AI Propagation Engine
- Dashboard

---

## 2. AI Delay Predictor
Status: PAUSED

Purpose:
Predict train delays using:
- congestion
- weather
- route traffic
- peak-hour patterns

Dependencies:
- Real-world dataset
- Simulator
- Data preprocessing pipeline

Outputs:
- Delay estimations
- Risk probability

---

## 3. AI Propagation Engine
Status: PAUSED

Purpose:
Predict how one delay affects the entire network.

Dependencies:
- Delay predictor
- Route graph
- Train schedule data
- Simulator

Outputs:
- Cascading delay predictions
- Congestion alerts

---

## 4. Dashboard
Status: ACTIVE

Purpose:
Visualize:
- train movement
- AI decisions
- delay predictions
- simulation state

Dependencies:
- Backend API
- Simulation engine
- AI outputs

---

## 5. Data Collection System
Status: COMPLETED

Purpose:
Allow non-coding team to enter railway data.

Outputs:
- Training datasets
- Validation datasets

---

# FUTURE SYSTEMS

## Signal Optimization
## Predictive Maintenance
## Crew Management
## Passenger Information
## Resource Allocation
## Energy Monitoring

---

# MASTER DEPENDENCY FLOW

Data Collection
    ↓

Simulator
    ↓

Delay Predictor
    ↓

Propagation Engine
    ↓

Dashboard