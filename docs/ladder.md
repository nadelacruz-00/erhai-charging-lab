# The ladder

Every rung records its numbers here before the next one runs. A rung without a
number has no verdict. Commands that change the system need root; run them as
`sudo ...` (in Claude Code: `! sudo ...`).

Design and eng review: `~/.gstack/projects/charging-fixes/nadelacruz-unknown-design-20261022-114844.md`.

## Safety contract

- **Revert path:** a forced restart (hold **power + volume up**), then pick the
  **package kernel** in the rEFInd menu (`7.3.0-rc5-sm8750-themojoman`,
  package `oneplus-erhai-linux-7.3` `+b29` on 2026-10-22).
- **Hold the kernel package** for the campaign so the baseline cannot move:
  `sudo apt-mark hold oneplus-erhai-linux-7.3` (release with `unhold` at the end).
- A lab kernel only ever adds `/boot/<kver>-lab-<candidate>.efi` and
  `/usr/lib/modules/<kver>-lab-<candidate>/`. rEFInd's default never changes.
- Every charging parameter an experiment changes is a runtime `0644` module
  param; a reboot restores all of them. The restore order below still applies
  without a reboot.
- **Never re-enable the charge pump (`pps_cp_enable=1`) while
  `pps_max_curr_ma` or `pps_target_curr_ma` is above 3000**, except with the
  240 W cable (5 A, confirmed in c1.5). Any other cable gets its own c1.5 first.
- Above 3000 mA, raise the current in one step per run (4000, then 5000)
  and stop the run if the pack goes above 45 °C.
- Charging stops at 80 % by design (`erhai_otg_glink` `max_soc=80`). Every
  measurement below needs the battery **below 75 %**.

## Measurement conditions

- Watts: the Anker 737's own display, read after 2 minutes of steady charging.
- Tablet side: `/sys/class/power_supply/erhai-adsp-bat/{voltage_now,current_now,capacity}`
  and the `erhai_pps: charging vbus … ibat …` log lines (`journalctl -k`).
- Dock runs: monitor and keyboard attached, nothing else; write down the dock model once.
- Record battery %, temperature and the time with every number.

## Base

| Item | Value |
|---|---|
| Port tree | `TheMojoMan/oneplus-erhai-linux` @ `candidates/BASE_SHA` (`74ba8ddb4`, 7.3-rc5) |
| Package kernel | `7.3.0-rc5-sm8750-themojoman+b29` |
| Charger | Anker 737 PowerCore 24K (PD 3.1, PPS) |
| Dock | Cygnett Unite 7-in-1 USB-C Hub (single HDMI) |
| Cable brick→tablet | _rating: unknown until c-1 / c1.5_ |

## c-1 — Android ground truth (no Linux code)

Boot Android. Battery below 75 %. Same Anker, same cable, same dock.

| Setup | Battery % | Anker watts | VBUS (mV) | IBUS (mA) | Source of VBUS/IBUS |
|---|---|---|---|---|---|
| Android, through the dock | 33 → 33 | 8-9 | not read | not read | — |
| Android, direct, screen on | 33 → 36 (both direct runs) | 35-36 | not read | not read | — |
| Android, direct, screen off | (see above) | 46-47 | not read | not read | — |

Run 2026-10-22 (Android, same Anker 737, same cable, same dock, about 2 min
per setup). Watts are the Anker's display, so the dock row includes the dock's
own draw (hub + HDMI). 33 → 33 over 2 minutes does not prove zero net charge:
1 % of this pack takes several minutes even at 5 W net.

Result for the dock: **Android gets the same 8-9 W as Linux.** Neither of the
ladder's two outcomes was this one. The dock's 18 W target came from vendor
source (9 V × `pd-iclmax` 2000), not from a measurement on this dock with this
Anker, and Android does not reach it. Before c2a is written, the open question
is whether the dock offers the tablet anything above 5 V with the Anker
upstream.

