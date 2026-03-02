# Bellezza Bellona Phase 1 BOM and Integration Guide

> **Scope:** Phase 1 = electrical interface and bench bring-up only (no full production enclosure, no advanced pressure profiling).
>
> **Goal:** Safely control heater/pump/solenoid and read brew/steam switches + boiler temperature using an ESP32-based controller compatible with the GaggiMate software architecture.

## 0) Safety first (read before buying anything)

- You are working with **mains AC** and high-power loads.
- Use local code-compliant wiring, grounding, strain relief, fusing, and creepage/clearance practices.
- If you are not trained for mains work, use a licensed electrician/technician for final wiring verification.
- Perform initial tests on a current-limited protected circuit and always validate fail-safe "outputs off" behavior before thermal tests.

---

## 1) Mandatory Bellona research package (must complete before purchase/integration)

Before moving forward, complete and archive this machine-specific research. **Do not merge/execute the wiring plan until this section is complete.**

1. **Sensor inventory and electrical characteristics**
   - Identify each installed temperature/pressure/level sensor on the Bellona.
   - Record sensor type (NTC/PT100/PT1000/K-type/etc.), nominal resistance/output curve, wiring count, and isolation requirements.
   - Confirm whether the stock brew temperature probe can be read directly by MAX31855 path, needs an ADC front-end, or requires replacement.

2. **Actuator inventory and load characterization**
   - Heater: nominal voltage, wattage, expected current (cold and steady-state if available).
   - Pump: type (vibration/rotary), voltage, inrush/steady current.
   - Solenoid(s): coil voltage/current.
   - Existing PID/relay board behavior: active-high/active-low logic and fail-safe default state.

3. **Wiring and topology capture**
   - Produce a full wire map from mains entry to each load/switch/sensor.
   - Identify which paths are switched on line vs neutral side.
   - Verify grounding points (frame, boiler, cup-warmer, chassis).

4. **Controls and UX mapping**
   - Document all physical controls (brew, steam, hot water, etc.) and expected machine behavior.
   - Note whether any function is latched, momentary, or interlocked mechanically.

5. **Safety devices and constraints**
   - Locate thermal fuse, pressure safety valve, over-temp cutouts, and any stock safety interlocks.
   - Confirm which safety devices remain untouched and always in circuit after retrofit.

### 1.1 Research deliverables (required artifacts)

- **R1: Sensor compatibility report** (with explicit adapter/replacement decision per sensor).
- **R2: Load sizing sheet** (heater/pump/solenoid voltage/current + switching device selections).
- **R3: Bellona wiring diagram** (as-found and as-planned, with wire labels).
- **R4: Interlock/fail-safe matrix** (timeout, reboot, sensor fault, comms loss).
- **R5: Commissioning risk log** (unknowns, mitigations, owner, status).

If any R1–R5 artifact is missing, treat the project as **blocked**.


## 1.2 Bellona baseline research findings (completed from public sources)

The following data has been extracted from the official Bellezza Bellona product/specification page and should be treated as the current baseline:

- Machine type: **Dualboiler** (independent brew + steam circuits).
- Pump type: **Silent vibration**, approx. **40 l/h** flow.
- Valve: **Rotary valve**.
- Control: **PID control = Yes**.
- Water tank: **1.8 L**.
- Boiler material: **Stainless steel**.
- Boiler volume: **500 ml brew boiler + 1 L steam boiler**.
- Pump pressure gauge: **Yes**.
- Weight: **23 kg**.
- Supply voltage: **220 V / 120 V**.
- Frequency: **50/60 Hz**.
- Energy absorption: **2000 W**.

Source used:
- Official Bellona product page: https://bellezzaespresso.com/espresso-machines/bellona
- Bellona manuals listed by manufacturer:
  - https://bellezzaespresso.com/wp-content/uploads/Manual_Bellona_EN_GE_Australia_Saudi_Europe-EN.pdf
  - https://bellezzaespresso.com/wp-content/uploads/Manual_Bellona_US-EN.pdf

### 1.3 Research gaps still requiring on-machine validation

Public documentation does **not** conclusively expose all retrofit-critical internals. These must be measured directly on your unit before finalizing electrical integration:

- Exact brew/steam sensor technology and wiring (NTC/PT100/PT1000/thermocouple).
- Exact heater element resistance/current split between brew and steam circuits.
- Pump and solenoid startup current (inrush) and switch-side orientation in the stock harness.
- Existing safety chain wiring order (thermal fuse/over-temp cutout/pressure safety components).
- Switch logic polarity and whether any controls are latched/interlocked.

---

## 2) Phase 1 BOM (baseline)

## 2.1 Controller + firmware platform

1. **ESP32-S3 controller board** (1x)
   - Prefer a board equivalent to the existing GaggiMate controller class (ESP32-S3, 3.3V GPIO, USB serial).
2. **Optional display board** (0–1x)
   - Not required for Phase 1 if you run controller-only bench tests.
3. **USB cable + isolated USB power source** (1x each)

## 2.2 Sensor front-end

4. **K-type thermocouple amplifier board (MAX31855-compatible)** (1x)
   - Use only if R1 confirms K-type compatibility or a dedicated retrofit probe is installed.
5. **K-type probe / thermocouple mounting adapter** (1x)
   - If reusing machine sensor is not electrically compatible, install dedicated probe point.
