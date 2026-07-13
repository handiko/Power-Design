# README — Power Subsystem, EC200U-Based Telematics Tracker

## 1. Scope

This package covers the power-path and protection architecture only, from the vehicle/battery input connector through to the two regulated rails feeding the MCU and the Quectel EC200U cellular modem. It intentionally excludes MCU pinout, RF matching, and GNSS antenna path, per the assessment scope.

## 2. Assumptions and Constraints

These were fixed first, before any topology or component decision, because every choice downstream depends on them.

| # | Assumption | Basis |
|---|---|---|
| A1 | Primary power source is vehicle-side, wide-range input: nominal 12 V or 24 V systems, with cranking dips and load-dump transients per ISO 7637-2 | Stated target: "vehicle-powered tracking device" |
| A2 | A battery-powered variant is a secondary case, sharing the same rails downstream of the reverse-polarity/eFuse stage, differing only in the input connector and TVS clamp voltage | Assessment allows either source type |
| A3 | Modem is Quectel EC200U-class (Cat-1/Cat-1bis with 2G fallback retained) | Stated in this exercise |
| A4 | Modem supply must never drop below 3.3 V, and must be able to source at least 3.0 A peak to cover the 2G-fallback case, not just the 2.0 A LTE-only case | Per Quectel EC200U hardware design guide |
| A5 | MCU + GNSS + peripheral rail peak current is 150 mA, average current is ~60 mA | Engineering assumption for a typical Cortex-M class MCU + GNSS receiver; not vendor-sourced — flagged as an assumption to revisit once real parts are selected |
| A6 | Device is a sealed, vehicle-mounted enclosure — thermal dissipation from any continuous-loss component (e.g. a series diode) is a real constraint, not just an efficiency nicety | Product form factor |
| A7 | Product is asset-tracking hardware, not a safety-critical ECU — margin policy and protection scope are set accordingly (25% current margin, not automotive-ECU-grade 30%+, and no supercap holdup stage) | Stated design philosophy, see Section 6 |

## 3. High-Level Power Architecture

```mermaid
flowchart TB
    A["Input Connector<br/>12/24V nominal"] --> B["Bidirectional TVS +<br/>CM Choke + EMI Filter"]
    B --> C["Ideal-Diode + FET<br/>Reverse polarity"]
    C --> D["eFuse<br/>Overcurrent limit"]
    D --> E["Pre-regulator Buck<br/>+5V intermediate rail"]
    D -. PGOOD, enable .-> E
    E --> F["+3V3 Buck<br/>MCU + digital rail"]
    E --> G["+3V8 Buck<br/>EC200U VBAT rail"]
    F --> H["Supervisor IC<br/>UVLO + reset"]
    H -. enable, delayed .-> G
```

Reading order: energy is progressively "narrowed" — from an uncontrolled, wide-range, bidirectional-fault-capable input down to two clean, narrow-tolerance rails. Each stage exists to remove one specific degree of freedom (transient amplitude, polarity, current magnitude, voltage tolerance) before the next stage has to deal with it.

Two additions relative to earlier revisions of this diagram, both carried through into `POWER_DESIGN.md`:

- **The TVS is bidirectional, not unidirectional**, and is followed by a common-mode choke with differential filter capacitors across its terminals. The bidirectional clamp addresses negative-going transients (e.g. ISO 7637-2 pulses 1 and 2a/2b) that a unidirectional part would pass through to the reverse-polarity stage unclamped; the choke + capacitor combination is a first-pass conducted-EMI filter, not a redundant surge stage — surge energy is still the TVS's job.
- **The eFuse's PGOOD output gates the pre-regulator's enable pin.** This is a sequencing addition, not just a protection one — see assumption S0 in Section 4.

## 4. Startup Sequencing Assumptions

The architecture relies on the following sequencing behavior being true; each assumption is stated explicitly because the design would fail its purpose if any one of them were violated.

| # | Sequencing assumption | Why it has to hold |
|---|---|---|
| S0 | The eFuse's PGOOD output gates the pre-regulator's enable pin — the pre-regulator is not allowed to start switching until the eFuse reports no fault (no overcurrent/overvoltage condition, FET fully on) | Without this gate, the pre-regulator could start ramping into an already-faulted protection stage (e.g. mid-inrush, or during an eFuse current-limit event) instead of a clean, current-limited-good input. This pushes the "is the input actually healthy" decision to the one component whose job that is, rather than leaving the pre-regulator to find out the hard way |
| S1 | +5V (pre-regulator) reaches regulation before either downstream buck is enabled | Downstream bucks are not specified to operate correctly from a ramping, not-yet-regulated input |
| S2 | +3V3 reaches regulation and the supervisor releases MCU reset **before** the modem enable/PWRKEY line is allowed to assert | The MCU must be alive and able to arbitrate a fault before the noisiest, highest-current rail in the system turns on |
| S3 | The supervisor's release delay is longer than +3V3's own worst-case power-up ramp time (across the full input voltage range in A1) | A delay shorter than the ramp time would let the modem enable race the MCU rail instead of waiting for it |
| S4 | Once released, modem enable is a one-way gate for that power cycle — it does not re-trigger on minor +3V3 ripple | Without this, a full reset-and-resequence could be triggered by common-mode noise rather than an actual undervoltage event, which is a self-inflicted reset loop distinct from the modem-collapse case in Section 6 below |
| S5 | No rail relies on firmware to complete its own sequencing | Firmware cannot run until +3V3 is already stable, so any rail whose sequencing depended on firmware would have a circular dependency on itself |

