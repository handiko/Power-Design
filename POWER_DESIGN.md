# POWER DESIGN — EC200U Telematics Tracker

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

### 4.1 Input TVS + EMI Filter

**Choice:** Automotive-grade bidirectional TVS at the input connector (36 V standoff class), immediately followed by a common-mode choke with differential-mode filter capacitors across the choke's terminals.

**Justification — bidirectional, not unidirectional:** ISO 7637-2 includes negative-going transients (pulses 1, 2a, 2b) in addition to the positive load-dump pulse (5b). A unidirectional TVS clamps only one polarity and would pass a negative transient through essentially unclamped to the reverse-polarity stage, leaving the ideal-diode controller's reverse-bias detection to do double duty as both a polarity-fault response and a transient clamp — two different response-time requirements bundled into one mechanism. Going bidirectional keeps transient clamping and polarity protection as two separate, independently-verifiable jobs. Trade-off: a bidirectional part of the same standoff voltage clamps at a marginally higher voltage than its unidirectional counterpart in the same package family, which is carried through into the FET voltage-margin check in §8.2.

**Justification — CM choke + filter caps immediately after the TVS:** The TVS is a surge-suppression device, sized for high pulse energy at low frequency; it is not a conducted-EMI filter. The common-mode choke, with differential capacitors across its terminals, is a separate, deliberately small first-pass LC filter stage aimed at conducted emissions (relevant given the wiring harness this connector sits on) rather than at surge energy — the two threats have different frequency content and are addressed by different topologies. This is a minimum first-pass filter at the architecture stage; final CM choke impedance and filter cap values are tuned against CISPR 25 conducted-emissions margin once real harness length and layout are known, not fixed here.

**Note on the module-side TVS:** This input-connector TVS is a separate part from — and serves a separate function than — the TVS Quectel recommends directly at the EC200U's own VBAT pins (4.7 V standoff, up to 2550 W peak pulse power). That part protects the module from whatever transient energy survives everything upstream of it; it would be badly undersized if used at the vehicle-facing connector, which sees the full amplitude of ISO 7637-2 load-dump pulses. Both TVS parts are included, at different points in the chain, protecting different things from different threat models.

### 4.2 Reverse polarity

**Choice:** High-side ideal-diode controller (LM74610-Q1 class) driving an external automotive-rated N-channel MOSFET.

**Justification:** At the sized burst current (3.75 A design figure), a series Schottky diode's forward drop would dissipate roughly 1–1.5 W continuously in a part that does nothing but sit in the current path, and would subtract directly from the voltage headroom between vehicle input and the pre-regulator's minimum operating voltage — headroom that is thinnest exactly when the battery is weakest (cold crank). The controller reduces this to a near-negligible I×R drop across the FET's on-resistance. Trade-off: one additional IC, and a design obligation to correctly margin the FET's voltage rating against the TVS clamp voltage per the controller vendor's automotive load-dump guidance.

### 4.3 Overcurrent / short-circuit

**Choice:** Electronic fuse (eFuse) with an adjustable current limit and a fault-flag output readable by the MCU.

**Justification:** A PTC resettable fuse's trip time (seconds) is too slow to protect the pre-regulator from a fast downstream short, and it provides no fault information to the rest of the system. An eFuse trips in microseconds and can assert a fault flag before the rail collapses, letting firmware log a timestamped fault event — materially more debuggable in the field than a silent brownout. The current-limit threshold is set above the 3.75 A design figure with enough margin to avoid nuisance-tripping during a legitimate TX burst; this threshold is the first thing to verify on the bench (see `VALIDATION_PLAN.md`).

### 4.4 Brownout / undervoltage handling

**Choice:** Dedicated voltage supervisor IC monitoring +3V3, fixed factory-trimmed threshold with an externally programmable power-on delay, gating both the MCU reset line and the modem buck's enable pin.