6. **Alternative sensor front-end (ADC/RTD interface)** (0–1x)
   - Populate this instead of MAX31855 path if R1 determines stock Bellona sensor is NTC/RTD.

## 2.3 Power switching (mains side)

7. **Heater SSR (zero-cross AC SSR, correctly rated)** (1x)
   - Size from R2 load sheet with derating and thermal management.
8. **Pump switching device** (1x)
   - SSR or relay selected per pump type and measured current profile.
9. **Solenoid valve switching device** (1x)
10. **Optional AUX relay channel hardware** (1x)
   - For future grinder/aux function parity.

## 2.4 Protection + wiring hardware

11. **DIN rail or insulated mounting plate** (1x)
12. **Enclosure / shielded electronics compartment** (1x)
13. **Terminal blocks (mains rated)** (as needed)
14. **Appropriate gauge wire**
    - HV: machine/load current rated
    - LV: signal/control wiring
15. **Inline fuses / fuse holders** (as needed)
16. **Ring/fork terminals, ferrules, spade terminals, heatshrink** (assorted)
17. **Cable glands + strain relief** (as needed)
18. **Ground bonding hardware** (as needed)

## 2.5 Commissioning/test tools

19. **True-RMS multimeter** (1x)
20. **Clamp meter** (1x recommended)
21. **Insulation resistance tester** (recommended)
22. **Bench lamp/load for dry-run output testing** (1x)
23. **Notebook for wire map + test logs** (1x)

---

## 3) Pre-build design package (do this before touching machine wiring)

Create these artifacts first:

1. **Bellona wire map** (table)
   - Original wire, source, destination, voltage domain, expected current.
2. **I/O mapping table**
   - Heater control pin, pump control pin, valve control pin, brew switch input, steam switch input, thermocouple pins.
3. **Interlock matrix**
   - What must turn off on watchdog timeout, sensor fault, and reboot.
4. **Power-up state definition**
   - Explicitly define safe default output states (all actuators OFF).

---

## 4) Detailed implementation steps (Phase 1)

## Step A — Bench harness (no mains loads connected yet)

1. Flash controller firmware and confirm serial boot.
2. Wire only low-voltage side:
   - ESP32 + sensor front-end chosen by R1 + button inputs.
3. Replace actuator outputs with safe indicator loads (e.g., pilot lamps/LED opto boards).
4. Verify boot defaults keep all outputs off.
5. Trigger brew/steam inputs and confirm state updates over logs/UI.

**Pass criteria:** No spurious output toggles, stable sensor reading, deterministic switch behavior.

## Step B — Dry actuator switching (isolated from machine)

1. Connect SSR/relay modules to controller outputs.
2. Keep mains output side disconnected from machine loads.
3. Toggle each channel from control path and validate contact/output behavior with multimeter.
4. Power cycle repeatedly to verify no unsafe startup pulse.

**Pass criteria:** All channels switch correctly; restart/fault returns to OFF.

## Step C — Single-load integration: heater path

1. Integrate heater switching path through SSR with proper thermal management.
2. Keep pump/solenoid disconnected initially.
3. Validate temperature readings against expected room/heat trend.
4. Run conservative target temp and monitor overshoot.

**Pass criteria:** Controlled ramp, no runaway, timeout/fault safely de-energizes heater.

## Step D — Add pump and solenoid

1. Integrate pump switching channel.
2. Integrate solenoid channel.
3. Validate brew mode command sequence (valve + pump behavior) under short pulses.
4. Verify no relay chatter or unexpected simultaneous states.

**Pass criteria:** Repeatable on/off behavior and safe abort path.

## Step E — End-to-end safety validation

Run these forced-failure tests:

1. **BLE/display/control heartbeat loss** -> outputs OFF.
2. **Temperature sensor fault/disconnect** -> heater OFF.
3. **ESP32 reboot during active heat/pump** -> outputs OFF on reboot.
4. **Manual emergency stop procedure** verified.

Record results in a commissioning sheet.

---

## 5) Firmware tasks to pair with Phase 1 hardware

1. Add a dedicated Bellona board profile in controller config (pins + capabilities).
2. Start with conservative capability flags:
   - pressure=false, dimming=false unless proven hardware exists.
3. Ensure default-safe output polarity for each actuator channel.
4. Keep watchdog timeout and thermal fault paths enabled and tested.
5. If R1 indicates non-thermocouple sensing, add/enable matching sensor peripheral driver before mains testing.

---

## 6) Suggested minimum acceptance checklist

- [ ] R1–R5 research artifacts completed and reviewed.
- [ ] All wiring continuity and grounding checks pass.
- [ ] No actuator energizes at boot.
- [ ] Heater shuts down on timeout and sensor fault.
- [ ] Pump and solenoid execute short command pulses reliably.
- [ ] 10 repeated power cycles pass without unsafe behavior.
- [ ] Short heat test completed without unstable oscillation.

---

## 7) What is intentionally out of Phase 1

- Pressure profiling and advanced flow estimation.
- Mechanical enclosure polish / final cable routing cosmetics.
- Long-duration thermal characterization and PID optimization.
- Product-grade compliance documentation.

---

## 8) Procurement tips

- Buy SSR/relay components from reputable suppliers and verify authentic ratings.
- Over-spec switching components with realistic thermal derating.
- Keep spare modules/fuses/terminals so troubleshooting does not stall integration.
- Label every wire before disconnecting any factory harness.
