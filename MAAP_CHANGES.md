# Changes from upstream

Upstream: https://github.com/geoschem/GCClassic, tag `14.8.0` at commit d25e74367c95e0bec0f3a4ee803d359ae80cc32a (2026-09-16), by the Developers of GEOS-Chem, HEMCO, and Related Software (see `AUTHORS.txt`), MIT License (`LICENSE.txt`). The submodules carry the same MIT license and attribution in their own `LICENSE.txt` or `LICENSE` files.

This copy starts from upstream's file tree at d25e743 without its git history. Left out: `.github/` (issue templates and CI workflows, including the compile tests). Apart from the submodule changes and the README note below, the superproject's own files are unchanged.

## Submodules

GEOS-Chem Classic is a superproject; the model code lives in submodules. Two of them point to fixed copies, the others are the upstream repositories at the commits 14.8.0 pins.

| Path | Repository | Commit | Notes |
|---|---|---|---|
| `src/HEMCO` | https://github.com/gchang/hemco | ce04b23 | HEMCO 3.13.0 (upstream 870e880) plus one fix, see its `MAAP_CHANGES.md` |
| `src/GEOS-Chem` | https://github.com/gchang/geos-chem | d3e8f2c | GEOS-Chem 14.8.0 (upstream a4551f9) plus three fixes, see its `MAAP_CHANGES.md` |
| `src/Cloud-J` | https://github.com/geoschem/Cloud-J | 16d18b0 | unchanged |
| `src/HETP` | https://github.com/geoschem/HETerogeneous-vectorized-or-Parallel | df2f942 | unchanged |
| `docs/source/geos-chem-shared-docs` | https://github.com/geoschem/geos-chem-shared-docs | f882128 | unchanged |

Nested submodules are unchanged too: `src/GEOS-Chem/KPP/standalone` (geoschem/KPP-Standalone 6cd8fdb) and `src/HEMCO/docs/source/geos-chem-shared-docs` (f882128).

Get everything with:

```sh
git clone --recurse-submodules https://github.com/gchang/geos-chem-classic.git
```

## 2026-10-25: the fixes, in short

- **HEMCO: long data paths no longer crash the model.** The VOLCANO, PARANOX and LightNOx extensions wrote the file path and a status prefix into 255-character strings, which stopped the model with `Fortran runtime error: End of record` once the data root passed about 140 characters (seen in fullchem dry runs). These strings are now 1023 characters, as in HEMCO's own file reader. Results do not change.
- **GEOS-Chem: `download_data.py` fetches the restart file from the `geoschem+http` portal.** It used to request `https://geos-chem.s3-us-west-2.amazonaws.com/ExtData/GEOSCHEM_RESTARTS/...`, which does not exist (404), so the run directory had no initial conditions file.
- **GEOS-Chem: `download_data.py` exits with an error when a download fails.** It used to exit 0 even after a 404.
- **GEOS-Chem: `createRunDir.sh` skips first-time user registration when its input is not a terminal.** With piped answers, the registration questions used to read those answers and post them to the registration server as name, email and so on (and with no input at all they looped forever). Interactive use and the `GC_USER_REGISTERED=true` opt-out are unchanged.

No change affects the model's results.

Not changed: on machines whose kernel name contains `aws`, `createRunDir.sh` still tries to copy the sample restart with `aws s3 cp --request-payer requester` from `s3://gcgrid`, using whatever AWS credentials the machine has. Without credentials it fails harmlessly and `download_data.py` fetches the restart from the public bucket instead.

Known limit that remains: GEOS-Chem itself builds some input paths (for example `CHEM_INPUTS/Olson_Land_Map_201203/Olson_2001_Drydep_Inputs.nc` and the UCX and photolysis tables) in 255-character strings, so the data root (`GC_DATA_ROOT`) must stay under about 190 characters. With the HEMCO fix, HEMCO no longer sets a lower limit.

## 2026-10-25: README note

One paragraph near the top of `README.md` pointing to this file and giving the clone command above.

## Check

On 2026-10-25, on Linux x86-64 in a fresh conda-forge environment (`gfortran=15 gcc=15 gxx=15 cmake make netcdf-fortran libnetcdf netcdf4 wget pyyaml numpy xarray`; gfortran 15.3.0, netcdf-fortran 4.6.4, CMake 4.4.3), with 4 OpenMP threads, `OMP_STACKSIZE=500m` and an unlimited stack:

1. Fresh recursive clone of this repository (commit 5d62672, before this file was added): every submodule at the commit in the table above.
2. Run directories made by piping the answers into `createRunDir.sh` (`printf '7\n1\n1\n1\n<dir>\n<name>\nn\n'` for 4x5 MERRA-2 TransportTracers, `'1\n1\n1\n1\n1\n...'` for 4x5 MERRA-2 fullchem), with `GC_DATA_ROOT` set to a 190-character path and no `GC_USER_REGISTERED`: registration was skipped with a note, and no registration server was contacted.
3. Build the documented way (`cmake <code dir> -DRUNDIR=<run dir>`, `make -j4`, `make install`): 2 min 11 s, 0.6 GB peak memory, no compiler warnings.
4. Fullchem dry run with the 190-character root: runs to the end and lists the volcano tables with paths of 262 and 267 characters. Upstream 14.8.0 stops in the same dry run with `End of record` (in PARANOX).
5. `python download_data.py log.dryrun geoschem+http`: all TransportTracers inputs (22 files, 555 MB for the 1-hour and 24-hour runs together) and the restart from `https://geos-chem.s3-us-west-2.amazonaws.com/GEOSCHEM_RESTARTS/GC_14.8.0/GEOSChem.Restart.TransportTracers.20190101_0000z.nc4` (md5 3dfb1ff7c89e7b45f727dea4c2fd7330), exit 0. With a file the portal does not have added to the dry-run log, it exits 1; upstream exits 0 in both cases and its restart request gets 404.
6. TransportTracers runs from 2019-01-01 00:00 with the 190-character root, where two input paths are 258 and 261 characters long: 1 hour (hourly diagnostics, as in the integration tests) 12 s and 2.7 GB peak memory; 24 hours (daily means) 77 s and 2.7 GB. Every variable in all 10 output collections and the end restart file is identical to upstream 14.8.0 run with a short data root (394 variables per run).
7. Physical checks on the 24-hour run: the passive tracer's global mass changes by 1.8e-12, dry air mass by -1.0e-7, the radon-222 burden change (-0.0009 kg) equals soil emission minus decay (0.0375 - 0.0383 kg), and the modeled decay constant is 2.095e-6 per second against 2.098e-6 for the 3.8235-day half-life.

## Versions

The version numbers are static (`CMakeLists.txt`: GCClassic 14.8.0; HEMCO 3.13.0 in its `CMakeLists.txt` and `HCO_VERSION`; GEOS-Chem 14.8.0), so a git tag does not change them. CMake's "Current status" line (`git describe --tags`) shows the tag, and `createRunDir.sh` records the GEOS-Chem commit in each run directory's `rundir.version`.
