# Bellezza Bellona: First-Principles I/O Model and Porting Plan (for Gaggiuino-class control)

## Purpose

This document builds a **grounded system model** for the Bellezza Bellona from publicly available evidence, then turns that model into an implementation plan for a high-control retrofit (Gaggiuino-style behavior) using this repository as a reference, not a constraint.

---

## 1) Evidence base used

## 1.1 Manufacturer / product-level specs
- Bellezza Bellona official page:
  - https://bellezzaespresso.com/espresso-machines/bellona
- Published machine-level points extracted:
  - Dual boiler architecture
  - Silent vibration pump (~40 l/h stated)
  - Rotary valve user control
  - PID control
  - Boiler volumes (500 ml brew + 1 L steam)
  - 220/120 V, 50/60 Hz, 2000 W shown on page

## 1.2 Service/parts data (critical for internals)
- 1st-line Bellona parts diagram (Version 1 scope):
  - https://www.1st-line.com/technical-support/bellezza-technical-support/parts-diagram-bellezza-bellona/
  - Marked: `COD. 9942072 REV. 00 ed.08/21`
  - 10 exploded assemblies + callout/SKU tables.

## 1.3 1st-line Bellona product pages (component semantics)
- Power board 110V: `BE.004.002`
  - https://www.1st-line.com/buy/bellezza-be-004-002-bellona-power-board-110v/
- Steam boiler assembly V1: `BE.040.012`
  - https://www.1st-line.com/buy/bellezza-be-040-012-bellona-steam-boiler-assembly/
- Coffee boiler heating element: `BE.090.013` (1000W 120V)
  - https://www.1st-line.com/buy/bellezza-be-090-013-bellona-coffee-boiler-heating-element/
- Steam boiler heating element UL (V2 context): `96.05889(UL)`
  - https://www.1st-line.com/buy/bellezza-96-05889ul-bellona-steam-boiler-heating-element-ul/
- Pump pressure gauge: `BE.001.022`
  - https://www.1st-line.com/buy/bellezza-be-001-022-bellona-coffee-meter/
- Touch pad board: `95.05486`
  - https://www.1st-line.com/buy/bellezza-95-05486-bellona-touch-pad-board/
- Version 2 display board: `01.07728`
  - https://www.1st-line.com/buy/bellezza-01-07728-bellona-version-2-display-board/

---

## 2) Versioning reality (important)

The Bellona appears to have at least **Version 1** and **Version 2** electrical/assembly differences:
- Parts-diagram page is explicitly **Version 1**.
- `BE.004.002` power board page is 110V board with a "5 terminals" compatibility check.
- Steam boiler page states V1 assembly and references different V2 heating-element replacement path.
- Touch/display boards indicate UI/electronics differences between versions.

**Planning implication:** do not finalize retrofit architecture until your machine is version-identified by board/assembly photos and part numbers.

---

## 3) Confirmed component mapping to exploded callouts (V1 diagram)

From the 1st-line parts-diagram SKU tables, these mappings are directly resolvable:

- Callout **48** -> `BE.004.002` -> Bellona Power Board 110V
- Callout **74** -> `BE.040.012` -> Bellona Steam Boiler Assembly (V1)
- Callout **90.13** -> `BE.090.013` -> Coffee Boiler Heating Element 1000W 120V
- Callout **90.10** -> `BE.090.010` -> Dispersion shower screen
- Callout **24** -> `BE.001.022` -> Pump pressure gauge
- Callout **25** -> `BE.001.023` -> Coffee meter/gauge mount
- Callout **22** -> `BE.001.020` -> Version 1 display film
- Callout **33** -> `BE.003.007` -> Drip tray grid (stainless)
- Callout **34** -> `BE.003.008` / `BE.003.008W` variant family
- Callout **60** -> `BE.360.360` -> grouped mechanical sub-assembly root
- Callout **70** -> `BE.470.001` -> valve sub-assembly root
- Callouts **103..106** -> `BE.005.006..009` -> water tank/conduit components

Notes:
- Several IN/ZB-prefixed SKUs are present in diagram tables but have no directly exposed Bellona product-page metadata; treat as unresolved until manual/service docs or teardown confirm function.

---

## 4) What is now known about sensors and protective chain

## 4.1 Confirmed from parts descriptions
- V1 steam boiler assembly includes:
  - PID sensor probe
  - boiler refill probe
  - insulation
  - fittings
  - two thermostats, one high-limit resettable safety thermostat
- This is strong evidence that Bellona uses **multi-layer thermal safety** (control probe + high-limit device) and level/refill protection for steam circuit.

## 4.2 Confirmed from machine/product data
- Pump pressure gauge exists (mechanical readout to operator).
- Dual boiler with separate brew/steam thermal domains.
- Vibration pump architecture.

## 4.3 Still unresolved without direct inspection
- Exact probe electrical type per circuit (NTC/PT/PT1000/K-type, etc.).
- Exact refill probe interface type and control thresholds.
- Exact control-board I/O pinout and switching stage implementation.
- Exact solenoid and pump drive topology in your specific unit revision.

---

## 5) First-principles model: complete input -> compute -> output flow

## 5.1 Inputs to the machine

### User inputs
1. Main power switch
2. Brew activation input
3. Steam / hot water mode inputs
4. Rotary valve / flow path mechanical control
5. UI inputs (touch pad / display-board controls, version-dependent)

