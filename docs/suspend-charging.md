# Charging while suspended (opened 2026-10-26)

Owner report: "my device won't charge when suspend is active". Kernel at the
time: package `7.3.0-rc5-sm8750-themojoman+b34` (contains the merged
erhai_pps pump work from PR #2). Chargers on hand: Anker 737 (PPS), 100 W PD
brick, Cygnett dock, and now a **genuine SuperVOOC brick**.

## What the logs show so far

- No log on a b31+ kernel catches a suspend with a charger attached. The
  newest suspends with a charger attached are from 2026-10-21 (sleep-fix lab
  kernels, before the pump code).
- Boot -2 (10-25): long suspend 18:11:50 → 20:43:18 with **no** charger
  (`charger removed` 17:51:54); then awake on battery until a low-battery
  power-off at 02:20:52 (`01:10 Running on LOW Battery`).
- Boot -1 (10-26) started at 10:49 with the pack at 2 %. Its first three
  seconds are stamped 02:21 because the clock was stale until
  `rtc-pm8xxx: setting system clock to 2026-10-26T02:49:44 UTC`. There is no
  8.5 h gap.

## Root cause hypothesis (static read, not yet reproduced)

PPS path (Anker, PD brick):
- `erhai_pps.c` has no suspend handling: no PM notifier, no wakeup source.
  Its poll is an ordinary `schedule_delayed_work()` timer, which does not run
  while the system sits in s2idle.
- While the pump runs, the buck is suspended (ICL 0) and the SC8547A must be
  fed within its 1000 ms watchdog (`REG_09 0x13`, `sc8547a.c:913/948`).
- Suspend → no poll → the pump's watchdog expires and CHG_EN clears, while
  the ADSP input stays suspended. `pps_cp_backstop()` only runs from the poll,
  so nothing charges until resume, when the poll restarts.

SuperVOOC path (new brick), same class of risk:
- Delivery and the watchdog feed run from `sc8547_attach_work`
  (`schedule_delayed_work`, `sc8547a.c:2406/2467`), also a plain timer, so they
  also pause in s2idle. What the adapter and the ADSP do once the pump stops
  is unknown.

## Reproduction (to run)

Battery below 80 %, bypass off:

1. PPS: Anker direct; wait for ~39 W (pump engaged). Note the battery %.
   `sudo rtcwake -m mem -s 300` (mem_sleep is s2idle). Watch the Anker
   display while suspended. After wake: battery % and `journalctl -k -b`.
2. Control: the same with `pps_cp_enable=0` first (buck only; restore to 1
   after).
3. SuperVOOC: the same with the SuperVOOC brick (no display: compare the
   battery % before and after, plus the log).

Pass/fail reading: while suspended, the Anker reads ~0.1 W and the pack does
not gain → hypothesis confirmed for PPS. Buck-only control gains → the fix
belongs in erhai_pps.

## Fix options (decide after the reproduction)

- A) Hand off to the buck before suspend: a PM notifier
  (`PM_SUSPEND_PREPARE`) drops the pump and writes ICL unsuspend, so the ADSP
  charges on the buck while asleep (~17-25 W). `PM_POST_SUSPEND` re-arms the
  pump. Same idea for the SuperVOOC delivery.
- B) Android's way: hold a wakeup source while fast charging, so the system
  does not suspend while the pump runs.
