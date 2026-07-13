# POWER_DESIGN.md — EC200U Telematics Tracker

## 1. Rail Definitions

| Rail | Nominal Voltage | Feeds | Regulation |
|---|---|---|---|
| VIN | 8–32 V (12/24 V vehicle systems, with transients) | Pre-regulator input | N/A — raw input, protected only |
| V5 | 5.0 V | Both downstream bucks | Synchronous buck, wide-input |
| V3V3 | 3.3 V | MCU, GNSS receiver, sensors, peripherals | Synchronous buck, independent loop |
| V3V8 | 3.8 V | EC200U VBAT_RF / VBAT_BB | Synchronous buck, independent loop, sized to modem burst |

## 2. Current Budget

Two figures matter for every rail: **average current** (thermal design, battery-life estimation) and **peak/burst current** (regulator bandwidth, bulk capacitance, inductor saturation rating, and eFuse threshold setting). Conflating the two is the most common sizing mistake in this class of design, so they are tracked separately throughout.

### 2.1 V3V8 — Modem rail (EC200U)

| Mode | Current | Basis |
|---|---|---|
| Power off | ~12 µA | Per Quectel EC200U hardware design guide, typical power-off current |
| Sleep | ~1.35 mA (typ.) | Per Quectel EC200U hardware design guide |
| Idle / registered, no activity | ~28 mA (typ.) | Per Quectel EC200U hardware design guide |
| GNSS-assist / periodic wake | 80–150 mA (assumption, mid-range for this module class) | Engineering estimate — flagged as assumption pending bench measurement |
| **LTE attach / TX burst (peak)** | **up to 2.0 A** | Quectel-specified minimum supply current capability for LTE-only operation |
| **2G (GSM) fallback TX burst (peak)** | **up to 3.0 A** | Quectel-specified minimum supply current capability when 2G fallback is retained — this is the design figure used for rail sizing (see §4) |
| Average (mixed duty cycle, typical reporting interval) | ~150 mA (assumption) | Blended estimate: mostly idle/sleep with periodic short TX bursts — used for battery-life estimation only, not for regulator sizing |

### 2.2 V3V3 — MCU / digital rail

| Mode | Current | Basis |
|---|---|---|
| MCU active, peripherals idle | ~20–30 mA (assumption) | Typical Cortex-M class MCU at moderate clock, no vendor part fixed yet |
| GNSS acquisition | ~25–50 mA (assumption) | Typical standalone GNSS receiver during cold-start acquisition |
| Sensors (accel/temp/etc.) | ~5 mA (assumption) | Negligible relative to MCU/GNSS |
| **Peak (all active simultaneously)** | **~150 mA** | Sum of above with no coincidence discount — conservative on purpose |
| Average | ~60 mA (assumption) | Duty-cycled GNSS + intermittent MCU wake |

All V3V3 figures are engineering assumptions pending real part selection; they are called out explicitly because, unlike the modem figures, they are not vendor-sourced and should not be mistaken for measured or datasheet values.

### 2.3 V5 — Pre-regulator (reflected current, not raw draw)

The pre-regulator's own output current isn't simply the sum of the two downstream rails' currents — it's the current required to deliver that power through each downstream buck at its own efficiency. Assuming 90% efficiency for each downstream buck as a conservative design-stage placeholder:

- V3V8 peak reflected to V5: P = 3.8 V × 3.0 A = 11.4 W → at 90% efficiency, P_in = 12.67 W → I = 12.67 W / 5 V ≈ **2.53 A**
- V3V3 peak reflected to V5: P = 3.3 V × 0.15 A = 0.495 W → at 90% efficiency, P_in = 0.55 W → I = 0.55 W / 5 V ≈ **0.11 A**
- **Combined V5 peak ≈ 2.64 A**

This is the number the pre-regulator and its inductor must actually be sized against — not the sum of downstream currents at their own voltages, which would understate the real V5-referred demand.

## 3. Margin Policy

**25% margin on every rail's peak current rating; 20% voltage headroom on TVS clamp voltage relative to downstream absolute-max ratings.**

Rationale: 30%+ margins are common in safety-critical automotive ECU design, where the cost of an undersized part failing is a safety event. This is asset-tracking hardware — a missed report cycle is the worst case of a marginal rail, not a safety event — so 25% was chosen as the point that still comfortably absorbs a TX-burst spike and cold-start inrush without forcing an oversized inductor and buck package purely out of caution that isn't proportional to the actual consequence of being wrong.

Applying 25% margin to the V5 peak figure above: 2.64 A × 1.25 ≈ **3.3 A** — this is the number used to select the pre-regulator IC, its inductor's saturation current rating, and its input/output bulk capacitors.

