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

## 1) Phase 1 BOM (baseline)

## 1.1 Controller + firmware platform

1. **ESP32-S3 controller board** (1x)
   - Prefer a board equivalent to the existing GaggiMate controller class (ESP32-S3, 3.3V GPIO, USB serial).
2. **Optional display board** (0–1x)
   - Not required for Phase 1 if you run controller-only bench tests.
3. **USB cable + isolated USB power source** (1x each)

## 1.2 Sensor front-end

4. **K-type thermocouple amplifier board (MAX31855-compatible)** (1x)
   - Match firmware expectation for thermocouple path used by controller code.
5. **K-type probe / thermocouple mounting adapter** (1x)
   - If reusing machine sensor is not electrically compatible, install dedicated probe point.

## 1.3 Power switching (mains side)

6. **Heater SSR (zero-cross AC SSR, correctly rated)** (1x)
   - Current rating with derating margin and proper heatsink if required.
7. **Pump switching device** (1x)
   - SSR or relay selected per pump type (vibration pump AC load).
8. **Solenoid valve switching device** (1x)
9. **Optional AUX relay channel hardware** (1x)
   - For future grinder/aux function parity.

## 1.4 Protection + wiring hardware

10. **DIN rail or insulated mounting plate** (1x)
11. **Enclosure / shielded electronics compartment** (1x)
12. **Terminal blocks (mains rated)** (as needed)
13. **Appropriate gauge wire**
    - HV: machine/load current rated
    - LV: signal/control wiring
14. **Inline fuses / fuse holders** (as needed)
15. **Ring/fork terminals, ferrules, spade terminals, heatshrink** (assorted)
16. **Cable glands + strain relief** (as needed)
17. **Ground bonding hardware** (as needed)

## 1.5 Commissioning/test tools

18. **True-RMS multimeter** (1x)
19. **Clamp meter** (1x recommended)
20. **Insulation resistance tester** (recommended)
21. **Bench lamp/load for dry-run output testing** (1x)
22. **Notebook for wire map + test logs** (1x)

---

## 2) Pre-build design package (do this before touching machine wiring)

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

## 3) Detailed implementation steps (Phase 1)

## Step A — Bench harness (no mains loads connected yet)

1. Flash controller firmware and confirm serial boot.
2. Wire only low-voltage side:
   - ESP32 + thermocouple amp + button inputs.
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
3. Validate thermocouple readings against expected room/heat trend.
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

## 4) Firmware tasks to pair with Phase 1 hardware

1. Add a dedicated Bellona board profile in controller config (pins + capabilities).
2. Start with conservative capability flags:
   - pressure=false, dimming=false unless proven hardware exists.
3. Ensure default-safe output polarity for each actuator channel.
4. Keep watchdog timeout and thermal fault paths enabled and tested.

---

## 5) Suggested minimum acceptance checklist

- [ ] All wiring continuity and grounding checks pass.
- [ ] No actuator energizes at boot.
- [ ] Heater shuts down on timeout and sensor fault.
- [ ] Pump and solenoid execute short command pulses reliably.
- [ ] 10 repeated power cycles pass without unsafe behavior.
- [ ] Short heat test completed without unstable oscillation.

---

## 6) What is intentionally out of Phase 1

- Pressure profiling and advanced flow estimation.
- Mechanical enclosure polish / final cable routing cosmetics.
- Long-duration thermal characterization and PID optimization.
- Product-grade compliance documentation.

---

## 7) Procurement tips

- Buy SSR/relay components from reputable suppliers and verify authentic ratings.
- Over-spec switching components with realistic thermal derating.
- Keep spare modules/fuses/terminals so troubleshooting does not stall integration.
- Label every wire before disconnecting any factory harness.
