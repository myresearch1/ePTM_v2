# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

ePTM v2 is a Java particle-tracking model that simulates juvenile salmon migrating through the Sacramento-San Joaquin Delta. It extends the Particle Tracking Model from California DWR's DSM2 with fish behaviors, and is calibrated for Chinook salmon. It reads hydrodynamics from a DSM2 tide file (HDF5), which is not in this repo.

## Environment constraints

- **The model only runs on Windows with a 32-bit Java 8 JVM.** `Globals.initialize()` calls `System.loadLibrary("PTM")`, and `PTMHydroInput` depends on its native methods. That library is not in the repo. It is found through `PATH`, which the setup guide points at the DSM2 install (`%DSM2_HOME%\bin;%DSM2_HOME%\vista\bin`).
- The setup guide calls for Java 1.8 with DSM2 8.1.3 or earlier, and Java 1.6 with DSM2 8.2.
- A full model run is not possible from Linux or WSL, and this machine has no `java` on `PATH`.

## Build and run

There is no build system, test suite, linter, or CI. The documented build is Eclipse:

1. Create a Java project rooted at `ptm_SWFSC/`.
2. Add every jar in `ptm_SWFSC/lib/` to the build path.
3. Select a 32-bit JDK.
4. Export a runnable JAR with main class `DWR.DMS.PTM.MainPTM`.

Compile check without Eclipse (not verified here; requires a JDK):

```
cd ptm_SWFSC && javac -cp "lib/*" -d <out_dir> DWR/DMS/PTM/*.java
```

`eptm.jar` and `animateEPTM.jar` are committed prebuilt binaries. **Source edits have no effect on either jar until it is re-exported.**

Run the model (`scripts/runEPTMBasic.py` builds this command):

```
java -Xss5M -Xms512M -Xmx1024M -jar eptm.jar <config.inp> <behavior.h5> <out.h5> <riverBendsDir> <junctionsDir> \
  ChansOrientsBends812.csv Junctions812.csv latVel.csv latEps.csv latdEp.csv latd2Ep.csv VerEps.csv VerdEp.csv
```

The arguments are positional. If a fourth argument is given, `MainPTM` reads all 13, so the set is effectively all-or-nothing. The existing output file is deleted at startup.

## Workflow across files

1. **DSM2 config:** copy `configFiles/DSM2ConfigTemplate.inp` and fill it in, including the `HYDROTIDEFILE` path. `scripts/h5rpkBatch.bat` runs `h5repack -l CONTI` on a tide file to write a contiguous-layout copy.
2. **Inputs:** `scripts/ePTMCreateInputs.R` writes the behavior HDF5 file and the ePTM `.inp` file. It reads a release CSV with a `time` column.
3. **Simulation:** `MainPTM` runs the model, launched by `runEPTMBasic.py`, `ePTMRun.py`, or `ePTMBatch.py`.
4. **Animation (optional):** add a `ptm anim out 15min <file.bin>` line to the `IO_FILE` section, then convert the binary with `scripts/runAnimEPTM.py`, which runs `animateEPTM.jar` (source: `animateEPTM/plotPTM/readEPTMbinary.java`).

Fish do not interact, so large release histories can be split across separate JVM processes. `BehavedParticle` keeps run state in static fields, so each process handles one run.

## Code architecture

All model code is in the single package `ptm_SWFSC/DWR/DMS/PTM/`. Most of it is inherited DWR PTM code. The ePTM-specific work is concentrated in these classes:

- **`MainPTM`:** parses arguments, sets up the environment, and runs the time loop. Each step refreshes hydro data, updates `SmartChannel`s when the hydro step changes, computes `isDaytime`, and calls `updatePosition` on each particle. It hard-codes `Particle.mixCase = 4` and always creates `BehavedParticle`s (`behavior = true`).
- **`Particle`:** the random-walk core. Displacement is the sum of external/internal and deterministic/random terms, computed by overridable `calc[XYZ]Displacement*` and velocity methods.
- **`BehavedParticle`:** the fish behaviors (swimming, holding, orientation, junction routing, mortality). It also handles most behavior-model I/O. A static initializer reads the behavior HDF5 file, and deaths and checkpoint arrivals are written to the output HDF5 file.
- **`SmartChannel`:** a `Channel` subclass, created in `PTMFixedInput`, that supplies channel direction, tide, and bend information to behaviors.
- **`RiverBendsInput`, `JunctionsInput`, `ProfileData`, `LatData`, `VertData`:** load the static CSV geometry and the velocity and mixing profiles.
- **`PTMEnv`, `PTMFixedInput`, `PTMHydroInput`:** the network, the parsed `.inp` configuration, and the native bridge to hydro data.

## Behavior HDF5 settings

The R script writes these settings, and `BehavedParticle` reads them:

- `swimCode`: only 0, 1, and 11 are implemented. 11 is the current behavior set.
- `nodeDecision`: selects the routing model. Only 0 (outflow-based) and 4 (streamline-based junction rule) are implemented, and other values throw an exception, even though the R script's comment says to cycle between 0 and 4.
- `immortal`: 1 disables mortality.
- `channelPars`: one row per channel, keyed by internal channel number. Column order comes from `configFiles/ePTMv2CalibValidParams812.csv` and is read by index, so it must not change.

`filterK` and `tideCountThr` are legacy values that the R script still has to write, because `BehavedParticle` reads them.

## Things to watch for

- **Internal vs. external DSM2 numbering:** users specify releases and checkpoints with *external* node numbers. The R script converts checkpoints to internal numbers using `nodesIntExt_v8_1_2.csv`. `channelsIntExt_v8_1_2.csv` holds the channel mapping.
- **Grid-specific files:** `*812*` and `*_v8_1_2*` files in `configFiles/` belong to the DSM2 8.1.2 grid. A new grid needs new versions of them, and the model must be recalibrated for other systems or species.
- **Scripts are templates:** ALL_CAPS values such as `PATH_TO_...` are placeholders, so the scripts will not run until they are filled in. The setup guide recommends copying them to a study folder outside the repo, with forward slashes and no spaces in paths. `inpDir` in the R script needs a trailing `/`.
- **Duplicate config text:** `ePTMCreateInputs.R` builds the `.inp` content inline, separately from `configFiles/ePTMConfigTemplate.inp`. Changes to one must be copied to the other.

Known defects in the template scripts:

- `ePTMRun.py` uses `setupPath`, which is undefined (it defines `calibPath`), and expects `ePTM.jar`, while the committed file is `eptm.jar`.
- `ePTMBatch.py` launches `ePTMValidation.py`, which does not exist, using Windows-only `start cmd`.
- `ePTMCreateInputs.R` reads `nodesIntExt_v8_1_2` without the `.csv` extension.
- `ePTMCreateInputs.R` parses release times with `mdy_hm`, but `documentation/RunningEPTMv2.docx` describes `dd/mm/yyyy hh:mm`. Month-first is what the code accepts.

## Reference material

- `documentation/RunningEPTMv2.docx`: full install and run tutorial. The `documentation/videos/` folder covers the same steps.
- `documentation/ePTM_Eclipse_setup_02mar20.docx`: Eclipse project setup.
- [Project wiki](https://github.com/cvclcm/ePTM_v2/wiki): model structure and instructions for changing behaviors.
- `Data/`: the telemetry data used for calibration and validation, described in `Data/readme`. Comparing results against it is not automated in this repo.