Applying the same margin directly to V3V8: 3.0 A × 1.25 = **3.75 A** — inductor and buck FET current ratings for the modem rail are selected against this figure, not the bare 3.0 A datasheet number.

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

**Choice:** Dedicated voltage supervisor IC monitoring V3V3, with adjustable threshold and power-on delay, gating both the MCU reset line and the modem buck's enable pin.

**Justification:** Each buck's own UVLO protects that buck from operating outside its own valid input range, but says nothing about system-level sequencing between two independently regulated rails. A dedicated supervisor is the one component whose job is enforcing the order rails come up in — and because MCU-rail stability can't depend on firmware (firmware can't run until the MCU rail is already stable), this sequencing has to live in hardware, not in a GPIO-driven enable line.

## 5. Rail Sizing Rationale

### 5.1 Why 5 V as the intermediate rail voltage

5 V gives both downstream bucks enough step-down headroom to run in continuous conduction mode with fast transient response, without stepping down further than necessary — an unnecessarily high intermediate voltage just adds another duty-cycle penalty at the next stage with no benefit.

### 5.2 Why the pre-regulator is synchronous, not diode-rectified

At a combined V5 peak near 2.64 A (3.3 A with margin), a non-synchronous buck's catch-diode conduction loss becomes a real efficiency and thermal problem — more significant here than at the downstream stages, because the pre-regulator carries the *sum* of both rails' reflected current.

### 5.3 Why V3V8 is sized to the 3.0 A (GSM-fallback) figure, not the 2.0 A (LTE-only) figure

This is the single most consequential sizing decision in the design. The 2.0 A figure is cheaper — smaller inductor, lower peak-current FETs. But 2G fallback triggers in exactly the coverage conditions an asset tracker is disproportionately likely to encounter (rural gaps, underground/enclosed parking, fringe signal along transport routes). Sizing to 3.0 A (3.75 A with margin) means the rail doesn't fail precisely when the device needs it most, at the cost of a larger inductor and higher-current-rated switching FETs — a cost accepted deliberately, not incurred by default.

### 5.4 Why V3V3 is a synchronous buck rather than an LDO

An LDO would be simpler and electrically quieter, which is attractive for a rail feeding a GNSS receiver. It was rejected primarily on thermal grounds: linearly dropping 5 V to 3.3 V at this current level dissipates continuous power as heat with no benefit, inside an enclosure that is already thermally constrained by being sealed and vehicle-mounted. If MCU-rail switching noise turns out to be a GNSS sensitivity problem during validation, the fallback is a small LC post-filter or a low-dropout linear "cleanup" stage right at the GNSS supply pin — cheaper than running the entire rail linearly, and this is exactly the kind of thing the validation plan is designed to catch rather than assume away.

## 6. Stability and Integrity

### 6.1 Decoupling Strategy — bulk vs. local, and placement intent

Two different jobs are being done by capacitance in this design, and conflating them is a common source of a design that looks adequate on a BOM but isn't adequate in placement.

**Bulk capacitance** exists to hold up a rail's voltage across a fast current step until the regulator's control loop can respond — its job is energy storage close enough to the load that the trace inductance between the cap and the load doesn't undo the benefit. **Local (high-frequency) decoupling** exists to suppress high-frequency switching noise and provide the very first few nanoseconds of current a fast digital or RF transition needs, before even the bulk cap can respond — its job is proximity, not capacity.

| Location | Component | Placement intent |
|---|---|---|
| EC200U VBAT pins | 100 µF low-ESR bulk (ESR ≈ 0.7 Ω) + MLCC array (100 nF, 33 pF, 10 pF) | Per Quectel's own guidance, these sit **directly at the module's VBAT pins**, not just "somewhere on the V3V8 net" — the whole point of this specific placement is that it's the module's own burst-current demand being served, and trace inductance between this capacitance and the pins it's protecting directly subtracts from its effectiveness |
| V3V8 buck output (local bulk) | Additional bulk capacitor sized to the buck's own control-loop stability | Placed at the buck's own output node, close to the inductor — this capacitance is serving the converter's feedback loop, not the module directly, and is a distinct physical placement from the module-side bulk above even though both sit on the same net |
| V3V3 buck output | Bulk + 100 nF local per IC | Bulk at the buck output for loop stability; 100 nF placed at each individual IC's supply pin, as close as the footprint allows — standard practice, but worth stating explicitly: a single shared 100 nF "somewhere on the rail" does not serve the same purpose as one per IC |
| V5 pre-regulator output | Bulk sized to the combined downstream transient demand (≈2.64 A) | Placed at the pre-regulator's own output, before the net splits toward the two downstream bucks — its job is to prevent the pre-regulator's own output from sagging when both downstream bucks pull a coincident transient, which is a different failure mode than either downstream buck's own local bulk cap addresses |

The general placement principle applied throughout: bulk capacitance is placed as close as possible to *whichever load's transient it is meant to serve*, and every net gets bulk capacitance planned at each of its meaningfully distinct current-consuming points, not one bulk cap assumed to serve an entire rail end-to-end regardless of where the current is actually being pulled.

### 6.2 Avoiding Modem-Induced Rail Collapse and Reset Loops

This is one of the central failure modes the whole architecture is built to prevent, so it's worth stating explicitly as its own item rather than leaving it implicit in "the rails are independent."

The failure sequence being designed against: the modem draws a fast, large TX-burst current step → if that step is drawn from a rail shared with (or poorly isolated from) the MCU, the shared rail sags → the MCU's own supervisor or brownout detection sees an undervoltage condition and resets the MCU → the MCU reset re-initializes the modem enable line → the modem re-attaches, drawing another burst → the rail sags again → repeat. This is a self-sustaining reset loop, and it's a particularly bad failure mode because it can look, from the outside, like the device is "trying" to work (LEDs blinking, brief modem activity) while never actually completing a report cycle.

Three independent design decisions each break this loop at a different point, deliberately redundant rather than relying on one mechanism alone:

1. **Independent regulation (Section 4 rationale, Section 5.1–5.4 sizing):** V3V8 and V3V3 are generated by separate converters with separate control loops, so a V3V8 transient is not directly a V3V3 event. This is the first and strongest line of defense.
2. **Bulk capacitance at the module VBAT pins (Section 6.1):** even within the V3V8 rail itself, the vendor-specified bulk capacitance is sized specifically to keep the module's own pins from sagging below 3.3 V during a burst — addressing the problem at its source rather than only isolating its effects downstream.
3. **Supervisor debounce/delay (README Section 4, assumption S4):** even if some coupling did reach V3V3, the supervisor's release behavior is specified as a one-way gate per power cycle rather than a level-sensitive re-trigger, so a brief, sub-threshold ripple event doesn't cascade into a full resequence.

The reasoning behind using three overlapping mechanisms instead of relying on the strongest one (independent regulation) alone: rail isolation is a schematic-level guarantee, but real coupling can still occur through shared ground return paths or layout parasitics that aren't visible at the schematic level (see 6.3 below) — so the design doesn't bet the whole failure mode on one mechanism being perfectly realized in layout.

### 6.3 Grounding and Return-Path Intent

Architecture-level grounding intent, to be carried into layout rather than left implicit:

- **Star-point return at the input connector.** The protection stage (TVS, reverse-polarity FET, eFuse) returns directly to the connector's ground pin, because this path carries the highest-energy transient currents in the system (load dump, short-circuit fault current) and should not share a trace segment with any sensitive return path.
- **Per-converter power-ground islands.** Each buck's power ground — switch node, input and output capacitors — returns locally to its own ground island, stitched to the main ground plane at a single point near the input-side star point, rather than being allowed to share a return path segment with a neighboring converter's switching currents.
- **RF/GNSS ground kept geometrically separated from switching-noise sources.** The EC200U's RF ground and any GNSS receiver ground are planned to stay on the same plane as everything else but physically distanced from the buck switch-node areas, since switching noise is the most likely aggressor against RF/GNSS sensitivity — this is the layout-level counterpart to the LC-cleanup fallback mentioned in Section 5.4 if V3V3 switching noise turns out to be a problem during validation.
- **Explicit limit of this document:** this is architecture-level intent, not a routed layout. Final via-stitching density, whether any plane splits are used, and actual copper geometry require the real board stack-up, which isn't fixed yet — stating the intent here is meant to constrain the schematic/layout work that follows, not to substitute for it.

## 7. Decoupling Summary (component list)

| Location | Component | Basis |
|---|---|---|
| EC200U VBAT pins | 100 µF low-ESR bulk (ESR ≈ 0.7 Ω) + MLCC array (100 nF, 33 pF, 10 pF) | Quectel vendor specification — non-negotiable |
| V3V8 buck output (local) | Additional bulk capacitor sized to the buck's own control-loop stability requirement | Converter stability, separate from the module requirement above |
| V3V3 buck output | Standard bulk + 100 nF local per IC | Standard MCU/peripheral decoupling practice |
| V5 pre-regulator output | Bulk sized to supply both downstream bucks' combined transient demand (≈2.64 A) without excessive droop | Two-stage architecture requirement |
