# piHPSDR NNR Patch

This patch ports **NNR (Neural Noise Reduction)** from WDSP 2.10 to
[piHPSDR](https://github.com/dl1ycf/pihpsdr), which is still on WDSP 2.00.

NNR was introduced by Warren Pratt (NR0V) in the official WDSP repository
[TAPR/OpenHPSDR-wdsp](https://github.com/TAPR/OpenHPSDR-wdsp/tree/master/wdsp%202.10).
The sister project [deskhpsdr](https://github.com/dl1bz/deskhpsdr) was the
first HPSDR program to integrate WDSP 2.10
(see [discussion #207](https://github.com/dl1bz/deskhpsdr/discussions/207)),
and served here as the practical template for the integration. The actual
NNR source files were verified against the official TAPR repository: the
two embedded model weight files (`nnr_model_0.c`, `nnr_model_1.c`) are
byte-identical to the TAPR originals, and `nnr.c`/`nnet.c`/`nnio.c` differ
from deskhpsdr's copy only in formatting (deskhpsdr ran the entire WDSP
tree through a code formatter), not in logic.

NNR is a new, network-based noise reduction model and is **not** the same
as the existing NR2/NR3/NR4 algorithms. It is added to piHPSDR as a fifth,
independent reduction mode ("NNR"), with two selectable models ("Standard"
and "Premium") and an adjustable mask floor.

This patch does **not** upgrade the entire WDSP tree to 2.10 (the diff
between the two WDSP trees touches almost every file, but is overwhelmingly
pure reformatting, not functional change). Instead, only the new,
self-contained NNR source files (including the two embedded model weights)
were taken over and wired into piHPSDR following the existing pattern used
for NR3 (`rnnr`) / NR4 (`sbnr`).

**Performance note:** According to the deskhpsdr developer, NNR increases
CPU load significantly; it has only been tested there on a Raspberry Pi 5
(or more powerful systems). On older/weaker Raspberry Pi models, NNR may be
too slow for real-time operation. On a regular desktop PC/laptop this
should not be an issue.

## Prerequisites

- A fresh clone of <https://github.com/dl1ycf/pihpsdr>, ideally at `master`
  commit `f9ab6ee5` ("Dither fix for P2 DIVERSITY.") or a later state. The
  patch was created against exactly this commit and tested against a fresh
  checkout.
- The usual piHPSDR build dependencies (see the piHPSDR README), in
  particular `fftw3`, since NNR uses FFTW internally — this is already a
  piHPSDR dependency anyway.
- `git` (for `git am`) or `patch` (for `patch -p1`).

## Applying the patch

```bash
git clone https://github.com/dl1ycf/pihpsdr.git
cd pihpsdr

# Option 1 (recommended): apply as a commit
git am /path/to/0001-Add-NNR-Neural-Noise-Reduction-WDSP-2.10-support.patch

# Option 2: just change the files, without a commit
patch -p1 < /path/to/0001-Add-NNR-Neural-Noise-Reduction-WDSP-2.10-support.patch
```

The patch is fairly large at about 42 MB, because the two embedded NN
models (`nnr_model_0.c` "Standard", `nnr_model_1.c` "Premium") are included
in source code as byte arrays — that is expected.

## Building

```bash
make -j$(nproc)
```

This first builds `wdsp/libwdsp.a` (now including `nnr.o`, `nnet.o`,
`nnio.o`, `nnr_model_0.o`, `nnr_model_1.o`) and then the piHPSDR binary as
usual.

## Usage

In the Noise menu (noise reduction dialog), the "Reduction" selector now
offers **NNR** in addition to NONE/NR/NR2/NR3/NR4. The "NNR Settings" radio
button gives access to the model choice (Standard/Premium) and the mask
floor (dB). The setting is persisted per operating mode as well as in the
piHPSDR properties, exactly like NR2/NR4.

## Scope of the patch

- `wdsp/`: new files `nnr.c/h`, `nnet.c/h`, `nnio.c/h`, `nnet_profile.h`,
  `nnr_model_0.c`, `nnr_model_1.c` (author of the original WDSP files:
  Warren Pratt, NR0V), plus wiring in `RXA.c/RXA.h`, `comm.h`, `wdsp.h` and
  `Makefile`. The `RXAbp1Check` signature gained a new `nnr_run` parameter,
  updated accordingly in `amd.c`, `anf.c`, `anr.c`, `emnr.c`, `rnnr.c`,
  `sbnr.c`, `snb.c`.
- `src/`: `receiver.h/.c`, `profiles.h/.c` (new mode `nr=5`, parameters
  `nnr_model`, `nnr_mask_floor`), `noise_menu.c` (new "NNR Settings" page),
  `client_server.h/.c`, `client_thread.c`, `server_thread.c` (client/server
  remote-operation protocol extended with the new fields).

## Tested

- `wdsp/` and the complete piHPSDR binary build without errors.
- The patch was applied against a fresh clone of piHPSDR (commit
  `f9ab6ee5`) via `git am` and built successfully there as well.
- The built binary starts up and goes through the normal hardware
  discovery sequence without crashing.

**Not tested:** NNR in actual operation with real hardware (audio quality,
real-world CPU load) and hands-on use of the new menu entry in the GUI.

## License

piHPSDR and WDSP are licensed under the GPL. The NNR source files and the
model weights they contain are unchanged in substance from the official
[TAPR/OpenHPSDR-wdsp](https://github.com/TAPR/OpenHPSDR-wdsp/tree/master/wdsp%202.10)
repository (obtained here via deskhpsdr's reformatted copy) and are
(C) 2026 Warren Pratt, NR0V, likewise under the GPL.
