# POWER_DESIGN.md — EC200U Telematics Tracker

## 1. Rail Definitions

| Rail | Nominal Voltage | Feeds | Regulation |
|---|---|---|---|
| VIN | 8–32 V (12/24 V vehicle systems, with transients) | Pre-regulator input | N/A — raw input, protected only |
| +5V | 5.0 V | Both downstream bucks | Synchronous buck, wide-input |
| +3V3 | 3.3 V | MCU, GNSS receiver, sensors, peripherals | Synchronous buck, independent loop |
| +3V8 | 3.8 V | EC200U VBAT_RF / VBAT_BB | Synchronous buck, independent loop, sized to modem burst |

## 2. Current Budget

Two figures matter for every rail: **average current** (thermal design, battery-life estimation) and **peak/burst current** (regulator bandwidth, bulk capacitance, inductor saturation rating, and eFuse threshold setting). Conflating the two is the most common sizing mistake in this class of design, so they are tracked separately throughout.

### 2.1 +3V8 — Modem rail (EC200U)

| Mode | Current | Basis |
|---|---|---|
| Power off | ~12 µA | Per Quectel EC200U hardware design guide, typical power-off current |
| Sleep | ~1.35 mA (typ.) | Per Quectel EC200U hardware design guide |
| Idle / registered, no activity | ~28 mA (typ.) | Per Quectel EC200U hardware design guide |
| GNSS-assist / periodic wake | 80–150 mA (assumption, mid-range for this module class) | Engineering estimate — flagged as assumption pending bench measurement |
| **LTE attach / TX burst (peak)** | **up to 2.0 A** | Quectel-specified minimum supply current capability for LTE-only operation |
| **2G (GSM) fallback TX burst (peak)** | **up to 3.0 A** | Quectel-specified minimum supply current capability when 2G fallback is retained — this is the design figure used for rail sizing (see §4) |
| Average (mixed duty cycle, typical reporting interval) | ~150 mA (assumption) | Blended estimate: mostly idle/sleep with periodic short TX bursts — used for battery-life estimation only, not for regulator sizing |

### 2.2 +3V3 — MCU / digital rail

| Mode | Current | Basis |
|---|---|---|
| MCU active, peripherals idle | ~20–30 mA (assumption) | Typical Cortex-M class MCU at moderate clock, no vendor part fixed yet |
| GNSS acquisition | ~25–50 mA (assumption) | Typical standalone GNSS receiver during cold-start acquisition |
| Sensors (accel/temp/etc.) | ~5 mA (assumption) | Negligible relative to MCU/GNSS |
| **Peak (all active simultaneously)** | **~150 mA** | Sum of above with no coincidence discount — conservative on purpose |
| Average | ~60 mA (assumption) | Duty-cycled GNSS + intermittent MCU wake |

All +3V3 figures are engineering assumptions pending real part selection; they are called out explicitly because, unlike the modem figures, they are not vendor-sourced and should not be mistaken for measured or datasheet values.

### 2.3 +5V — Pre-regulator (reflected current, not raw draw)

The pre-regulator's own output current isn't simply the sum of the two downstream rails' currents — it's the current required to deliver that power through each downstream buck at its own efficiency. Assuming 90% efficiency for each downstream buck as a conservative design-stage placeholder:

- +3V8 peak reflected to +5V: P = 3.8 V × 3.0 A = 11.4 W → at 90% efficiency, P_in = 12.67 W → I = 12.67 W / 5 V ≈ **2.53 A**
- +3V3 peak reflected to +5V: P = 3.3 V × 0.15 A = 0.495 W → at 90% efficiency, P_in = 0.55 W → I = 0.55 W / 5 V ≈ **0.11 A**
- **Combined +5V peak ≈ 2.64 A**

This is the number the pre-regulator and its inductor must actually be sized against — not the sum of downstream currents at their own voltages, which would understate the real +5V-referred demand.