Result for direct: the fair Linux comparison is screen on, **35-36 W on Android
vs 24-25 W on Linux**; the ceiling is 46-47 W screen off. 46-47 W through a
2:1 pump (VBUS_OVP 10 V) means more than 4.6 A on the bus, which a PD source
only gives over a 5 A e-marked cable. That points to a 5 A cable but is not
the c1.5 measurement: the bus voltage was not read.

Decision: if Android through the dock is well above 27 W, premise 1 (dock =
fixed 9 V at most) is wrong and the fixed-PD design is re-derived first. If
Android direct is about 47-48 W, that operating point (VBUS × IBUS at the same
battery %) is c3's target, and the cable carries about 5 A (settles c1.5).

## c0 — `00-cp-trace` (no behaviour change)

Pass: one pump engage captured in `/sys/kernel/tracing/trace` at
`pps_poll_ms=100`, with ibus; Anker direct still 24-25 W.

Capture (root): `pps_poll_ms=100`, then
`echo 1 > /sys/kernel/tracing/events/erhai_pps/enable`, plug in, save
`/sys/kernel/tracing/trace`. Restore `pps_poll_ms=2000`. Samples carry the
ADSP vbus, the setpoints, the pack, the pump's own vbus/vbat/diff/ibus, and
raw REG_06/REG_07 (CHG_EN is bit 7 of REG_07). REG_0F is not read: an interrupt-flag register
may clear on read and hide flags from the IRQ handler. The design asked for it.

c0 also prints one `erhai_pps: adapter online: real_type … pps_type …
pps_max_curr … vbus …` line per plug-in, for any adapter. The dock's line is
the dock real_type read (and its VBUS: 5 V or more). The Anker's
`pps_max_curr` is the current its PPS offer allows: above 3 A the Anker has
seen a 5 A cable. The unit is raw ADSP, so compare it against a known 3 A cable.

| Run | Battery % | Engage (vbus/vbat/diff/ibus) | Drop (vbus/vbat/diff/ibus) | Anker watts |
|---|---|---|---|---|
| 1 (2026-10-22 17:40, Anker direct, `lab-00-cp-trace-2`) | 42-43 | 8043/3946/151/20 (trace, t=214.4 s) | 8006/3951/104/155 (t=215.1 s) | 17-18 |

**c0 passes:** the trace captured 3 engages and 3 drops at a median 124 ms
cadence (max 166 ms, so no poll slip; T10 is not needed). 1627 samples, saved at
`~/c0-anker.trace`. Tracing went on after the plug-in, so the trace covers
t=149-353 s. The first three engages (t=26-100 s) are in dmesg only.

What the trace shows, with the pump engaged (18 samples):
- REG_07 = 128 (CHG_EN latched) and REG_06 = 4 (CP switching bit set): **the
  pump really was switching.**
- **ibus 121-165 mA.** The pump carried almost nothing; ibat stayed at
  3.0-3.2 A, the same as on the buck alone. Anker 17-18 W.
- **diff 110-125 mV**, right on the 110 mV floor, then the drop at 104-106 mV.
- **The bus never followed the voltage servo.** Every sample has the PPS
  setpoint at the 9500 mV ceiling (the servo walked up there before the
  trace started), yet ADSP vbus is 7953-8346 mV and chip vbus at most 8047 mV.
  Input was about 2.2 A (17.5 W / 8.05 V), under the 3000 mA setpoint, so the
  Anker was not current-limiting.

The pump's ADCs read frozen values while it is off (chip vbus 7935 for 60 s,
then 8006), so chip readings with `cp=0` are stale. Use them only while
engaged.

dmesg engages before the trace (poll 2000 ms): t=26.6 s diff 568 ibus 18 →
drop 28.7 s (diff 62, ibus 18); 33.0 s diff 253 → held **55 s**, drop at
diff 107, ibus 238; 90.5 s diff 186 → held 9 s, drop at diff 103, **ibus 555**.

