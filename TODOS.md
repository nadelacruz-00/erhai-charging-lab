# TODOS

## Charging (erhai_pps / lab)

### c3.5: drive the slave SC8547A in the PPS path (5 A, ~45 W)

**What:** Run the slave pump in parallel with the master in `erhai_pps`'s pump path (the port's VOOC path already has slave engage/disengage), then retry 5000 mA.

**Why:** At 3.76 A one pump already needs diff ~600 mV (c3 run 2b); 5 A extrapolates to ~800 mV, past the 700 mV drop line. Android's `virtual_cp` runs both SC8547A (docs/vendor-pps.md) and reaches 46-47 W screen off (c-1).

**Context:** docs/ladder.md c3 run 2b; `sc8547a.c` slave section (`SC8547_SLAVE_ENGAGE_IBUS_MA`, `slave_settle_ms`).

**Effort:** M
**Priority:** P2
**Depends on:** c4 merged

### Open questions from the c3 runs

- Anker display vs trace: run 2 showed 32-33 W where ibus × set ≈ 27.5 W (run 1b agreed at 28 W). Check the pump's ibus ADC scale against a USB-C meter.
- One plug-in took 2 min 48 s to report PD_PPS (`real_type 0` first) after a quick re-plug (Experiment 5). Watch for a repeat in daily use.
- The engine still writes props 49/40/39 on entry; Android does not on this path (docs/vendor-pps.md). Try `pps_voocphy_enable=0 pps_cp_init=0` and compare.
- Dock 9 V question (D12) is still open: the Mac + dock + Anker watts test, or c2a's own readback.

### Cache the battery power_supply lookup in erhai_pps

**What:** Resolve `pps_bat_psy()` once (cache the `struct power_supply *`, drop the per-call `power_supply_get_by_name` / put pair).

**Why:** `erhai_pps.c:420-426` looks the battery up by name on every read (two lookups with a fallback). Harmless at the production 2 s poll; at the lab's `pps_poll_ms=100` it is 10-20 lookups a second for nothing. Fold into c4 only if the c0 trace timestamps show polls slipping at 100 ms.

**Context:** Found in the 2026-10-22 eng review of the charging-fixes design (Performance #1). Cache on the first successful lookup; the bq27z561 gauge can probe after the engine, so a NULL must stay retryable; put the reference on module exit.

**Effort:** S
**Priority:** P3
**Depends on:** None

### Standard-PD (UCSI) path for the fixed-PD dock request

**What:** After c2a passes, probe why `ucsi_glink` returns no partner PDOs on sm8750 (`port0-partner usb_power_delivery_revision=0.0`, no `source-capabilities/`; the quirk table at `drivers/usb/typec/ucsi/ucsi_glink.c:430-446` has no sm8750 entry) and whether `charger_pd` honours a UCSI power-level request; if it does, re-base the 9 V request on the mainline `usb_power_delivery` path instead of the Oplus property numbers.

**Why:** Upstream-shaped dock fix with no vendor property IDs; the charger's and dock's offers become a `cat` of sysfs. Approach C in the design doc, rejected for now because partner PDOs read empty on this firmware.

**Context:** Design doc `~/.gstack/projects/charging-fixes/nadelacruz-unknown-design-20261022-114844.md`, Approach C and eng-review Scope Challenge. Start with a `GET_PDOS` trace in `ucsi_glink` on a plug event; compare against the working vendor-prop baseline from c2a.

**Effort:** M
**Priority:** P3
**Depends on:** c2a (`10-pd9v`) passing, so there is a measured baseline to compare against
