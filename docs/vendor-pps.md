# How Android runs PPS + charge pump on erhai (vendor source)

Source: `OnePlusOSS/android_kernel_modules_and_devicetree_oneplus_sm8750` @ `e888579`
(OPD2415_15.0.1.701). Paths are short: `pps.c` =
`vendor/oplus/kernel/charger/v2/oplus_chg_pps.c`, `adsp.c` =
`vendor/oplus/kernel/charger/v2/charger_ic/oplus_hal_adsp.c`, `sc8547a_hal.c` =
`vendor/oplus/kernel/charger/v2/ufcs_ic/oplus_hal_sc8547a.c`, DT under
`kernel_platform/qcom/opensource/devicetree/oplus/oplus_chg/erhai/`.

Gathered 2026-10-22 by a research pass. Lines marked **(checked)** were
re-read by hand. The rest come from that pass and get checked when c3 relies on them.

## The ADSP only carries the PDO request

- PPS on erhai uses `virtual_cp`: two SC8547A on AP-side I2C, driven by
  `sc8547a_hal.c` (DT `"oplus,sc8547a"`), plus `adsp_mos`
  (erhai-common.dtsi:380-397, 1106-1108, 1642-1664).
- `USB_PPS_CP_MODE_INIT` (40) and `USB_PPS_VOOCPHY_ENABLE` (49) are never
  written in `adsp.c` **(checked: no write in the file)**. Prop 39 is the
  `adsp_mos` bypass CP (adsp.c:9906-9926). The Linux engine writes 49, 40 and
  39 on entry; Android does not on this path.
- **Every PDO request writes prop 34 then prop 35, back to back**
  (`oplus_chg_set_pps_config`, adsp.c:10302-10303 **(checked)**). Android
  never sends 34 alone. The Linux CP servo does: it writes only 34 while the
  current is already at target.

## The buck is suspended while the pump runs

- PPS votes `WIRED_CHARGE_SUSPEND` before its first PDO request
  (pps.c:1819-1820 **(checked)**). That ends in `USB_INPUT_CURR_LIMIT = 0`
  (adsp.c:6294-6295 **(checked)**; unsuspend writes 0xFFFFFFFF). The vote is
  released only on exit (pps.c:1600-1601, 1618-1619).
- This matches Experiments 3-5: with the buck allowed 3 A, it takes the whole supply.

## Start sequence

1. Set the CP to 2:1, ADC on (pps.c:1781-1798), then request 5500 mV / 1500 mA
   (pps.c:1822 **(checked)**).
2. Every 500 ms: target = `floor100(2 × cp_vout) + 450` mV (pps.c:1881-1887,
   1931). Voltage steps of 2 V / 1 V / 500 / 200 / 100 mV (pps.c:2020-2035) at
   1500 mA (pps.c:2077).
3. Enable the pump only when `cp_vin` is in [target, target + 200]
   (pps.c:1940, 2001-2014); confirm 500 ms later.

## Charging loop

- `current_work` every 200 ms while changing, 500 ms when settled
  (pps.c:3543-3598). Whenever voltage or current changes:
  `cp_set_iin`, then `pdo_set(vbus, curr)` (pps.c:3584-3592 **(checked)**).
- Third-party adapters: current goes straight to the target, no ramp
  (pps.c:3559-3561).
- Voltage servo on the pump's ibus: +40 mV while ibus is below
  target − 100 mA, −40 mV above target + 100, −100 mV above target + 500;
  80 mV deadband (pps.c:3281-3283 **(checked)**, 3242-3252, 3310-3361). Window:
  `2 × vbat + 100` to `11000 − 500` mV (pps.c:3299-3306).
- Exits: 4 results under 300 mA (abnormal, recovery allowed); 10 over-current
  results; end of the curve hands off to the buck (pps.c:3325-3353, 3451-3455).

## Current target on this SKU

- `max_curr = power × 1000 / 11000`: 33 W → ~3000 mA, 55 W → 5000 mA
  (pps.c:1735-1774; oplus-chg-24976.dtsi:8, 14-19 — 55 W when region 0xA7;
  oplus-chg-24976-eu.dtsi:8 — 55 W). `curr_max_ma = 5000`.
- Android on this unit measured 46-47 W screen off (c-1), which fits the 55 W
  cap and ~5 A.

## SC8547A ADC

- The ADC stays on for the whole session; it is turned off only on exit, on a
  failed start and at shutdown (sc8547a_hal.c:1075, 2051, 1147-1148, 2958;
  pps.c:1594, 1614, 1993). The Linux port's `sc8547_reset_voocphy()`
  (`sc8547a.c:799`) turns it off on every pump drop (Experiment 4).