## 3. Margin Policy

**25% margin on every rail's peak current rating; 20% voltage headroom on TVS clamp voltage relative to downstream absolute-max ratings.**

Rationale: 30%+ margins are common in safety-critical automotive ECU design, where the cost of an undersized part failing is a safety event. This is asset-tracking hardware — a missed report cycle is the worst case of a marginal rail, not a safety event — so 25% was chosen as the point that still comfortably absorbs a TX-burst spike and cold-start inrush without forcing an oversized inductor and buck package purely out of caution that isn't proportional to the actual consequence of being wrong.

Applying 25% margin to the +5V peak figure above: 2.64 A × 1.25 ≈ **3.3 A** — this is the number used to select the pre-regulator IC, its inductor's saturation current rating, and its input/output bulk capacitors.

Applying the same margin directly to +3V8: 3.0 A × 1.25 = **3.75 A** — inductor and buck FET current ratings for the modem rail are selected against this figure, not the bare 3.0 A datasheet number.

## 4. Protection Choices and Justification

### 4.1 Input TVS

**Choice:** Automotive-grade bidirectional TVS at the input connector, standoff voltage sized above the nominal 12/24 V range with margin (33–40 V class), clamping voltage kept below the absolute-max rating of the downstream reverse-polarity FET and eFuse.

**Justification:** This is a separate part from — and serves a separate function than — the TVS Quectel recommends directly at the EC200U's own VBAT pins (4.7 V standoff, up to 2550 W peak pulse power). That part protects the module from whatever transient energy survives everything upstream of it; it would be badly undersized if used at the vehicle-facing connector, which sees the full amplitude of ISO 7637-2 load-dump pulses. Both TVS parts are included, at different points in the chain, protecting different things from different threat models.

### 4.2 Reverse polarity

**Choice:** High-side ideal-diode controller (LM74610-Q1 class) driving an external automotive-rated N-channel MOSFET.

**Justification:** At the sized burst current (3.75 A design figure), a series Schottky diode's forward drop would dissipate roughly 1–1.5 W continuously in a part that does nothing but sit in the current path, and would subtract directly from the voltage headroom between vehicle input and the pre-regulator's minimum operating voltage — headroom that is thinnest exactly when the battery is weakest (cold crank). The controller reduces this to a near-negligible I×R drop across the FET's on-resistance. Trade-off: one additional IC, and a design obligation to correctly margin the FET's voltage rating against the TVS clamp voltage per the controller vendor's automotive load-dump guidance.

### 4.3 Overcurrent / short-circuit

**Choice:** Electronic fuse (eFuse) with an adjustable current limit and a fault-flag output readable by the MCU.

**Justification:** A PTC resettable fuse's trip time (seconds) is too slow to protect the pre-regulator from a fast downstream short, and it provides no fault information to the rest of the system. An eFuse trips in microseconds and can assert a fault flag before the rail collapses, letting firmware log a timestamped fault event — materially more debuggable in the field than a silent brownout. The current-limit threshold is set above the 3.75 A design figure with enough margin to avoid nuisance-tripping during a legitimate TX burst; this threshold is the first thing to verify on the bench (see `VALIDATION_PLAN.md`).

### 4.4 Brownout / undervoltage handling

**Choice:** Dedicated voltage supervisor IC monitoring +3V3, with adjustable threshold and power-on delay, gating both the MCU reset line and the modem buck's enable pin.