### Sensor and state inputs
1. Brew boiler temperature probe
2. Steam boiler PID temperature probe
3. Steam boiler refill/level probe
4. Safety thermostat(s) / high-limit thermostat state
5. Pump-line pressure indication (at minimum via mechanical gauge; electrical transducer unknown in stock config)
6. Tank water availability / low-water indication chain

### Electrical health/environmental inputs
1. AC mains presence/quality
2. Protective devices (thermal cutouts, fuses, resettable safety thermostat)

---

## 5.2 Compute / decision layers (stock architecture abstraction)

1. **UI/control logic layer**
   - Touch/display + user intent decoding
2. **Power/control board layer**
   - Command arbitration + actuator drive enable logic
3. **PID thermal control loop(s)**
   - Compare setpoint vs measured temperature
   - Generate heater drive for brew and steam domains
4. **Safety interlock layer**
   - Overrides commands if over-temp/no-water/fault conditions

---

## 5.3 Outputs from the machine

1. Brew boiler heater drive
2. Steam boiler heater drive
3. Pump drive
4. Solenoid/valve electrical actuation (where applicable)
5. User-facing indicators/display states
6. Steam/hot-water physical outputs (fluid/thermal process outputs)

---

## 6) Control interaction model (PID + safety + hydraulics)

At a first-principles level:
- PID loop regulates boiler temperature by modulating heater power.
- Water-level/refill probing prevents dry heating and triggers refill behavior or lockout.
- High-limit thermostat provides an independent hard safety cutoff path.
- Pump creates pressure/flow; hydraulic resistance (coffee puck, valves) determines extraction pressure profile.
- Operator valve state and brew command determine whether fluid path is extraction, vent, or steam/hot-water service mode.

For Gaggiuino-like behavior, you need explicit instrumentation and deterministic control over all these states (not only temperature).

---

## 7) Gap analysis vs Gaggiuino-class goals

To achieve advanced profile control and telemetry comparable to Gaggiuino-style systems, stock Bellona architecture is typically missing or opaque in these areas:

1. Direct, high-rate pressure transducer input (vs only mechanical gauge)
2. Structured flow/weight feedback integration
3. Open, programmable state machine for preinfusion/declining-pressure/flow targets
4. Rich event/fault telemetry export
5. Explicit actuator abstraction layer for version-to-version part differences

---

## 8) New planning doc: implementation strategy from this understanding

## Phase A — Reverse-engineer your exact machine revision
1. Photograph internals, all connectors, all boards, all probes, all thermostats.
2. Record board markings (e.g., power board revision, display/touch board revision).
3. Build a **callout-linked wire map** using the V1 exploded diagram indexes where applicable.
4. Verify if your unit matches V1 mapping (`BE.004.002` etc.) or mixed V1/V2 lineage.

## Phase B — Build an explicit I/O abstraction for retrofit
Define interfaces before coding/wiring:
- Inputs: `TempBrew`, `TempSteam`, `LevelSteam`, `UserBrew`, `UserSteam`, `PressureLine(optional)`
- Outputs: `HeaterBrew`, `HeaterSteam`, `Pump`, `ValveBrew`, `ValveSteam/HotWater`, `UIStatus`
- Safety lines: `HighLimitTrip`, `NoWater`, `SensorFault`

## Phase C — Introduce deterministic safety supervisor (hard requirements)
1. Any of: sensor fault, no-water, watchdog expiry -> heaters OFF, pump OFF.
2. High-limit thermostat remains in-circuit independent of software.
3. Power-up default state = all controlled outputs OFF.
4. Add startup interlock: no brew/steam until probes are plausible and level is valid.

## Phase D — Control loop architecture for advanced behavior
1. Separate control loops for brew boiler and steam boiler.
2. Add profile engine:
   - preinfusion phase
   - transition phase
   - target pressure/flow phases
3. If no pressure transducer initially: implement time/power profile mode with fallback safety bounds.
4. Add tunable setpoint schedulers and anti-windup for thermal loops.

## Phase E — Verification ladder
1. Low-voltage bench simulation of input/output state machine.
2. Dry switching test with dummy loads.
3. Heater-only controlled thermal test.
4. Pump + valve pulse tests.
5. Integrated brew-mode test with failure injection (sensor disconnect, no-water, reboot under load).

---

## 9) Deliverables to produce next (concrete)

1. `bellona-r1-sensor-report.md`
   - Probe types, wiring, measured values, connector pinouts.
2. `bellona-r2-load-and-driver-sheet.csv`
   - Actuator voltages/currents/inrush and selected switching hardware.
3. `bellona-r3-wiring-map-vX.svg`
   - As-found and retrofit wiring with safety path highlighted.
4. `bellona-r4-interlock-matrix.md`
   - Fault -> required output state table.
5. `bellona-iomap-vX.json`
   - Machine I/O abstraction binding to hardware pins/channels.

---

## 10) Constraints and confidence labeling

- **High confidence**: component identities and many part roles where explicit product pages exist.
- **Medium confidence**: subsystem boundaries inferred from exploded assemblies and SKU families.
- **Low confidence until measured**: exact sensor electrical protocols, board-level pinout, and full internal control logic behavior per revision.

This plan is intentionally designed so low-confidence assumptions are isolated and resolved early by measurement.
