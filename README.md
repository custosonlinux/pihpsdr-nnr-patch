# piHPSDR NNR Patch

Dieser Patch portiert **NNR (Neural Noise Reduction)** aus WDSP 2.10
(implementiert im Schwesterprojekt [deskhpsdr](https://github.com/dl1bz/deskhpsdr),
siehe [Diskussion #207](https://github.com/dl1bz/deskhpsdr/discussions/207))
nach [piHPSDR](https://github.com/dl1ycf/pihpsdr), das noch auf WDSP 2.00 basiert.

NNR ist ein neues, netzwerkbasiertes Rauschunterdrückungsmodell und **nicht**
dasselbe wie die vorhandenen NR2/NR3/NR4-Verfahren. Es wird in piHPSDR als
fünfter, unabhängiger Reduction-Modus ("NNR") ergänzt, mit zwei wählbaren
Modellen ("Standard" und "Premium") und einem einstellbaren Mask Floor.

Es wurde **nicht** das komplette WDSP auf 2.10 angehoben (der Diff zwischen
den beiden WDSP-Bäumen betrifft fast jede Datei, ist aber überwiegend reines
Reformatting von deskhpsdr, keine funktionale Änderung). Stattdessen wurden
nur die neuen, in sich abgeschlossenen NNR-Quelldateien (inkl. der beiden
eingebetteten Modell-Gewichte) aus `deskhpsdr/wdsp-2.10` übernommen und nach
dem bereits vorhandenen Muster von NR3 (`rnnr`) / NR4 (`sbnr`) in piHPSDR
eingebunden.

**Hinweis zur Performance:** Laut dem deskhpsdr-Entwickler erhöht NNR die
CPU-Last deutlich; es wurde dort nur auf einem Raspberry Pi 5 (bzw. potenteren
Systemen) getestet. Für ältere/schwächere Raspberry-Pi-Modelle ist NNR
möglicherweise zu langsam für Echtzeitbetrieb. Auf einem normalen
Desktop-PC/Laptop sollte das kein Problem sein.

## Voraussetzungen

- Ein frischer Klon von <https://github.com/dl1ycf/pihpsdr>, idealerweise auf
  `master` beim Commit `f9ab6ee5` ("Dither fix for P2 DIVERSITY.") oder einem
  späteren Stand. Der Patch wurde gegen genau diesen Commit erstellt und
  gegen einen frischen Checkout getestet.
- Die üblichen piHPSDR-Build-Abhängigkeiten (siehe piHPSDR-README), insbesondere
  `fftw3`, da NNR intern FFTW verwendet — das ist aber ohnehin schon eine
  piHPSDR-Abhängigkeit.
- `git` (für `git am`) oder `patch` (für `patch -p1`).

## Patch anwenden

```bash
git clone https://github.com/dl1ycf/pihpsdr.git
cd pihpsdr

# Variante 1 (empfohlen): als Commit übernehmen
git am /pfad/zu/0001-Add-NNR-Neural-Noise-Reduction-WDSP-2.10-support.patch

# Variante 2: nur die Dateien ändern, ohne Commit
patch -p1 < /pfad/zu/0001-Add-NNR-Neural-Noise-Reduction-WDSP-2.10-support.patch
```

Der Patch ist mit ca. 42 MB recht groß, weil die beiden eingebetteten
NN-Modelle (`nnr_model_0.c` "Standard", `nnr_model_1.c` "Premium") als
Byte-Arrays im Quellcode enthalten sind — das ist normal.

## Bauen

```bash
make -j$(nproc)
```

Das baut zunächst `wdsp/libwdsp.a` (jetzt inklusive `nnr.o`, `nnet.o`,
`nnio.o`, `nnr_model_0.o`, `nnr_model_1.o`) und anschließend das piHPSDR-Binary
wie gewohnt.

## Benutzung

Im Noise-Menü (Rauschunterdrückungs-Dialog) steht in der "Reduction"-Auswahl
jetzt zusätzlich zu NONE/NR/NR2/NR3/NR4 der Eintrag **NNR** zur Verfügung.
Über den Radio-Button "NNR Settings" lassen sich das Modell (Standard/Premium)
und der Mask Floor (dB) einstellen. Die Einstellung wird pro Betriebsart
(Mode) sowie in den piHPSDR-Properties persistiert, genau wie bei NR2/NR4.

## Umfang des Patches

- `wdsp/`: neue Dateien `nnr.c/h`, `nnet.c/h`, `nnio.c/h`, `nnet_profile.h`,
  `nnr_model_0.c`, `nnr_model_1.c` (Autor der WDSP-Originaldateien:
  Warren Pratt, NR0V), plus Anbindung in `RXA.c/RXA.h`, `comm.h`, `wdsp.h`
  und `Makefile`. Die `RXAbp1Check`-Signatur hat einen neuen `nnr_run`-Parameter
  bekommen, entsprechend angepasst in `amd.c`, `anf.c`, `anr.c`, `emnr.c`,
  `rnnr.c`, `sbnr.c`, `snb.c`.
- `src/`: `receiver.h/.c`, `profiles.h/.c` (neuer Modus `nr=5`, Parameter
  `nnr_model`, `nnr_mask_floor`), `noise_menu.c` (neue "NNR Settings"-Seite),
  `client_server.h/.c`, `client_thread.c`, `server_thread.c` (Remote-Protokoll
  für Client/Server-Betrieb um die neuen Felder erweitert).

## Getestet

- `wdsp/` und das komplette piHPSDR-Binary bauen fehlerfrei.
- Patch wurde gegen einen frischen Klon von piHPSDR (Commit `f9ab6ee5`) via
  `git am` angewendet und dort ebenfalls erfolgreich gebaut.
- Das gebaute Binary startet und durchläuft den normalen
  Hardware-Discovery-Ablauf ohne Absturz.

**Nicht getestet:** NNR im laufenden Betrieb mit echter Hardware
(Audioqualität, tatsächliche CPU-Last) und die Bedienung des neuen
Menüpunkts in der GUI.

## Lizenz

piHPSDR und WDSP stehen unter der GPL. Die NNR-Quelldateien und die darin
enthaltenen Modellgewichte stammen unverändert aus deskhpsdrs WDSP-2.10-Fork
und sind dort (C) 2026 Warren Pratt, NR0V, ebenfalls unter der GPL.