These are assumptions the schematic and component selection must satisfy, not behaviors that emerge automatically from picking "a supervisor IC" — S3 in particular has to be checked against the specific supervisor's programmable delay range and the specific buck's startup time once real parts are chosen.

## 5. Rail Dependency

Stating explicitly which failures are total and which are recoverable was one of the first decisions made, because it drives the protection and sequencing choices elsewhere in this design rather than being a consequence of them.

| Rail | If this rail fails... | Consequence | Recoverable without a full power cycle? |
|---|---|---|---|
| VIN / protection stage | System has no input at all | Total system failure | N/A — no power, nothing to recover |
| +5V (pre-regulator) | Both downstream rails lose their source | Total system failure — both MCU and modem go down together | No |
| +3V3 (MCU) | MCU cannot run; nothing can arbitrate faults, log data, or re-enable the modem | Total system failure by definition — this rail is not allowed to degrade gracefully, only to be protected as strongly as possible upstream (dedicated supervisor, independent from modem rail) | No |
| +3V8 (modem) | Modem browns out, resets, or fails to attach | **Degraded, not total.** MCU stays alive, GNSS logging continues, data buffers to local storage; firmware retries modem power-up on a backoff timer | Yes — this is the one rail this architecture is explicitly designed to let fail without taking the system down |

This table is the reason the modem rail gets its own regulator, its own bulk capacitance strategy, and a firmware-level retry expectation, while the MCU rail gets a dedicated hardware supervisor and no fallback path at all — the two rails are protected to different standards on purpose, because they fail differently.

## 6. Design Rationale and Trade-offs (summary)

Full reasoning lives in `POWER_DESIGN.md`; this is the short version for orientation.

- **Two-stage regulation (wide-in pre-reg → two narrow-in bucks), not one wide-in buck per rail.** A single-stage buck from 32 V down to 3.8 V runs at a duty cycle (~12%) that has poor transient response — exactly wrong for a modem TX burst. Cost: one extra conversion stage, ~2–4% additional loss.
- **Modem and MCU never share a rail.** A modem TX-burst transient sagging a shared rail could brown out the MCU. Cost: two regulators instead of one.
- **Reverse polarity via an active ideal-diode controller, not a series diode.** At 3 A burst, a diode's forward drop wastes 1–1.5 W and eats voltage headroom exactly when it's thinnest (cold crank, weak battery). Cost: one more IC, careful voltage margining against the TVS clamp level.
- **eFuse, not a PTC, for overcurrent.** Trips in microseconds instead of seconds, and can flag the MCU with a fault event before the rail collapses. Cost: higher part cost, current-limit threshold must be set with margin above legitimate TX-burst current to avoid nuisance trips.
- **Hardware supervisor gates modem enable off the MCU rail, not firmware.** Removes a circular dependency — the rail firmware needs to exist can't be sequenced by firmware that doesn't exist yet.
- **Modem rail sized to 3.0 A (GSM-fallback figure), not 2.0 A (LTE-only figure).** Coverage gaps that trigger 2G fallback are disproportionately likely in exactly the environments this product operates in. Cost: larger inductor, higher-current-rated FETs.
- **No supercap/holdup stage.** A modem brownout costs a missed report cycle, recoverable by firmware buffer-and-retry — not a safety event. Adding holdup capacitance would be solving a firmware-solvable problem in (more expensive) silicon.
- **Bidirectional input TVS, not unidirectional, plus a common-mode choke and differential filter caps.** A unidirectional TVS only clamps one polarity of transient; ISO 7637-2 includes negative-going pulses (1, 2a, 2b) that a unidirectional part passes straight through to the reverse-polarity stage. Going bidirectional clamps both directions at the one point in the circuit designed to absorb transient energy, rather than asking the ideal-diode controller's reverse-bias detection to also serve as a secondary transient clamp. Cost: bidirectional TVS parts clamp slightly more aggressively at a given standoff voltage than their unidirectional counterparts, which was factored into the voltage-margin check against the downstream FET (see `POWER_DESIGN.md` §4.1/§8.1).
- **eFuse PGOOD gates the pre-regulator enable pin, not just an MCU-readable fault flag.** The fault flag alone tells firmware something went wrong after the fact; gating the pre-regulator's enable directly in hardware means the pre-regulator is structurally prevented from switching into an unresolved input fault, instead of relying on firmware (which isn't running yet at this point in the boot sequence anyway — see S5) to notice and react. Cost: none in components, since PGOOD is a native output of the selected eFuse; the cost is one additional sequencing dependency to verify at power-up (see assumption S0 and `VALIDATION_PLAN.md` T8).

## 7. Document Map

| File | Contents |
|---|---|
| `README.md` | This file — assumptions, architecture overview, sequencing assumptions, rail dependency, rationale summary |
| `POWER_DESIGN.md` | Current budget, protection justification, rail sizing math, decoupling placement, grounding/return-path intent, modem-collapse avoidance |
| `VALIDATION_PLAN.md` | Test matrix, pass/fail thresholds, worst-case stress scenario, risk/fallback plan |
| Power-path schematic | Power path only, input connector through both regulated rails |