Seen before c0, package kernel `+b29`, 2026-10-22 16:57, Anker direct, 35 %:
the pump engaged at 16:57:06 (`diff 653 mV`) and held for **6.5 minutes**
(the engine polled at the 500 ms `pps_cp_feed_ms` the whole time, which it
uses only while the pump counts as engaged). It dropped at 17:03:32 at
`diff 52 mV` (floor 110). VBUS was 8.01-8.06 V at the 3000 mA PPS setpoint,
ibat ~4.2-4.3 A with the pump engaged; at 17:07, on the buck path, ibat was
2.36 A at 38 %. The Anker showed **17-18 W** when read around 17:05, after
the drop. Bus current was not measured. The 3000 mA is a setpoint, not a
reading. Whether the pump was switching (ibus) is what c0 shows.

Replug at 17:11:44 (39 %): the pump engaged twice during the current ramp
(PPS setpoint 1250 then 1750 mA, `diff 486 mV`). Both times the very next
feed dropped it (`diff 94`, then `85 mV`), and the session settled on the
buck path at 8.13 V, ibat ~2.8 A. The Anker read **17-18 W**. No pump-on
watts reading yet.

Two leads for c0 to test (hypotheses, not findings):
1. **Instant drops:** the 16:57 engage that held came at a 2000 mA setpoint.
   The two instant drops came at 1250 and 1750 mA. If the pump draws more than
   the PPS current setpoint, the source current-limits and VBUS sags below the
   110 mV floor. c0's ibus at the drop (Experiment 1) shows whether ibus sits at
   the setpoint.
2. **The 6.5-minute drop:** in `erhai_pps.c` the PPS voltage servo runs only
   while the pump is *not* engaged (the `else` branch of `a->cp_engaged()`).
   While engaged, VBUS stays put as vbat rises, so `diff` can only shrink.
   That fits a slow slide from 653 to 52 mV. c0's per-poll `diff` shows the
   slope.

## c1 — experiments (params only)

### Experiment 2a — who owns VBUS (today's package kernel, no build)

1. Direct Anker, dock out, battery below 75 %.
2. `echo 0 | sudo tee /sys/module/erhai_pps/parameters/pps_cp_enable` (pump off first)
3. `echo 4000 | sudo tee /sys/module/erhai_pps/parameters/pps_max_curr_ma`
4. `echo 4000 | sudo tee /sys/module/erhai_pps/parameters/pps_target_curr_ma`
5. `echo 8900 | sudo tee /sys/module/erhai_pps/parameters/pps_target_volt_mv`
6. Unplug, replug; after `ramp -> charging`, read `vbus` and `vbat` on the tick lines.
7. Restore, **in this order**: unplug; `pps_max_curr_ma=3000`,
   `pps_target_curr_ma=3000`, `pps_target_volt_mv=8400`; only then
   `pps_cp_enable=1`. Or reboot.

| Battery % | vbus (mV) | vbat (mV) | 2×vbat (mV) | Verdict |
|---|---|---|---|---|
| 67 | 8702-8792 (steady ~8780 for 2 min) | 4190-4208 | ~8400 | **Linux owns VBUS → branch (a)** |

Run 2026-10-22 13:23 (package kernel `+b29`, Anker 737 direct, dock out,
34 °C). Requested 8900 mV / 4000 mA with `pps_cp_enable=0`; VBUS followed the
request to within ~120 mV and sat ~380 mV above 2×vbat. Experiment 2b is not
needed. The earlier ~8.2 V "ADSP settle" (requested 8400 mV with PPS current
= ICL = 3000 mA) was the source sitting at its own current limit.

Side observation, same run: Anker display **18.2 W** versus 24-25 W at the
default 3000/3000; tablet log agrees (ibat ~2.7 A here vs ~4.3 A before), so
input was ~2.1 A at 8.78 V instead of ~3 A at 8.2 V. The input limit the ADSP
applied is not readable without root. Open question for c0's trace: is the
buck input capped near 2 A whenever the source is not current-limiting?

Verdict: vbus ≈ 8900 → Linux owns VBUS → branch (a). vbus ≈ 2×vbat → run 2b.

### Experiment 1 — ibus at the drop (c0 kernel, trace on)

