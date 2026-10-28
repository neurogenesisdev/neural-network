---
type: area
status: active
created: '2026-04-21T06:26:03Z'
updated: '2026-04-21T06:26:03Z'
tags: [personal, health, cardiovascular, heart-rate, vo2-max, blood-pressure, longitudinal]
---

# Cardiovascular Health — longitudinal log

Log of cardiovascular-health metrics captured over time. Append new measurements as new `## Entry — <date>` sections. Zone formulas in the Reference section below should be recomputed per entry using age-at-measurement, since max-HR drifts down as age rises.

## Reference formulas

Theoretical maximum heart rate:

$$\text{Max HR} = 220 - \text{age}$$

### Zone distribution (% of max HR)

| Zone | Name | Range | Purpose |
|---|---|---|---|
| 1 | Active recovery / warmup | 50 – 60 % | Recovery, warmup, cooldown |
| 2 | Aerobic base / endurance | 60 – 70 % | Fat oxidation, mitochondrial density |
| 3 | Tempo | 70 – 80 % | Aerobic capacity, sustainable pace |
| 4 | Lactate threshold | 80 – 90 % | Lactate clearance, race pace |
| 5 | VO2 max | 90 – 100 % | Peak anaerobic capacity |

### Other metrics to track

- **VO2 max** (ml / kg / min) — direct measure of aerobic capacity; target depends on age + sex
- **Resting heart rate** (bpm) — lower typically indicates better cardiovascular fitness
- **Blood pressure** (systolic / diastolic, mmHg) — ideal < 120 / 80

## Entry — ~2023 (age ~27, estimated)

Legacy data from the original `Cardiovascular Health.md` before the rewrite to longitudinal format. Only Zones 2 and 5 were captured. Other metrics not recorded.

| Metric | Value |
|---|---|
| Age | ~27 (estimated) |
| Theoretical max HR | 193 bpm (220 − 27) |
| Empirical max HR | not recorded |
| Zone 1 | not recorded |
| Zone 2 | 116 – 135 bpm |
| Zone 3 | not recorded |
| Zone 4 | not recorded |
| Zone 5 | 174 – 193 bpm |
| VO2 max | not recorded |
| Resting HR | not recorded |
| Blood pressure | not recorded |

## Entry template (copy to start a new entry)

```
## Entry — YYYY-MM-DD

| Metric | Value |
|---|---|
| Age | <> |
| Theoretical max HR | <220 − age> |
| Empirical max HR | <measured, if known> |
| Zone 1 | <50–60% of max HR> |
| Zone 2 | <60–70%> |
| Zone 3 | <70–80%> |
| Zone 4 | <80–90%> |
| Zone 5 | <90–100%> |
| VO2 max | <ml/kg/min> |
| Resting HR | <bpm> |
| Blood pressure | <systolic / diastolic> |

**Context**: <any relevant lifestyle / training context at time of measurement>
```
