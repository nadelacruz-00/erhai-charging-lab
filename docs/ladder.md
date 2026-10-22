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
  `pps_max_curr_ma` or `pps_target_curr_ma` is above 3000.** The cable rating
  is unknown until c1.5.
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

| Run | Battery % | Engage (vbus/vbat/diff/ibus) | Drop (vbus/vbat/diff/ibus) | Anker watts |
|---|---|---|---|---|
| | | | | |

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
| | | | |

Bands: ≥ 2800 mA → the pump was delivering and software killed it; 500-2800 mA →
conducting but current-limited (keep the ibus check, revisit `pps_cp_min_ibus_ma`);
< 500 mA → the pump never conducted.

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

| Dock plugged | ADSP USB_REAL_TYPE (prop 11) | GET_PPS_TYPE (prop 32) |
|---|---|---|
| | | |

Params restored or tablet rebooted: yes (2026-10-22, after Experiment 2a: unplug, 3000/3000/8400, then pps_cp_enable=Y; verified by readback)

## c1.5 — cable rating

_3 A / 5 A, and how it was established:_

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

| Cable | Branch | Pump held (min) | Pack (mA) | Anker watts | Exits clean (80 %, unplug) |
|---|---|---|---|---|---|
| | | | | | |

## c4 — `30-pr-final`

24 h daily use, both setups: _notes_. PR: _link_.
