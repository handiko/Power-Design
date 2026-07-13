
# VALIDATION_PLAN.md — EC200U Telematics Tracker Power Subsystem

## 1. Validation Philosophy

The design in `POWER_DESIGN.md` makes several specific, falsifiable claims — that the modem rail holds up during a 3 A burst, that the eFuse trips before a short damages anything downstream, that sequencing prevents reset chatter. Each claim gets one or more concrete, instrumented tests with a numeric pass/fail threshold, not a subjective "looks stable on the scope." Where a test could fail, this plan states in advance what the fallback action is — deciding that under test-bench pressure, after a result is already disappointing, tends to produce worse engineering decisions than deciding it now.

## 2. Test Matrix

| ID | Test | Instrument(s) | Procedure | Pass / Fail Threshold |
|---|---|---|---|---|
| T1 | Input voltage range sweep | Programmable DC supply, DMM | Sweep VIN from 8 V to 32 V in 1 V steps; verify V5, V3V3, V3V8 stay in regulation at each point | All three rails within ±3% of nominal across the full sweep |
| T2 | Reverse polarity | Programmable DC supply (reversed leads) | Apply reverse polarity at nominal voltage for 10 s | Zero current draw past the reverse-polarity FET; no downstream rail activity; device undamaged and functional after leads corrected |
| T3 | Load-dump transient (ISO 7637-2 pulse 5b, simulated) | Transient/surge generator, oscilloscope | Inject simulated load-dump pulse at input connector | TVS clamps input excursion below the reverse-polarity FET's absolute-max rating; no downstream rail deviates more than ±5% during the pulse |
| T4 | Cranking / brownout dip | Programmable DC supply (voltage dip profile) | Step VIN down to simulate cold-crank dip (e.g. 12 V → 6 V for 100 ms → 12 V) | Supervisor holds MCU in reset during the dip if V3.3V would otherwise fall below its UVLO threshold; no uncontrolled reset chatter (single clean reset cycle, not repeated toggling) |
| T5 | Modem TX-burst transient — **worst-case stress scenario** | Electronic load (or live EC200U on a network), oscilloscope on V3V8 | With VIN at worst-case low input (per T1, near the pre-regulator's dropout margin), trigger a simulated or live 3 A / ~1 ms burst on V3V8 | V3V8 sag stays above 3.3 V floor at all times during the burst; V3V3 shows no observable coupling deviation greater than ±2% during the same event |
| T6 | Overcurrent / short-circuit trip | Electronic load in short-circuit mode, oscilloscope, logic analyzer on eFuse fault flag | Command a hard short on V3V8 output | eFuse trips within its rated response time (µs-class, per selected part); fault flag asserts and is captured by the MCU before the rail collapses; no downstream component exceeds its absolute-max rating during the interval before trip |
| T7 | eFuse nuisance-trip margin | Electronic load, repeated legitimate TX-burst profile | Repeat the T5 burst profile 100 times consecutively | Zero false trips across all 100 cycles |
| T8 | Power-up sequencing | Oscilloscope (multi-channel: VIN, V3V3, V3V8, reset line, modem enable) | Cold power-up from 0 V | V3V3 reaches regulation and reset releases before modem enable asserts; modem enable never asserts while V3V3 is below the supervisor threshold |
| T9 | Thermal — continuous operation | Thermal camera or thermocouples, chamber (if available) | Run sustained average-current profile (per §2 current budget) at maximum rated ambient for 1 hour | No component exceeds its datasheet maximum junction/case temperature with margin; reverse-polarity FET and eFuse specifically checked, since these carry full system current continuously |
| T10 | Long-duration reset-loop check | Logic analyzer on reset line, extended run | Run device through 20 consecutive cold power-cycles | Zero reset loops or failed boots across all 20 cycles |

## 3. Worst-Case Electrical Stress Scenario (T5, expanded)

This is the scenario the entire two-rail, hardware-sequenced architecture exists to survive, so it gets the most detailed procedure:

1. Set VIN to the lowest voltage at which the pre-regulator is still specified to maintain output regulation (per the pre-regulator's datasheet dropout voltage, added to the required 5 V output).
2. Hold at this VIN for several minutes to let thermal and control-loop conditions settle — not just an instantaneous sweep.
3. Trigger a TX-burst-equivalent load step on V3V8 (either via a live network attach on real hardware, or an electronic load programmed to replicate a ~3 A / ~1 ms step, whichever is available first in the bring-up sequence).
4. Capture V3V8, V3V3, and V5 simultaneously on a multi-channel scope.
5. **Pass condition:** V3V8 never dips below 3.3 V at any point in the capture; V3V3 shows no coupled deviation beyond ±2%; no reset event is logged on the MCU during or after the burst.

## 4. Risk Items and Fallback Actions

| Risk | Likelihood | If the test fails |
|---|---|---|
| V3V8 sags below 3.3 V during T5 at low VIN | Medium — this is the scenario the design is explicitly sized for, but bulk capacitance or control-loop bandwidth could still be marginal in a first-pass layout | Increase local bulk capacitance at the EC200U VBAT pins beyond the vendor-minimum 100 µF; if still marginal, evaluate a faster-response buck compensation network before considering a topology change |
| eFuse nuisance-trips during legitimate bursts (T7) | Medium — this is a genuine risk flagged at design time, not a surprise | Raise the eFuse current-limit threshold incrementally, re-verify against T6 (real short-circuit protection) to confirm the new threshold still trips fast enough to protect downstream components |
| V3V3 shows coupled noise from V3V8 switching, affecting GNSS sensitivity (not in the main matrix but worth flagging) | Low-medium — depends on layout, not just schematic | Add a small LC post-filter or low-dropout linear "cleanup" stage at the GNSS supply pin, per the fallback noted in `POWER_DESIGN.md` §5.4, rather than redesigning the V3V3 rail as a full LDO |
| Reset chatter observed during T4 (cranking dip) | Low — supervisor threshold/delay should prevent this by design | Re-check supervisor threshold and delay setting against the actual UVLO point of the V3V3 buck; a mismatch between the two is the most likely root cause |
| Reverse-polarity FET or eFuse runs hotter than expected during T9 | Low-medium — depends on actual continuous current once real MCU/GNSS parts are selected | Revisit the current-budget assumptions in `POWER_DESIGN.md` §2.2 (flagged there as engineering estimates), and re-derive margin once real average current is measured |

## 5. DFT / Production Screening Notes (optional)

- Test points on V3V3, V3V8, V5, and the eFuse fault-flag line, accessible without disassembly, to allow production screening without requiring a full functional test rig for every unit.
- A simple go/no-go production test: apply nominal VIN, verify all three rails reach regulation within a specified time window, and verify the eFuse fault flag is not asserted at power-up — catches gross assembly defects without needing the full validation matrix above on every unit.