**Justification:** Each buck's own UVLO protects that buck from operating outside its own valid input range, but says nothing about system-level sequencing between two independently regulated rails. A dedicated supervisor is the one component whose job is enforcing the order rails come up in — and because MCU-rail stability can't depend on firmware (firmware can't run until the MCU rail is already stable), this sequencing has to live in hardware, not in a GPIO-driven enable line. The finalized part (§8.7) uses a fixed threshold trimmed for a 3.3 V rail rather than an externally-set resistor divider — this removes one source of threshold-setting error at the cost of losing the ability to retune the trip point without a different device variant, which is an acceptable trade given the monitored rail's nominal voltage isn't expected to change.

### 4.5 eFuse-gated pre-regulator enable (sequencing addition)

**Choice:** The eFuse's PGOOD output is wired to the pre-regulator's enable pin, so the pre-regulator does not begin switching until the eFuse reports the input path is fault-free.

**Justification:** This closes a gap that existed in the original protection-only view of the eFuse: a fault flag the MCU can read is useful for logging, but the MCU isn't running yet at this point in the boot sequence (see README assumption S5), so it can't act on that flag before the pre-regulator has already started. Gating the pre-regulator's own enable pin directly off PGOOD means the "is the input actually clean" decision is enforced in hardware, before any downstream stage draws current from it — consistent with the same design principle used for the +3V3-to-+3V8 supervisor gate in §4.4, just one stage earlier in the chain. Cost: none in additional components (PGOOD is a native output of the selected eFuse, §8.3); the cost is one more power-up sequencing dependency to verify on the bench (`VALIDATION_PLAN.md` T8).

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

### 8.1 Input TVS + EMI Filter

| Parameter | Selection |
|---|---|
| Representative part | Vishay **SM8S36CA** (DO-218AB), AEC-Q101 qualified, **bidirectional** load-dump TVS |
| Standoff voltage | 36 V (typ.) — clears both 12 V and 24 V nominal systems with margin |
| Breakdown voltage | 40 V (min) |
| Clamping voltage | ≈58 V class at rated pulse current — must stay below the reverse-polarity FET's and eFuse's absolute-max voltage rating (see 8.2, 8.3); re-verify against the specific SM8S36CA datasheet curve at schematic capture, since bidirectional parts clamp marginally higher than the unidirectional SM8S36A at the same standoff voltage |
| Qualification | Meets ISO 7637-2 and ISO 16750-2 surge specifications; AEC-Q101 qualified, TJ = 175 °C |
| Orientation note | Bidirectional — clamps both positive load-dump and negative transients (ISO 7637-2 pulses 1, 2a, 2b) at the same part. Reverse-battery *steady-state* current (not transient) is still handled by the dedicated reverse-polarity stage (8.2); the two mechanisms are not redundant with each other — the TVS is a fast clamp for a transient event, the ideal-diode/FET stage is a sustained block for a wired-backwards fault |
| EMI filter (added stage) | Common-mode choke placed immediately downstream of the TVS, with a differential filter capacitor pair (22 nF, 1 kV rating) across the choke's terminals |
| EMI filter justification | The TVS is a surge-energy clamp, not a conducted-EMI filter — different threat, different frequency content, different part. The choke + capacitor pair is a minimum first-pass LC filter for conducted emissions on the harness; final choke impedance and cap value are tuned against measured CISPR 25 margin once harness length and layout are fixed, not derived at the architecture stage. The 1 kV cap rating is chosen with deliberate margin over the clamped transient voltage, not the steady-state 12/24 V input, since these caps sit directly across a node the TVS has just clamped |

### 8.2 Reverse Polarity — Controller + FET

| Parameter | Selection |
|---|---|
| Controller | TI **LM74610-Q1**, AEC-Q100 qualified ideal-diode controller, drives an external high-side N-channel MOSFET, ~2 µs turn-off response on reverse-bias detection |
| External FET | 40–60 V-class automotive N-channel MOSFET, low RDS(on) (representative example: TI **CSD18540Q5B**-class, ~4–5 mΩ RDS(on), 40 V rating) |
| FET voltage rating basis | Must exceed the TVS clamping voltage from 8.1 (≈58 V) with margin — this is why a 60 V-class FET, not a 40 V-class one, is the safer default unless the clamp voltage is independently re-verified against a 40 V part |
| Conduction loss at design current | At 3.75 A (design peak) and CSD18540Q5B's actual RDS(on) — 2.2 mΩ max at VGS = 10 V, per datasheet, revised down from the ≈5 mΩ placeholder used before part finalization: P = I²R ≈ **31 mW** — negligible compared to the 1–1.5 W a series Schottky would dissipate at the same current, and lower than originally estimated |

### 8.3 Overcurrent — eFuse

| Parameter | Selection |
|---|---|
| Finalized part | TI **TPS26630RGER** (TPS2663x family), 4.5–60 V, 0.6–6 A adjustable, 31 mΩ integrated FET eFuse, 24-pin VQFN |
| Current limit setting | Set via external RILIM resistor to ~4 A, above the 3.75 A design peak (§3 margin figure) with enough headroom to avoid nuisance trips on a legitimate TX burst (validated in `VALIDATION_PLAN.md` T7) |
| Why this part specifically | Integrates adjustable OVLO, adjustable current limit, fault-flag output, current monitor, and a PGOOD output usable to enable/disable a downstream converter — the PGOOD output is what §4.5's pre-regulator gating relies on, not just a passive fault flag for firmware to poll later |
| Response time | Microsecond-class trip, per datasheet — orders of magnitude faster than a PTC, consistent with the justification in §4.3 |
| Fixed-limit vs. pulse variant, noted for the record | TPS26630 has a **fixed** overcurrent limit — once IOL is set, the device regulates at that current continuously rather than tolerating a short 2×IOL pulse the way the TPS26633 variant in the same family does. This means the RILIM current-limit threshold has to be set to comfortably clear the *full* 3.75 A design peak on a sustained basis, not just survive it as a brief pulse. This was an open trade-off at the representative-part stage (previous revision of this document); TPS26630 is the part actually selected, so the current-limit resistor and the T7 nuisance-trip validation both need to be checked against this fixed-limit behavior specifically, not against the pulse-tolerant TPS26633 behavior |
| Reverse polarity / reverse current | This part provides no reverse-current blocking or reverse-polarity protection on its own — consistent with the architecture, since that job is deliberately kept in the dedicated ideal-diode/FET stage (8.2) upstream, not duplicated here |

### 8.4 Pre-Regulator (VIN → +5V)

| Parameter | Selection |
|---|---|
| Finalized part | TI **TPS54560DDAR** (4.5–60 V, 5 A, synchronous, current-mode control, survives load-dump pulses to 65 V per ISO 7637 — same electrical load-dump spec as the -Q1 automotive-qualified sibling) |
| Automotive qualification — flagged explicitly | The -Q1 electrical/automotive-qualified variant of this exact part (**TPS54560-Q1**) exists and is pin-compatible; **TPS54560DDAR is the non-automotive (industrial/commercial-grade) part**, meaning it has not been through AEC-Q100 stress qualification even though its datasheet electrical performance (including the 65 V load-dump survival spec) is identical. This is a deliberate cost/lead-time trade at this design stage per the margin policy in §3 (A7 — asset-tracking hardware, not a safety-critical ECU), not an oversight; it is called out here so it's an explicit decision to revisit, not a silent one, before committing to a production BOM |
| Current margin | 5 A rated capability against the 3.3 A design figure (§3) — more headroom than the previously-considered 3.5 A part gave, at no inductor/layout cost difference since both are pin-compatible in the same package family |
| Enable / sequencing (new) | EN pin is driven by the eFuse's PGOOD output, not tied directly to VIN — see §4.5. The pre-regulator does not begin switching until the eFuse reports the input path fault-free |
| Inductor | ≈10 µH, saturation current rating ≥ 5 A (above the 3.3 A design figure with margin), low DCR to limit conduction loss at the reflected ≈2.64 A peak |
| Output capacitance | Ceramic bulk (e.g. 2×22 µF, X7R, rated above 5 V with derating margin) plus additional bulk capacitance sized to absorb both downstream bucks' combined transient demand without excessive droop, per §6.1 |
| Why synchronous | Per §5.2 — catch-diode conduction loss at this current level is a real efficiency/thermal cost the synchronous topology avoids |

### 8.5 +3V3 Buck (MCU / digital rail)

| Parameter | Selection |
|---|---|
| Finalized part | TI **TPS62822DLCR** — 2.4–5.5 V input, 2 A synchronous step-down converter, DCS-Control™ topology, 1% output accuracy, VSON-HR package |
| Automotive qualification — flagged explicitly | Commercial/industrial grade, **not** AEC-Q100 qualified — the earlier placeholder called out a "TPS6282x-Q1" automotive-qualified class; the part actually finalized here is the non-Q1 TPS62822. Same note as §8.4: this is downstream of the protection stage and never sees raw VIN or load-dump transients directly, which is the mitigating factor for accepting a non-automotive part here, but it's still worth stating as a conscious choice rather than leaving it implicit |
| Current margin | 2 A rated vs. the 150 mA peak design figure (§2.2) — substantial deliberate over-spec; this rail's headroom is not the tight constraint in this design (the modem rail is), so the margin here is generous by default rather than tightly derived |
| Input compatibility | 5.5 V absolute max input vs. the nominal 5.0 V pre-regulator output — roughly 0.5 V of headroom to the part's own input ceiling. Should be checked against the pre-regulator's worst-case output overshoot/ripple (not just its nominal 5.0 V) once the pre-regulator's transient response is characterized, since this is tighter headroom than the modem-rail buck sees on the same 5 V input (§8.6) |
| Inductor | 2.2 µH, per the finalized design (within the family's supported small-inductor range; saturation current rating to be selected above the 150 mA peak with generous margin, since this part is small and cheap to over-spec) |
| Output capacitance | Ceramic bulk (e.g. 22 µF) at the buck output, plus 100 nF local at each IC per §6.1 — not sized by burst current the way the modem rail is, since this rail's peak is comparatively small |
| Fallback component (contingent) | Small LC post-filter or a low-dropout linear "cleanup" stage at the GNSS supply pin specifically, only if T5/validation shows switching-noise coupling into GNSS sensitivity (§5.4, §6.3) |

### 8.6 +3V8 Buck (EC200U modem rail)

| Parameter | Selection |
|---|---|
| Finalized part | TI **TPS565201DDCR** — 4.5–17 V input, 5 A synchronous step-down converter, D-CAP2™ adaptive on-time control, 500 kHz, 31 mΩ high-side switch, SOT-23-6 package |
| Current margin against design peak | 5 A rated device capability vs. the 3.75 A design figure (§3) — this is the one part on this rail where margin actually matters, since the 3.75 A figure is the single most consequential sizing decision in the whole design (§5.3). The 5 A rating clears it with margin to spare; this should still be re-checked against the part's transient/peak-current behavior specifically during the T5 burst test, since a datasheet's steady-state current rating and its transient response during a fast 1 ms load step are not automatically the same guarantee |
| Why D-CAP2, not a fixed-frequency current-mode part like the pre-regulator | D-CAP2 (adaptive on-time) control is specifically suited to fast load-step response with minimal external compensation, which is the property this rail needs most for the TX-burst transient (§5.3) — consistent with the original rationale for wanting a "fast-transient-response" part class here, just now pinned to a specific control topology and part |
| Automotive qualification — flagged explicitly | Commercial/industrial grade, not AEC-Q100 qualified — same category of trade-off as §8.4/§8.5. This part only ever sees the clean, already-protected 5 V rail (see rationale below), which is the argument for why this is a more acceptable place to accept a non-automotive part than, say, the pre-regulator itself |
| Inductor | 2.2 µH, per the finalized design — within the ≈1.5–2.2 µH range called for in §5.3 for fast transient response to the TX-burst load step; saturation current rating still needs to clear the 3.75 A design figure with margin at schematic capture |
| Output / module-side capacitance | Per Quectel's own specification (§6.1): 100 µF low-ESR bulk (ESR ≈ 0.7 Ω) + MLCC array (100 nF, 33 pF, 10 pF) directly at the EC200U VBAT pins — non-negotiable — plus an additional local bulk capacitor at the buck's own output for control-loop stability, physically distinct from the module-side capacitance |
| Why this part over the pre-regulator's part | The pre-regulator (8.4) must survive the full wide VIN range and load-dump events directly; the +3V8 buck only ever sees the clean, already-protected 5 V rail, so it can be optimized purely for current capability and transient speed (D-CAP2, 500 kHz) rather than input voltage range or automotive qualification |

### 8.7 Supervisor / Sequencing IC

| Parameter | Selection |
|---|---|
| Finalized part | TI **TPS3808G33DBVR** — fixed-threshold (3.07 V typ. trip, factory-trimmed for a 3.3 V rail), programmable-delay supervisor, 6-pin SOT-23, 1.65–6.5 V operating supply range, 2.4 µA quiescent current |
| Threshold setting (revised from placeholder) | The earlier revision of this document called for an adjustable-threshold part with an external resistor divider off +3V3. The finalized "G33" variant instead uses a **factory-trimmed fixed threshold** (3.07 V typical, ~93% of the 3.3 V nominal rail) — this removes the external divider and its associated tolerance stack-up entirely, at the cost of no longer being field-adjustable if the monitored rail's nominal voltage ever changes. Given the rail is fixed at 3.3 V by design, this is a straightforward simplification, not a compromise |
| Delay setting | Set via an external capacitor on the CT pin (or one of two fixed preset delays if CT is tied to VDD or left floating), sized longer than +3V3's worst-case power-up ramp time across the full VIN range (README §4, assumption S3) — the actual capacitor value is still derived from the specific buck's measured ramp time, not assumed at the architecture stage |
| Automotive qualification | Commercial grade (TPS3808G33DBVR); an automotive-qualified equivalent exists in TI's portfolio (TPS3808G33Q-Q1-class part) as a pin-compatible upgrade path if required at production qualification — same category of trade-off as §8.4–§8.6 |
| Output behavior | RESET output gates both MCU reset and the +3V8 buck's enable pin; per README assumption S4, must behave as a one-way gate per power cycle rather than re-triggering on minor +3V3 ripple — this is inherent to the part's RESET-then-delay behavior (RESET reasserts and the full delay must re-elapse only if SENSE actually drops back below threshold, not on sub-threshold ripple) |

### 8.8 Automotive-Qualification Summary (cross-cutting note)

The finalized part list is a mix of AEC-qualified and commercial-grade silicon, which is worth stating in one place rather than leaving scattered across §8.1–8.7, since it's the kind of thing that should be a deliberate, defensible decision rather than something that falls out of whichever part was easiest to source first:

| Part | Automotive-qualified? |
|---|---|
| TVS — SM8S36CA | Yes (AEC-Q101) |
| Ideal-diode controller — LM74610-Q1 | Yes (AEC-Q100) |
| Reverse-polarity FET — CSD18540Q5B | No — TI does not currently offer an AEC-Q101-qualified part in this specific NexFET family |
| eFuse — TPS26630RGER | No (industrial-grade; IEC 61010-1 / UL1310-relevant surge and fire-safety features, but not AEC-Q100) |
| Pre-regulator — TPS54560DDAR | No — pin-compatible -Q1 automotive variant exists as an upgrade path |
| +3V3 buck — TPS62822DLCR | No — same family includes no -Q1 variant at this exact part number; nearest automotive alternative would need to be resourced |
| +3V8 buck — TPS565201DDCR | No |
| Supervisor — TPS3808G33DBVR | No — pin-compatible -Q1 automotive variant exists as an upgrade path |

Only the input-facing surge-critical parts (TVS, ideal-diode controller) are automotive-qualified in this finalized selection; everything downstream of the eFuse is commercial/industrial grade. This is consistent with the margin philosophy in §3 (A7 — asset-tracking hardware, not a safety-critical ECU) in that the parts seeing the harshest, least-predictable electrical environment directly are the ones held to the automotive bar, while parts operating from an already-protected, already-regulated rail are not automatically required to carry the same qualification and cost. Whether this split is acceptable for the eventual production BOM — as opposed to just this design-stage selection — is a call that should be revisited explicitly against the product's actual reliability/warranty requirements, not inherited silently from whichever part was easiest to source at this stage.

## 9. Thermal Note

This is a design-stage estimate of self-heating from the power path during **normal (average-current) operation**, not a full thermal analysis — no board copper area, enclosure volume, or airflow model is assumed yet, since none of that is fixed at this stage. It exists to sanity-check that nothing in the power path is thermally marginal by inspection, before the real answer comes from bench measurement (`VALIDATION_PLAN.md` T9).

### 9.1 Estimated dissipation at normal operating point

Using the average-current figures from §2 (150 mA on +3V8, 60 mA on +3V3 — themselves engineering assumptions, see §2.2) and the same 90% per-stage buck efficiency assumption used throughout §2.3, at a 12 V nominal input:

| Stage | Basis | Estimated dissipation |
|---|---|---|
| +3V8 buck (TPS565201DDCR) | P_out = 3.8 V × 0.15 A = 0.57 W at 90% eff. | ≈ 63 mW |
| +3V3 buck (TPS62822DLCR) | P_out = 3.3 V × 0.06 A = 0.198 W at 90% eff. | ≈ 22 mW |
| Pre-regulator (TPS54560DDAR) | Delivers the 0.853 W combined downstream demand (§2.3) at 90% eff. | ≈ 95 mW |
| Reverse-polarity FET (CSD18540Q5B) | I²R at ≈79 mA average system input current (reflected through both conversion stages) and 2.2 mΩ RDS(on) | ≈ 14 µW — negligible |
| eFuse (TPS26630RGER) | I²R at the same ≈79 mA and 31 mΩ RDS(on) | ≈ 0.2 mW — negligible |
| Quiescent overhead (supervisor, ideal-diode controller, eFuse bias, buck no-load Iq) | Sum of datasheet quiescent currents (µA-class per part) at their respective rail voltages | ≈ 1–2 mW, order-of-magnitude |
| **Total, normal operation** | Sum of the above | **≈ 0.18–0.19 W** across the whole power path |

### 9.2 What this does and doesn't tell us

The pre-regulator and the two downstream bucks dominate the normal-operation heat budget — expected, since they're the only stages doing real conversion work rather than just passing current through a low RDS(on) switch. The reverse-polarity FET and eFuse are thermally trivial under normal load; their thermal relevance is almost entirely during a fault or burst event, not steady-state operation (see below), which is why T9's pass condition explicitly checks them under sustained load rather than assuming this average-case number covers that case.

Total normal-operation dissipation across the whole power path (≈0.19 W) is small relative to what a sealed, vehicle-mounted enclosure (A6) can typically reject through the housing without forced air — but "typically" is doing real work in that sentence, since actual case-to-ambient rise depends on enclosure material, volume, and mounting, none of which are fixed yet. This estimate says the power path is not an obviously bad thermal actor by inspection; it does not replace T9, which measures actual case/junction temperature on real hardware at maximum rated ambient.

### 9.3 Burst-mode dissipation — why it's excluded from the normal-operation total above

During the ~1 ms modem TX-burst (3.75 A design peak, §3), instantaneous dissipation in the reverse-polarity FET and eFuse is higher — for example, the eFuse's 31 mΩ RDS(on) at 3.75 A is ≈436 mW instantaneous, and the FET's is ≈31 mW (§8.2) — but at a duty cycle low enough (a ~1 ms event within a multi-second reporting interval) that its contribution to *average* dissipation and steady-state case temperature is negligible next to the continuous conversion losses in §9.1. This is a duty-cycle argument, not a claim that burst-mode junction temperature is unconditionally fine — a component with too little thermal mass or too high a thermal resistance could still see a meaningful instantaneous temperature spike even from a short pulse repeated often enough. That's exactly what T9 (sustained profile) and the T5/T7 burst-repetition tests are there to catch on real hardware rather than by estimate.