| Battery % | ibus_master before the drop (mA) | Band | Conclusion |
|---|---|---|---|
| 42-43 | 18-238, once 555 (6 drops) | < 500 (5 of 6) | the pump switched (REG_06 bit 2, REG_07 bit 7) but carried ~150 mA: diff sat at the 110 mV floor because VBUS stayed at ~8.05 V while the setpoint read 9500 |

Lead for c3 (hypothesis): the first `SET_PPS_VOLT` write after entry sets
the bus, and later writes do not move it. The code already says this for
the ramp (`pps_start_volt_mv` comment: "the ADSP latches the PPS APDO range
to the first 34 write"). Experiment 2a fits it too: an 8900 first write gave
8.78 V; here an 8400 first write gave ~8.05 V and the walk to 9500 did
nothing. The 16:57 session that held 6.5 min engaged at diff 653. If this
holds, the pump needs a bigger diff at engage, and that has to come from
the entry voltage.

Bands: ≥ 2800 mA → the pump was delivering and software killed it; 500-2800 mA →
conducting but current-limited (keep the ibus check, revisit `pps_cp_min_ibus_ma`);
< 500 mA → the pump never conducted.

### Experiment 3 — does the first voltage request set the bus? (c0 kernel, D13)

Params only; currents stay at 3000 mA; the pump's own 110-700 mV window applies.

1. Unplug everything. Battery below 75 %.
2. `pps_target_volt_mv=8600`, `pps_poll_ms=100`, `pps_cp_feed_ms=100`; enable
   and clear the trace **before** plugging in.
3. Plug the Anker direct; 3 minutes; read the Anker's watts.
4. Save the trace to `~/c0-exp3.trace`; restore `pps_target_volt_mv=8400`,
   `pps_poll_ms=2000`, `pps_cp_feed_ms=500`; unplug.

| Battery % | First request (mV) | ADSP vbus (mV) | diff at engage (mV) | ibus while engaged (mA) | Held | Anker watts |
|---|---|---|---|---|---|---|
| 43-44 | 8600 | 8578-8647 before load; ~8.05 V with the pump on; 7.0-7.7 V on the buck after | 693, then 512 | 429-496 for ~0.5 s during the current ramp, then 18 (the ADC's idle value) | 6 s (747.8-753.9) | **24-26 W from ~1:20, steady** |

Run 2026-10-22 17:52, c0 kernel, trace `~/c0-exp3.trace` (t=720-853 s). Params
restored by the owner and checked by readback (8400 / 2000 / 500).

What it shows:
1. **The entry write sets the bus with no load:** 8600 requested → 8.58-8.65 V
   before charging started. That also explains c0 run 1 (8400 first →
   ~8.05 V at the tablet at 2.2 A).
2. **Under load the Anker current-limits at the 3000 mA setpoint.** After the
   drop, VBUS sat at 7.0-7.7 V while the setpoint read 9500, and the Anker
   showed 24-26 W (about 3.1-3.4 A at its output). A PPS source in current
   limit lets its voltage sag to whatever the load holds it at.
3. **The buck takes the whole 3 A, so the pump gets none.** The pump and the
   buck share one 3 A supply. The buck's input limit (`pps_iclim_ma`) is
   also 3000, so it absorbs all of it; the bus sags until the pump's share is zero
   (diff ~120 mV, ibus 18 mA, REG_06/07 still say switching). The only pump
   current seen (~430-500 mA) came while the setpoint was still ramping.
4. **Run 1's 17-18 W vs today's 24-26 W is the buck's input current**
   (ibat ~2.8 A vs ~4.0 A). Run 1 never hit the 3 A limit (2.2 A at 8.05 V);
   this run did. Why the buck settled at ~2.2 A in run 1 and ~3 A here is not
   known; both sessions had `pps_iclim_ma=3000`.

Verdict: the bus starts at the entry request, and under load the source's
current limit sets it. The pump can only carry current the buck leaves
over. So **the pump needs the source limit above the buck's input limit**:
either a lower `pps_iclim_ma` or a higher PPS current (5 A now confirmed for
this cable).

Original verdict rule: vbus lands near 8.6 V minus the same offset, and ibus rises well
above 150 mA → the entry write sets the bus; c3 picks the entry voltage from
2×vbat. vbus stays ~8.05 V → the bus is set elsewhere (record, rethink c3).

### Experiment 4 — buck input limited to 1 A (c0 kernel, D14)

One change from Experiment 3: `pps_iclim_ma=1000` (the buck's input limit,
written at negotiate time). The PPS current stays at 3000 mA, so the Anker
supplies the same 3 A as today; if the pump works, it carries the ~2 A the
buck leaves. If the pump drops, charging falls to the 1 A buck until you
replug with the defaults.

1. Unplug everything. Battery below 75 %.
2. `pps_iclim_ma=1000`, `pps_target_volt_mv=8600`, `pps_poll_ms=100`,
   `pps_cp_feed_ms=100`; clear the trace **before** plugging in.
3. Plug the Anker direct; 3 minutes; read the Anker's watts.
4. Save the trace to `~/c0-exp4.trace`; restore `pps_iclim_ma=3000`,
   `pps_target_volt_mv=8400`, `pps_poll_ms=2000`, `pps_cp_feed_ms=500`;
   unplug and replug (the ICL only takes effect at negotiate).

| Battery % | ibus while engaged (mA) | diff (mV) | ibat (mA) | Held | Anker watts |
|---|---|---|---|---|---|
| 43 | 538 at the drop | 685 at engage → 92 at the drop | 650-1070 on the 1 A buck | **0.33 s**, never re-engaged | 8-9 |

Run 2026-10-22 20:32, trace `~/c0-exp4.trace`. Params restored by the owner
and checked by readback (3000 / 8400 / 2000 / 500).

What it shows:
1. **The pump engaged while the PPS current setpoint was 1500 mA** (still
   ramping). It started carrying current (538 mA) on top of the ~1 A buck,
   the total hit the 1500 mA source limit, chip vbus fell 560 mV
   (8471 → 7912) in 0.33 s, and the window floor dropped it. That fits lead 1:
   engaging before the current ramp is done starves the pump.
2. **After the drop the pump's ADC is off, so the engine steers on stale
   numbers.** The pump teardown (`sc8547_reset_voocphy`, `sc8547a.c:799`)
   writes REG_11 = 0x00, which turns the ADC off. The next 855 samples (85 s) all read
   `chip_vbus=7912 chip_vbat=3910 diff=92 ibus=556`, while the ADSP vbus
   read 8583-8625 V. On that frozen `diff 92` the voltage servo walked the
   setpoint from 8600 to the 9500 ceiling in 50 mV steps, and `cp_engage()`
   (which reads the same registers) kept refusing. **The pump can never
   re-engage after a drop until the ADC is turned on again.** This is a code bug,
   not a tuning problem.
3. 8-9 W = the 1 A buck at 8.6 V, as expected after a drop.

Verdict: neither outcome in the rule below. The pump got 0.33 s, so whether
it carries load for minutes when it has room is still open.

Verdict: ibus ≈ 1500-2000 mA held for minutes → the pump carries load when
the buck leaves room; c3 limits the buck's ICL while the pump runs, then
raises the PPS current. ibus stays ~18 mA → something other than the buck
starves the pump (record, look at REG_06 bits and the ADSP).

### Experiment 5 — the pump's first try with room (c0 kernel, D15)

Changes from Experiment 4: `pps_start_curr_ma=3000` (the PPS current is
3000 mA from negotiate, no ramp) and `pps_target_volt_mv=8500` (at 44 %,
2×vbat ≈ 7.85 V, so the entry lands near diff 600, inside the 700 ceiling).
The buck stays at `pps_iclim_ma=1000`, so the pump has ~2 A of room at its
first engage. Total current stays at 3000 mA. Because of the ADC bug the pump gets one try
per plug.

1. Unplug the Anker. Battery below 75 %.
2. `pps_start_curr_ma=3000`, `pps_iclim_ma=1000`, `pps_target_volt_mv=8500`,
   `pps_poll_ms=100`, `pps_cp_feed_ms=100`; clear the trace.
3. Plug the Anker direct; 3 minutes; read the Anker's watts.
4. Save the trace to `~/c0-exp5.trace`; restore `pps_start_curr_ma=1000`,
   `pps_iclim_ma=3000`, `pps_target_volt_mv=8400`, `pps_poll_ms=2000`,
   `pps_cp_feed_ms=500`; unplug and replug.

| Battery % | diff at engage (mV) | ibus while engaged (mA) | Held | ibat (mA) | Anker watts |
|---|---|---|---|---|---|
| 43-44 | 686 | **860-1020, steady** (diff 145-182) | **2 min, no drop**, until the unplug at step 4 | 2400-2850 | 2-3 for ~3 min before the engine armed, then **17** |

Run 2026-10-22 20:36-20:41, trace `~/c0-exp5.trace` (t=1680-1792 s; engaged
from 1708.05 to the detach at 1832.3). Params restored by the owner and
checked by readback (1000 / 3000 / 8400 / 2000 / 500).

What it shows:
1. **With room at engage, the pump holds.** The PPS current was 3000 mA from
   negotiate, and the pump stayed switching (REG_06 = 4, REG_07 = 128) for the
   whole run without a single drop. That confirms lead 1: Experiment 4's
   0.33 s drop came from engaging while the current was still ramping.
2. **But it carries only ~1 A of the 2 A room.** With the buck at ~1 A, the input was
   ~2 A at ~8.05 V (≈16 W; the Anker showed 17 W), so the Anker was not
   current-limiting. The pump's current follows the voltage margin: diff held
   at ~160 mV and ibus at ~1 A. The bus sat at ~8.05 V at the tablet against
   the 8500 mV setpoint; while engaged the engine never moves the setpoint.
3. **Why the 2-3 W start:** the 20:36 plug-in (t=1511) reported
   `real_type 0` (unknown), so the engine stayed idle at 5 V. The ADSP
   reported PD_PPS only 2 min 48 s later (t=1679). This happened on a quick
   re-plug. Seen once; cause not known.

Verdict: room fixes the drops. **More current needs more voltage while the
pump runs**, which brings back the open question: do `SET_PPS_VOLT` writes
after entry move the bus? Run 1 (setpoint 9500, bus ~8.05 V at 2.2 A, the
Anker not current-limiting) says no; the vendor's Android code shows how
Android does it.

Verdict: ibus ~1500-2000 mA held for minutes → c3 = keep the ADC on, engage
only once the PPS current is at its target, buck ICL below the PPS current,
and then raise the PPS current toward 5 A. A drop within seconds with ibus
near 2 A → something besides room starves it (read REG_06 and the diff at
the drop).

### Experiment 2b — VOOCPHY_ENABLE 1 vs 0 (c0 kernel)

Same as 2a with `pps_voocphy_enable` = 1, then 0. Restore `pps_voocphy_enable=1`
before `pps_cp_enable=1`.

| prop 49 | vbus (mV) | 2×vbat (mV) | Follows the 8900 request? |
|---|---|---|---|
| 1 | | | |
| 0 | | | |

Verdict: follows only with 0 → ADSP owns VBUS → branch (b). Follows with both →
branch (a). Follows with neither → record; branch (b) with the lift-timeout fallback.

### Dock real_type (today's kernel, no code)

Decision D12 (2026-10-22, after c-1): direct path first, and c2a is written
only if this dock offers 9 V with the Anker upstream. Two ways to settle it:
the Mac on the dock with the Anker upstream (above 15 W on the Anker means the
dock offers 9 V, per the USB PD power rules), or c0's `adapter online` line
with the dock plugged.

| Dock plugged | ADSP USB_REAL_TYPE (prop 11) | GET_PPS_TYPE (prop 32) | VBUS (mV) |
|---|---|---|---|
| Cygnett Unite 7-in-1, Anker upstream (2026-10-22 17:46:52, c0) | **6** (PD, fixed) | 0 | 4833 |

The fixed-PD arming value is 6. The dock is at 5 V because nothing asks for
more. This line does not show whether the dock offers 9 V; that still needs
the Mac test or c2a's own readback (D12). Anker reading on the dock: 7-8 W.

Params restored after c0 run 1: `pps_poll_ms=2000`, `pps_cp_feed_ms=500` (owner, 2026-10-22 17:46).

Params restored or tablet rebooted: yes (2026-10-22, after Experiment 2a: unplug, 3000/3000/8400, then pps_cp_enable=Y; verified by readback)

## c1.5 — cable rating

Cable label: **240 W** (2026-10-22, owner). The USB PD spec requires a
5 A e-marker on every 240 W (EPR, 48 V × 5 A) cable, so this is a 5 A
cable by its rating. **Confirmed by the source:** c0's `adapter online`
line for the Anker direct reads `real_type 8 pps_type 1 pps_max_curr 5000`
(2026-10-22 17:40:36). The Anker offers 5 A on this cable. c1.5 = **5 A**.
The 3000 mA rule in the safety contract is lifted for this cable only.

## c2a — `10-pd9v`

| Check | Result |
|---|---|
| Anker watts on the dock (≥ 17 W) | |
| ADSP vbus × ibus cross-check | |
| 5 V revert on unplug | |
| Induced failure → one retry → hold 5 V (log lines) | |
| VOOC brick still identifies | |
| 10 plug cycles: no DRM hotplug/blank, lsusb identical | |
| At 80 %: stays 9 V, bypass watts | |

## c2b — negotiate-time 3 A (`pd_iclmax_ma=3000` before plug-in)

| 10 min at 9 V × 3 A, ADSP VBUS ≥ 8.5 V, no collapse | Anker watts |
|---|---|
| | |

## c3 — `20-cp-current-ladder`

Patches: `0001` (c0, unchanged) + `0002` (the pump controller, D16). It
follows Android's recipe from `docs/vendor-pps.md`:
- **Start:** once the PPS current is at its target, suspend the buck
  (ICL 0), request 2×vbat + 450 mV (prop 34, then prop 35), and engage when
  the window opens. Allow `pps_cp_lift_ms` 5000.
- **While engaged:** feed every 300 ms. Servo the PPS voltage on the pump's
  ibus: +40 mV below target − 100 mA, −40 mV above target + 100 mA, −100 mV
  above target + 500 mA. Never step up at diff ≥ 650 mV, and step down while
  ibat > 9000 mA. Every voltage change is a 34 + 35 pair.
- **The guard:** a drop, a failed entry, a failed pump ADC read, 4 polls
  under 300 mA (after a 2 s settle), taper, or `pps_cp_enable=0` all resume the
  buck in the same poll. Each counted stop holds off 10 s; 5 attempts per plug.
- **The pump ADC stays on** for the whole session (the Experiment 4 bug).
- Defaults ship at 3000 mA. New params: `pps_cp_buck_suspend`,
  `pps_cp_entry_margin_mv`, `pps_cp_lift_ms`, `pps_cp_rearm_ms`,
  `pps_cp_rearm_max`, `pps_cp_vstep_over_mv`, `pps_cp_ibus_band_ma`,
  `pps_cp_diff_max_mv`, `pps_cp_ibat_max_ma`, `pps_cp_min_ibus_ma`,
  `pps_cp_settle_ms`. Gone: `pps_cp_target_mv`, `pps_cp_min_volt_mv`.
  Changed defaults: `pps_cp_vstep_mv` 50 → 40, `pps_cp_feed_ms` 500 → 300.

Runs: (1) defaults (3000 mA); (2) `pps_target_curr_ma` and `pps_max_curr_ma`
= 4000; (3) 5000. Each: Anker direct, trace on before the plug, 3 minutes,
Anker watts, save the trace. Restore to 3000 after each run.

Pass: the pump holds for the whole run, ibus within 100 mA of the target, the
Anker reads about target × ~8.2 V, and no buck-off sample without the pump engaged
beyond the 5 s entry. Open question the trace answers: does the bus follow
34 + 35 pairs after entry (set vs ADSP vbus)?

| Run | Target (mA) | Pump held (min) | ibus (mA) | Pack (mA) | Anker watts | Exits clean (80 %, unplug) |
|---|---|---|---|---|---|---|
| 1 (build 3, 2026-10-22 21:37) | 3000 | **never engaged** | 20 | discharging ~1.2 A | **0.1 W for ~5.5 min** | unplug: clean |
| 1b (build 4, 22:18) | 3000 | **6.6 (394 s), no drop** | 2919 median (2257-3001 after 20 s) | 5004 median | **28** | unplug: drop at diff −41 → buck resumed → detach exit, clean |
| 2 (build 4, 22:31) | 4000 | **3.9 (236 s), no drop** | 2895 median, max 3440 | 4967 median, max ~6200 | **32-33** | params restored by readback |

Runs 1b and 2 (traces `~/c3-run1b.trace`, `~/c3-run2.trace`), battery 55 → 62 %:
- **The recipe works.** The pump engaged 0.5 s after the entry request in both
  runs and stayed engaged with the buck suspended; REG_06 = 4 / REG_07 = 128
  throughout.
- **The guard held:** one `buck_off` sample without the pump per run, the
  ~0.3 s between the entry request and the engage. The unplug in 1b went
  drop → buck resumed → detach exit.
- **Pack temperature 32-33 °C in both runs.**
- **Run 2 never reached 4000 mA:** the servo walked the setpoint to the
  `pps_cp_max_volt_mv` ceiling (9500) within ~20 s and stopped there. At
  9.5 V requested, the bus at the tablet was ~8.84 V, diff ~440 mV (well under
  `pps_cp_diff_max_mv` 650) and ibus ~2.9 A, so the 9500 ceiling is the limit,
  not the pump. The Anker's 32-33 W is above 9.5 V × 2.9 A ≈ 27.5 W; the pump's
  ibus ADC and the Anker display differ by more than in run 1b (28 W at the same
  median ibus). Not resolved: the Anker figure is read by eye.
- The vendor's ceiling is 11000 − 500 = 10500 mV (`docs/vendor-pps.md`);
  the SC8547A's VBUS_OVP is 10 V at the chip.

**Run 1 failed the guard.** At t=238.4 the entry ran ("buck suspended,
request 8640 mV at 3000 mA"). The Anker followed the 34 + 35 pair: ADSP vbus
went from ~8.4 to 8.75-8.83 V, and chip diff was 657 mV, inside the window.
But with the buck suspended the pack was briefly running the tablet, and its
`current_now` went negative. `pps_read_battery()` (from before this lab)
treated any negative reading as an error (`if (... i < 0 ...) return -EIO`),
so every CHARGING poll returned before `pps_cp_step()`: no engage, no 5 s
entry timeout, no temp/vbat exits. The buck stayed suspended until the owner
unplugged at t=569 (the trace shows `cp=0 buck_off=1` at a steady 320 ms
cadence, with stale pack readings). Battery 58 %, 33 °C; no harm beyond the
discharge.

Fix (build 4, `0002` updated, commit `5c57b20`):
1. **Read fix:** `pps_psy_int()` returns the errno separately, so a negative
   current is a reading.
2. **Read-failure guard:** a failed pack read in CHARGING/TAPER releases
   the pump and resumes the buck.
3. **Backstop:** `pps_cp_backstop()` runs after every poll, outside all of
   `pps_poll()`'s early returns. A suspended buck must have an engaged pump fed
   within 1000 ms, or be inside the entry window + 1 s; otherwise the buck is
   resumed at once.

Also learned: **the 34 + 35 pair moved the bus** after entry (8.4 → ~8.75 V
for an 8640 request, unloaded). That answers the open question for the
unloaded case.

## c4 — `30-pr-final`

24 h daily use, both setups: _notes_. PR: _link_.
