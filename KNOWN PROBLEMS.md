
# KNOWN PROBLEMS

Last Updated: 2026-05-07

## 1. Lack of Real Railway Data

### Problem
The biggest blocker is the absence of usable real-world data for:
- train delays
- weather
- peak-hour traffic
- congestion
- station load
- signal failures
- maintenance events

### Impact
- delay predictor cannot be trained properly
- propagation algorithm cannot be validated realistically
- simulator inputs may be too synthetic

### Status
BLOCKED

### Notes
Synthetic data may be needed until better data is available.

---

## 2. Simulator Not Fully Defined

### Problem
The simulator concept exists, but the exact internal model is still incomplete.

### Impact
- AI training environment is not ready
- propagation logic has no test bed
- visualization scope may expand too early

### Status
IN DEVELOPMENT

---

## 3. AI Delay Predictor Cannot Start Properly Yet

### Problem
The model needs training data and clear feature definitions.

### Impact
- no reliable predictions
- model architecture may be guessed too early
- testing quality will be poor

### Status
PAUSED

---

## 4. AI Propagation Logic Depends on Other Systems

### Problem
Propagation requires:
- train schedule data
- route graph
- simulator
- delay model

### Impact
- cannot be completed independently
- needs structured system dependencies

### Status
PAUSED

---

## 5. Team Inactivity During Exams

### Problem
Both coding and non-coding teams are temporarily inactive due to exams.

### Impact
- slower execution
- delayed updates
- less coordination

### Status
TEMPORARY

---

## 6. Project Scope Is Very Large

### Problem
The project can easily grow beyond Phase 2 if every idea is treated as urgent.

### Impact
- confusion
- blocked execution
- feature overload
- poor prioritization

### Status
ONGOING RISK

### Rule
Only the current phase and its direct dependencies should be worked on first.

---

## 7. Feature Interdependence

### Problem
Many systems depend on each other:
- simulator depends on route model
- delay predictor depends on data
- propagation depends on simulator
- dashboard depends on outputs

### Impact
- one missing module can stall several others

### Status
STRUCTURAL RISK

---

## 8. No Final Data Standard Yet

### Problem
Data input format is not fully standardized.

### Impact
- messy dataset collection
- inconsistent training data
- hard AI preprocessing

### Status
OPEN

---

## 9. No Clear Test Benchmark

### Problem
There is no fixed benchmark to decide whether the simulator or AI modules are “good enough.”

### Impact
- hard to measure progress
- difficult to compare versions

### Status
OPEN

### Suggested Benchmark Types
- prediction error
- propagation accuracy
- simulation stability
- speed
- usability

---

## 10. Dashboard Can Become Too Complex Too Early

### Problem
The dashboard may try to show too much before the core engine is ready.

### Impact
- distraction from core work
- unnecessary UI work
- slower development

### Status
WATCHLIST

### Rule
The dashboard should follow the engine, not lead it.