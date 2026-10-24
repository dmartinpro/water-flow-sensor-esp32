# CLAUDE.md

Guidance for Claude Code (and future me) when working in this repository.

## Project purpose

Monitor water flow in a swimming pool filtration installation using a **YF-DN40**
hall-effect flow sensor wired to an **ESP32**, and publish live readings to
**MQTT**. The MQTT data is used to:

1. Confirm the filtration pump is actually circulating water (dashboarding /
   alerting in Home Assistant or similar).
2. Detect abnormal conditions — most importantly **no flow while the pump is
   running**, which typically means the pump is dry-running (lost prime, closed
   valve, clogged strainer/filter) and at risk of damage.
3. Potentially trigger a **safety cutoff of the pump** when a no-flow condition
   is detected, either directly from the ESP32 or via an automation downstream
   of MQTT (exact split of responsibility is a design decision — see below).

Top priority: **easy to configure**, **robust** (runs unattended outdoors/in a
pool equipment room), and safe with respect to mains-voltage pump control.

## Status

Phase 1 (monitoring/reporting only) is implemented. Architecture decisions,
validated with the user:

- **Firmware**: ESPHome (`esphome/water-flow-sensor.yaml`), config validated
  with the actual `esphome config` CLI (2026.7.4) — passes.
- **Pump cutoff**: report-only via MQTT for now; no relay/pump control in
  this phase. The user's pump has an existing controller/contactor with a
  low-voltage interlock input, which is the intended integration point for a
  *future* automatic-cutoff phase (see README "Future enhancements") — not
  built yet.
- **MQTT**: standalone broker (Mosquitto), no Home Assistant. Discovery is
  off; topics are plain and documented in `README.md`.
- **Power**: 12V/24V DC already available at the equipment pad → buck
  converter to 5V for the ESP32; sensor powered directly from the raw
  12/24V rail (within its 3.5–24V spec).
- **Board**: ESP32-DevKitC (or pin-compatible dev board). Pulse input on
  GPIO4 with an external 4.7–10 kΩ pull-up to 3V3 (sensor output is NPN
  open-collector).

`README.md` is the source of truth for the full BOM, wiring diagram
(`docs/wiring-diagram.svg`), MQTT topic list, and setup instructions — keep
it in sync with any change to `esphome/water-flow-sensor.yaml`.

## Hardware facts (confirmed from manufacturer datasheets)

**YF-DN40 flow sensor**
- Thread: G1.5" (47 mm male thread), inline turbine + hall-effect pickup.
- Flow range: 5–150 L/min, accuracy ±2% (1–120 L/min).
- Supply voltage: DC 3.5–24V (some datasheets state 5–18V — confirm against
  the actual unit in hand before finalizing the power design).
- Output: **NPN open-collector pulse**, idle high, pulled low on each pulse.
  Max sink current ≤10 mA at 5V → needs an external pull-up resistor to logic
  level (safe with e.g. 4.7 kΩ–10 kΩ to 3.3V, well under the current limit).
- Pulse factor: **F(Hz) = 0.45 × Q(L/min)**, which is a constant
  **27 pulses per liter** regardless of flow rate. Useful conversions:
  - Flow rate: `Q(L/min) = F(Hz) / 0.45` (i.e. `F × 2.222`)
  - Volume: `1 pulse ≈ 0.037 L` (`1 / 27 L`)
- Expected pulse frequency at 5–150 L/min: ~2.25–67.5 Hz — low enough that
  ESP32 hardware pulse counting (PCNT) or even interrupt counting is
  comfortably accurate; no exotic timing constraints.
- Pressure rating ≤1.75 MPa, operating temp ≤80°C.

**Target MCU**: classic ESP32 (ESP32-D0WDQ6, 4 MB flash, micro-USB with a
CP2102 USB-UART chip). The user first planned an ESP32-S3, but the board
actually connected is a classic ESP32, so the firmware targets `esp32dev`.
The classic ESP32 also has a PCNT peripheral suitable for accurate pulse
counting. GPIO4 is a plain I/O pin on both chips, so wiring is unchanged.

## Design considerations to keep in mind

- **Fail-safe pump cutoff**: if a relay/SSR is used to interrupt the pump
  (or its contactor coil), prefer wiring so that loss of ESP32 power also
  stops the pump (i.e. relay energized = "flow OK, pump allowed to run";
  de-energized on power loss or fault = pump stops). Never assume "no flow
  detected" is the only fault condition worth defaulting safe against.
- **Mains-voltage safety**: pump circuits are typically 120/230V AC (or
  3-phase for larger pumps). Prefer switching a low-voltage/low-current
  contactor coil or existing pump-controller interlock input over switching
  pump mains power directly. Flag any mains wiring work as something the
  user should have reviewed/done by a qualified electrician if there's any
  doubt — do not casually hand-wave this.
- **Environment**: pool equipment rooms are humid/wet. Any enclosure and
  connector guidance in the README should assume that (IP-rated enclosure,
  ferrules/terminal blocks, no exposed mains, sensor cable strain relief).
- **Signal integrity**: NPN open-collector output is fine with a simple
  pull-up to 3.3V; keep sensor cable runs reasonably short or shielded if
  routed near pump motor/contactor wiring to avoid noise on the pulse line.
- **MQTT contract**: keep topic names and payloads stable once chosen, since
  they'll be consumed by Home Assistant / dashboards / automations outside
  this repo. Document the topic list in the README once decided (flow rate,
  cumulative volume, status/alarm, availability/LWT).

## Working conventions for this repo

- This is a small embedded/IoT project — avoid over-engineering. No need for
  elaborate abstractions; a single well-commented ESPHome YAML (or a compact
  firmware sketch, depending on the chosen stack) is appropriate.
- Document *why*, not *what*, in comments — especially around the pulse
  factor conversion and any fail-safe wiring decisions, since those are easy
  to get subtly wrong on a future edit.
- Keep `README.md` as the source of truth for: architecture decision, full
  bill of materials, wiring diagram, MQTT topic list, and setup/flashing
  instructions. Update it alongside any firmware change that affects wiring,
  topics, or configuration.
- Don't invent additional sensors/features (pressure, temperature, pump
  current sensing, etc.) unless the user asks — scope is flow monitoring and
  optional flow-based pump safety cutoff.
- **Always ask before running `git commit`** (or anything that creates a
  commit), even if the changes were just made at the user's request. Wait
  for explicit confirmation each time rather than committing proactively.
