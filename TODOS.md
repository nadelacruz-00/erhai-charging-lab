# TODOS

## Charging (erhai_pps / lab)

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
