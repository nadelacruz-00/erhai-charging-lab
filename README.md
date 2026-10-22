# erhai-charging-lab

A test lab for USB-C charging on the OnePlus Pad 3 (`erhai`, Snapdragon 8 Elite
/ SM8750) running Ubuntu on the mainline port at
[TheMojoMan/oneplus-erhai-linux](https://github.com/TheMojoMan/oneplus-erhai-linux).

On that port, a USB-C dock charges at 8-9 W (nothing asks the ADSP for a
fixed 9 V PDO) and a PPS brick direct stops at 24-25 W (the SC8547A charge pump
engages and is dropped 500 ms later). The lab measures first: Android's own
numbers with the same charger, then a trace of the pump, then one candidate
kernel per fix, each with the numbers it must produce. Only the candidate that
passes its rung becomes a PR to the port.

## How it works

- **CI** (`.github/workflows/kernel.yml`) builds each directory under
  `candidates/` from the pinned commit in `candidates/BASE_SHA`, with the
  tree's own `build-kernel.sh`, the candidate's patches and `extra.config`,
  and a `-lab-<candidate>` kernelrelease. Every requested config symbol is
  asserted in the final `.config`. The result is a Release: a UKI, a modules
  tarball, the `.config` and `SHA256SUMS`.
- **`erhai-lab`** (`tablet/erhai-lab`, stdlib-only python3) runs on the
  tablet. `erhai-lab fetch <candidate>` downloads a release, verifies every
  checksum and the UKI/modules version match, and installs the kernel beside
  the package kernel. rEFInd's default is never changed.

## Ladder

`docs/ladder.md`: c-1 Android ground truth, c0 trace, c1 experiments,
c1.5 cable, c2 dock (fixed PD 9 V), c3 charge pump, c4 PR.

## Tests

    python3 -m unittest discover -s tests -v

## License

GPL-2.0-only.
