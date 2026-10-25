# Bosgame M5 / BeyondMax (AXB35-02) — hardware-validated fan curve tuning

A systemd oneshot unit that applies tuned fan-curve thresholds to `fan1` and `fan2` at boot,
plus the measurements behind them. Everything below was hardware-tested on a
**Bosgame M5 / BeyondMax** (Sixunited **AXB35-02** board, AMD Ryzen AI Max+ 395 "Strix Halo",
BIOS 1.07, 128 GB LPDDR5) running Ubuntu with kernel 7.0.0-38-generic.

## Why

The driver's default thresholds (`rampup 60,70,83,95,97`) are too lazy for this machine: under
sustained all-core load the die reaches **93–99 °C** and holds there, because the stock EC
behavior ramps the fans late and the chip's power envelope runs hot. The unit in this directory
applies an aggressive curve that drives fans to maximum under load and steps down cleanly
afterwards.

## The curve

Levels are the EC's own: 0=0%, 1=20%, 2=40%, 3=60%, 4=80%, 5=100%.

| EC temp rises past | Level | Duty | Note |
|---|---|---|---|
| 45 °C | 1 | 20% | never fully stops — avoids spin-cycling at idle |
| 58 °C | 2 | 40% | |
| 65 °C | 3 | 60% | |
| 72 °C | 4 | 80% | sustained maximum, see the 80% note below |
| 83 °C | 5 | 100% | emergency rung only |

Down-steps use `rampdown_curve = 40,53,57,67,78` — each threshold 5–8 °C below the matching
up-threshold. The down-threshold for level 3 was deliberately widened (60 → 57 °C) after
hardware testing showed the EC sensor's ±2–3 °C noise caused a level 2↔3 bounce
(~800 rpm swing) in the 60–65 °C band.

**Why 80% is the practical ceiling:** independent measurements on the same board
([crmaris/FanControl.AXB35](https://github.com/crmaris/FanControl.AXB35)) showed that at 100%
duty the board cannot keep both fans spinning once hot — one repeatedly collapses to
~1,400–2,000 rpm — while 80% holds both at ~4,000 rpm, moving more combined airflow than 100%.
Our own testing saw both fans hold ~4,700 rpm at 100%, but the curve still treats 100% as an
emergency rung (≥83 °C) rather than a sustained state.

## What the EC temperature register actually reads

The curve's only input is EC register `0x70` (what `temp1/temp` exposes). Cross-checking
against kernel sensors on this machine:

- During GPU inference (97% GPU load): EC read 87–98 °C while `k10temp` read 60–67 °C and the
  amdgpu edge sensor 60–65 °C — the EC sensor sits on the die's GPU-side hot spot.
- During all-core CPU load: EC and `k10temp` agree (93–98 °C).
- It lags the instantaneous sensors and reads high: fans stay elevated a minute or two after
  load ends. That is sensor behavior, not a control failure.

Practical consequence: the curve reacts to GPU heat correctly without any GPU-load input —
temperature is the integrated signal. Do not add a GPU-clock input; a GPU can sit at maximum
clock with almost no load and produce almost no heat.

## Measured results (32-thread all-core load, ~200 s per run)

The vendor speed-mode switch cycles the EC power profile (`apu/power_mode`). Verified per profile:

| EC profile | Die temp under load (k10temp) | Fan behavior |
|---|---|---|
| quiet | 65–67 °C (power-limited) | level 3, ~3,380 rpm — correct, no more needed |
| balanced | 93–97 °C | level 2→4→5 within ~20 s, held ~4,690/4,750 rpm at 100%, no fan collapse |
| performance | 98–99 °C pinned | level 5 within 10 s, held ~4,690/4,750 rpm |

Step-down after load end: 98 → 56 °C in ~90 s, levels 5→3→2, with the widened hysteresis
eliminating the bounce. The die sitting in the high 90s under full load *at maximum fans* is the
chip's power envelope, not a cooling shortfall — the same conclusion the
[Hardware Busters article](https://hwbusters.com/systems/bosgame-m5-fan-control-broken-as-shipped-so-i-fixed-it-with-a-free-plugin/)
reached on this board.

## Install

Requires the `ec_su_axb35` module from this repository to be built, installed and loaded
(see the main README).

```bash
sudo cp contrib/bosgame-m5/ec-su-axb35-curve.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now ec-su-axb35-curve.service
```

`enable --now` applies the curve immediately and re-applies it at every boot. Re-applying at
boot matters: the driver's thresholds reset to the defaults on every module reload, and the EC
may retain curve mode across reboots — without the unit you end up with curve mode running the
lazy default thresholds.

Uninstall:

```bash
sudo systemctl disable --now ec-su-axb35-curve.service
sudo rm /etc/systemd/system/ec-su-axb35-curve.service
```

The driver resets all fans to firmware AUTO on unload, so removing the module also returns
the machine to stock behavior.

## Related work

- [cmetz/ec-su_axb35-linux](https://github.com/cmetz/ec-su_axb35-linux) — this driver
- [crmaris/FanControl.AXB35](https://github.com/crmaris/FanControl.AXB35) — Windows-side fix;
  source of the EC register map and the 80% cap measurements
- [nathanmarlor/strix-halo-fan-control](https://github.com/nathanmarlor/strix-halo-fan-control) —
  userspace alternative and reverse-engineering writeup

## License

GPL-2.0, the same license as this repository.
