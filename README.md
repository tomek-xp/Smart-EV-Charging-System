# Smart EV Charging (PV + Home Assistant + go-e + Bluelink)

## Overview

Home energy-aware EV charging system based on:
- Fronius (PV + grid data)
- go-e (charger control)
- Bluelink (vehicle SoC)
- Home Assistant (logic)

Goals:
- maximize PV self-consumption
- minimize grid usage
- protect battery health
- provide predictable daily readiness
- allow manual overrides

---

## Architecture

Fronius -> HA -> decision engine  
Bluelink -> HA -> SoC (low frequency)  
go-e -> HA -> execution (charging control)

---

## SoC Strategy

Hybrid approach:

- soc_real -> Bluelink (every 10–15 min)
- soc_est -> computed locally from go-e energy

soc_est = soc_anchor + (delta_kWh * efficiency / battery_capacity)

Anchor is updated on every Bluelink sync.

---

## Charging Modes

### 1. pv_surplus

- charge only from PV excess
- avoid grid import

available_power = export - buffer

- if below minimum -> stop
- otherwise adjust current dynamically

---

### 2. turbo

- max current
- ignore PV / battery / cost
- stop at target SoC

---

### 3. scheduled (safe + just-in-time)

Parameters:
- target_soc
- safe_soc (default: 80%)
- deadline

---

## Scheduled Logic

### Zone A: below safe_soc

- PV-first
- allowed to charge early
- no strict optimization

---

### Zone B: above safe_soc

- just-in-time charging
- do NOT charge early just because PV is available
- goal: reach target as late as possible

---

### Planning

missing_kWh  
hours_left  
required_power  
required_current  

---

### Start condition

If required current < minimum:

latest_start = deadline - (missing_kWh / min_power)

-> wait until latest_start

---

### Current rule

set_current = required_current

---

## Scheduling Model

### Daily Baseline

Default daily readiness target.

Example:
- Every day 07:00 -> 70%

Purpose:
- ensure daily usability
- minimize high SoC exposure

---

### Recurring Targets

Higher priority scheduled events.

Example:
- Monday 07:00 -> 80%
- Thursday 07:00 -> 80%

Purpose:
- ensure higher readiness on specific days

---

### One-time Override

Manual override for a specific time.

Example:
- Tomorrow 07:00 -> 100%

Purpose:
- trips / exceptional use

---

### Priority

turbo > one-time override > recurring > daily baseline

---

## Priorities (Energy Flow)

home -> EV -> battery -> grid

---

## Battery Rules

- 80% is NOT a hard limit
- key factor = time spent at high SoC

Ranges:

- 20–70% -> optimal
- 70–85% -> safe
- 85–95% -> moderate stress
- 95–100% -> avoid long duration

Rule:

Charging to 100% is fine  
Keeping it at 100% for hours is not

---

## Control Loop

- decision loop: every 10–15s
- Bluelink sync: every 10–15 min

---

## Anti-flicker

- start delay: ~60s
- stop delay: ~60–120s

---

## Key Principles

- control based on grid import/export, not raw PV
- use real energy (kWh), not planned values
- avoid frequent vehicle wakeups
- minimize time at high SoC

---

## TL;DR

- daily baseline ~70%
- PV to 80% aggressively
- above 80% just-in-time
- turbo when needed
- SoC = real + estimated
