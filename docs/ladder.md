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
| Dock | _model:_ |
| Cable brick→tablet | _rating: unknown until c-1 / c1.5_ |

## c-1 — Android ground truth (no Linux code)

Boot Android. Battery below 75 %. Same Anker, same cable, same dock.

| Setup | Battery % | Anker watts | VBUS (mV) | IBUS (mA) | Source of VBUS/IBUS |
|---|---|---|---|---|---|
| Android, through the dock | | | | | |
| Android, direct | | | | | |

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
| | | | | |

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

Params restored or tablet rebooted: _yes/no_

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