**Justification:** Each buck's own UVLO protects that buck from operating outside its own valid input range, but says nothing about system-level sequencing between two independently regulated rails. A dedicated supervisor is the one component whose job is enforcing the order rails come up in — and because MCU-rail stability can't depend on firmware (firmware can't run until the MCU rail is already stable), this sequencing has to live in hardware, not in a GPIO-driven enable line.

## 5. Rail Sizing Rationale

### 5.1 Why 5 V as the intermediate rail voltage

5 V gives both downstream bucks enough step-down headroom to run in continuous conduction mode with fast transient response, without stepping down further than necessary — an unnecessarily high intermediate voltage just adds another duty-cycle penalty at the next stage with no benefit.

### 5.2 Why the pre-regulator is synchronous, not diode-rectified

At a combined +5V peak near 2.64 A (3.3 A with margin), a non-synchronous buck's catch-diode conduction loss becomes a real efficiency and thermal problem — more significant here than at the downstream stages, because the pre-regulator carries the *sum* of both rails' reflected current.

### 5.3 Why +3V8 is sized to the 3.0 A (GSM-fallback) figure, not the 2.0 A (LTE-only) figure

This is the single most consequential sizing decision in the design. The 2.0 A figure is cheaper — smaller inductor, lower peak-current FETs. But 2G fallback triggers in exactly the coverage conditions an asset tracker is disproportionately likely to encounter (rural gaps, underground/enclosed parking, fringe signal along transport routes). Sizing to 3.0 A (3.75 A with margin) means the rail doesn't fail precisely when the device needs it most, at the cost of a larger inductor and higher-current-rated switching FETs — a cost accepted deliberately, not incurred by default.

### 5.4 Why +3V3 is a synchronous buck rather than an LDO

An LDO would be simpler and electrically quieter, which is attractive for a rail feeding a GNSS receiver. It was rejected primarily on thermal grounds: linearly dropping 5 V to 3.3 V at this current level dissipates continuous power as heat with no benefit, inside an enclosure that is already thermally constrained by being sealed and vehicle-mounted. If MCU-rail switching noise turns out to be a GNSS sensitivity problem during validation, the fallback is a small LC post-filter or a low-dropout linear "cleanup" stage right at the GNSS supply pin — cheaper than running the entire rail linearly, and this is exactly the kind of thing the validation plan is designed to catch rather than assume away.

## 6. Stability and Integrity

### 6.1 Decoupling Strategy — bulk vs. local, and placement intent

Two different jobs are being done by capacitance in this design, and conflating them is a common source of a design that looks adequate on a BOM but isn't adequate in placement.

**Bulk capacitance** exists to hold up a rail's voltage across a fast current step until the regulator's control loop can respond — its job is energy storage close enough to the load that the trace inductance between the cap and the load doesn't undo the benefit. **Local (high-frequency) decoupling** exists to suppress high-frequency switching noise and provide the very first few nanoseconds of current a fast digital or RF transition needs, before even the bulk cap can respond — its job is proximity, not capacity.

| Location | Component | Placement intent |
|---|---|---|
| EC200U VBAT pins | 100 µF low-ESR bulk (ESR ≈ 0.7 Ω) + MLCC array (100 nF, 33 pF, 10 pF) | Per Quectel's own guidance, these sit **directly at the module's VBAT pins**, not just "somewhere on the +3V8 net" — the whole point of this specific placement is that it's the module's own burst-current demand being served, and trace inductance between this capacitance and the pins it's protecting directly subtracts from its effectiveness |
| +3V8 buck output (local bulk) | Additional bulk capacitor sized to the buck's own control-loop stability | Placed at the buck's own output node, close to the inductor — this capacitance is serving the converter's feedback loop, not the module directly, and is a distinct physical placement from the module-side bulk above even though both sit on the same net |
| +3V3 buck output | Bulk + 100 nF local per IC | Bulk at the buck output for loop stability; 100 nF placed at each individual IC's supply pin, as close as the footprint allows — standard practice, but worth stating explicitly: a single shared 100 nF "somewhere on the rail" does not serve the same purpose as one per IC |
| +5V pre-regulator output | Bulk sized to the combined downstream transient demand (≈2.64 A) | Placed at the pre-regulator's own output, before the net splits toward the two downstream bucks — its job is to prevent the pre-regulator's own output from sagging when both downstream bucks pull a coincident transient, which is a different failure mode than either downstream buck's own local bulk cap addresses |

The general placement principle applied throughout: bulk capacitance is placed as close as possible to *whichever load's transient it is meant to serve*, and every net gets bulk capacitance planned at each of its meaningfully distinct current-consuming points, not one bulk cap assumed to serve an entire rail end-to-end regardless of where the current is actually being pulled.

### 6.2 Avoiding Modem-Induced Rail Collapse and Reset Loops

This is one of the central failure modes the whole architecture is built to prevent, so it's worth stating explicitly as its own item rather than leaving it implicit in "the rails are independent."

The failure sequence being designed against: the modem draws a fast, large TX-burst current step → if that step is drawn from a rail shared with (or poorly isolated from) the MCU, the shared rail sags → the MCU's own supervisor or brownout detection sees an undervoltage condition and resets the MCU → the MCU reset re-initializes the modem enable line → the modem re-attaches, drawing another burst → the rail sags again → repeat. This is a self-sustaining reset loop, and it's a particularly bad failure mode because it can look, from the outside, like the device is "trying" to work (LEDs blinking, brief modem activity) while never actually completing a report cycle.

Three independent design decisions each break this loop at a different point, deliberately redundant rather than relying on one mechanism alone:

1. **Independent regulation (Section 4 rationale, Section 5.1–5.4 sizing):** +3V8 and +3V3 are generated by separate converters with separate control loops, so a +3V8 transient is not directly a +3V3 event. This is the first and strongest line of defense.
2. **Bulk capacitance at the module VBAT pins (Section 6.1):** even within the +3V8 rail itself, the vendor-specified bulk capacitance is sized specifically to keep the module's own pins from sagging below 3.3 V during a burst — addressing the problem at its source rather than only isolating its effects downstream.
3. **Supervisor debounce/delay (README Section 4, assumption S4):** even if some coupling did reach +3V3, the supervisor's release behavior is specified as a one-way gate per power cycle rather than a level-sensitive re-trigger, so a brief, sub-threshold ripple event doesn't cascade into a full resequence.

The reasoning behind using three overlapping mechanisms instead of relying on the strongest one (independent regulation) alone: rail isolation is a schematic-level guarantee, but real coupling can still occur through shared ground return paths or layout parasitics that aren't visible at the schematic level (see 6.3 below) — so the design doesn't bet the whole failure mode on one mechanism being perfectly realized in layout.

### 6.3 Grounding and Return-Path Intent

Architecture-level grounding intent, to be carried into layout rather than left implicit:

- **Star-point return at the input connector.** The protection stage (TVS, reverse-polarity FET, eFuse) returns directly to the connector's ground pin, because this path carries the highest-energy transient currents in the system (load dump, short-circuit fault current) and should not share a trace segment with any sensitive return path.
- **Per-converter power-ground islands.** Each buck's power ground — switch node, input and output capacitors — returns locally to its own ground island, stitched to the main ground plane at a single point near the input-side star point, rather than being allowed to share a return path segment with a neighboring converter's switching currents.
- **RF/GNSS ground kept geometrically separated from switching-noise sources.** The EC200U's RF ground and any GNSS receiver ground are planned to stay on the same plane as everything else but physically distanced from the buck switch-node areas, since switching noise is the most likely aggressor against RF/GNSS sensitivity — this is the layout-level counterpart to the LC-cleanup fallback mentioned in Section 5.4 if +3V3 switching noise turns out to be a problem during validation.
- **Explicit limit of this document:** this is architecture-level intent, not a routed layout. Final via-stitching density, whether any plane splits are used, and actual copper geometry require the real board stack-up, which isn't fixed yet — stating the intent here is meant to constrain the schematic/layout work that follows, not to substitute for it.

## 7. Decoupling Summary (component list)

| Location | Component | Basis |
|---|---|---|
| EC200U VBAT pins | 100 µF low-ESR bulk (ESR ≈ 0.7 Ω) + MLCC array (100 nF, 33 pF, 10 pF) | Quectel vendor specification — non-negotiable |
| +3V8 buck output (local) | Additional bulk capacitor sized to the buck's own control-loop stability requirement | Converter stability, separate from the module requirement above |
| +3V3 buck output | Standard bulk + 100 nF local per IC | Standard MCU/peripheral decoupling practice |
| +5V pre-regulator output | Bulk sized to supply both downstream bucks' combined transient demand (≈2.64 A) without excessive droop | Two-stage architecture requirement |

## 8. Critical Component Selection

This section maps every block in the power-path diagram to a specific, orderable-class part (or a tightly bounded part family) plus the critical passives around it. "Critical" here means: parts whose value or rating is derived from the sizing math above, not left at a generic default. Common decoupling caps and generic resistors are intentionally excluded, per scope — those are populated during layout, not architecture.

Parts below are named as concrete, real, currently-orderable examples to make the selection defensible and traceable to a datasheet, not as a locked BOM — the exact orderable variant (package, exact current-limit/threshold option) should be confirmed against the vendor's current parametric table at schematic-capture time.

### 8.1 Input TVS

| Parameter | Selection |
|---|---|
| Representative part | Vishay/SMC **SM8S36A** (DO-218AB), automotive-qualified, unidirectional load-dump TVS |
| Standoff voltage | 36 V — clears both 12 V and 24 V nominal systems with margin |
| Clamping voltage | ≈58 V — must stay below the reverse-polarity FET's and eFuse's absolute-max voltage rating (see 8.2, 8.3) |
| Peak pulse current | 114 A (large-package DO-218AB, sized for ISO 7637-2 pulse 5b / ISO 16750-2 load-dump energy, not just small ESD-class transients) |
| Orientation note | Unidirectional, oriented to clamp positive over-voltage only — reverse-battery current is handled by the dedicated reverse-polarity stage (8.2), not by this part |

### 8.2 Reverse Polarity — Controller + FET

| Parameter | Selection |
|---|---|
| Controller | TI **LM74610-Q1**, AEC-Q100 qualified ideal-diode controller, drives an external high-side N-channel MOSFET, ~2 µs turn-off response on reverse-bias detection |
| External FET | 40–60 V-class automotive N-channel MOSFET, low RDS(on) (representative example: TI **CSD18540Q5B**-class, ~4–5 mΩ RDS(on), 40 V rating) |
| FET voltage rating basis | Must exceed the TVS clamping voltage from 8.1 (≈58 V) with margin — this is why a 60 V-class FET, not a 40 V-class one, is the safer default unless the clamp voltage is independently re-verified against a 40 V part |
| Conduction loss at design current | At 3.75 A (design peak) and ~5 mΩ RDS(on): P = I²R ≈ 70 mW — negligible compared to the 1–1.5 W a series Schottky would dissipate at the same current |

### 8.3 Overcurrent — eFuse

| Parameter | Selection |
|---|---|
| Representative part | TI **TPS26630 / TPS26633** (TPS2663x family), 4.5–60 V, 6 A, 31 mΩ integrated FET eFuse |
| Current limit setting | Set via external RILM resistor to ~4 A, above the 3.75 A design peak (8.2/§3 margin figure) with enough headroom to avoid nuisance trips on a legitimate TX burst (validated in `VALIDATION_PLAN.md` T7) |
| Why this family specifically | Integrates adjustable OVLO, adjustable current limit, fault-flag output, and reverse-current blocking in one part — matches the requirement in §4.3/§4.4 for a fault event the MCU can log, not just a fuse that silently opens |
| Response time | Microsecond-class trip, per datasheet — orders of magnitude faster than a PTC, consistent with the justification in §4.3 |

### 8.4 Pre-Regulator (VIN → +5V)

| Parameter | Selection |
|---|---|
| Representative part | TI **TPS54360B-Q1** (60 V, 3.5 A, AEC-Q100, synchronous, survives load-dump pulses to 65 V per ISO 7637) — if the full 3.3 A design margin figure from §3 needs more headroom, the pin-compatible 5 A sibling in the same family is the direct upgrade path |
| Inductor | ≈10 µH, saturation current rating ≥ 5 A (above the 3.3 A design figure with margin), low DCR to limit conduction loss at the reflected ≈2.64 A peak |
| Output capacitance | Ceramic bulk (e.g. 2×22 µF, X7R, rated above 5 V with derating margin) plus additional bulk capacitance sized to absorb both downstream bucks' combined transient demand without excessive droop, per §6.1 |
| Why synchronous | Per §5.2 — catch-diode conduction loss at this current level is a real efficiency/thermal cost the synchronous topology avoids |

### 8.5 +3V3 Buck (MCU / digital rail)

| Parameter | Selection |
|---|---|
| Representative part class | Small synchronous buck, ≤6 V input, ≥1 A output, automotive-qualified (representative class: TI **TPS6282x-Q1** family) — exact part pinned once real MCU/GNSS current draw is measured against the §2.2 assumptions |
| Inductor | ≈2.2–4.7 µH, saturation current ≥ 0.5 A (above the 150 mA peak design figure with generous margin, since this part is small and cheap to over-spec) |
| Output capacitance | Ceramic bulk (e.g. 22 µF) at the buck output, plus 100 nF local at each IC per §6.1 — not sized by burst current the way the modem rail is, since this rail's peak is comparatively small |
| Fallback component (contingent) | Small LC post-filter or a low-dropout linear "cleanup" stage at the GNSS supply pin specifically, only if T5/validation shows switching-noise coupling into GNSS sensitivity (§5.4, §6.3) |

### 8.6 +3V8 Buck (EC200U modem rail)

| Parameter | Selection |
|---|---|
| Representative part class | Fast-transient-response synchronous buck sized for ≥6 A, optimized for low output voltage / high current point-of-load use from a clean 5 V input (representative class: TI **TPS62912**-class DCS-Control buck; automotive-qualified equivalent to be confirmed at schematic-capture time) |
| Inductor | ≈1.5–2.2 µH, saturation current ≥ 6 A (above the 3.75 A design figure with margin), low DCR — small inductance and higher switching frequency chosen specifically for fast transient response to the TX-burst load step, per §5.3 |
| Output / module-side capacitance | Per Quectel's own specification (§6.1): 100 µF low-ESR bulk (ESR ≈ 0.7 Ω) + MLCC array (100 nF, 33 pF, 10 pF) directly at the EC200U VBAT pins — non-negotiable — plus an additional local bulk capacitor at the buck's own output for control-loop stability, physically distinct from the module-side capacitance |
| Why this part class over the pre-regulator's part | The pre-regulator (8.4) must survive the full wide VIN range and load-dump events directly; the +3V8 buck only ever sees the clean, already-protected 5 V rail, so it can be optimized purely for current capability and transient speed rather than input voltage range |

### 8.7 Supervisor / Sequencing IC

| Parameter | Selection |
|---|---|
| Representative part | TI **TPS3808**-class adjustable-threshold voltage supervisor (automotive-qualified equivalent to be confirmed at schematic-capture time), threshold set via external resistor divider off +3V3, delay set via external capacitor on the delay pin |
| Threshold setting | Set to trip below +3V3's regulation point but above the MCU's own minimum operating voltage, so the supervisor acts before the MCU itself would misbehave |
| Delay setting | Sized longer than +3V3's worst-case power-up ramp time across the full VIN range (README §4, assumption S3) — the actual capacitor value is derived from the specific buck's measured ramp time, not assumed at the architecture stage |
| Output behavior | Gates both MCU reset and the +3V8 buck's enable pin; per README assumption S4, must behave as a one-way gate per power cycle rather than re-triggering on minor +3V3 ripple |
