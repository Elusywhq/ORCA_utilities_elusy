# ORCA Personal Workflow Scripts

This repository is a personal toolbox for running and post-processing ORCA calculations, especially workflows related to:

- ground-state and excited-state optimization/frequency jobs
- fluorescence, ISC, RISC, phosphorescence, and SOC calculations
- PBS submission on multiple hosts
- extraction of rates, energies, spectra, orbital information, and excited-state descriptors

It is not a packaged software project. Most scripts are lightweight shell helpers written for day-to-day use, and many of them assume a specific directory layout, host name, scratch path, ORCA installation, and local helper commands.

## What This Repository Does

At a high level, the repository already implements a usable workflow:

1. generate ORCA input files from a small set of labels
2. generate a PBS submission script based on the current host
3. submit jobs and check queue/output status
4. extract results into text or CSV summaries

The core idea is centered around a consistent file naming scheme and a small set of job categories.

## Main Job Categories

The current scripts are easiest to understand if grouped by the job being run.

| Job category | ORCA-Input job index | Typical output label | Main scripts |
| --- | --- | --- | --- |
| Optimization + frequency | `1` | `optfreq` | `ORCA-Input`, `ORCA-submission`, `auto_submit`, `ORCA-chkjob` |
| Optimization only | `2` | `opt` | `ORCA-Input`, `ORCA-submission` |
| Frequency only | `3` | `freq` | `ORCA-Input`, `ORCA-submission` |
| Fluorescence / ESD | `4` | `fluo` | `ORCA-Input`, `ORCA-submission -esd`, `ORCA-Grepk`, `ORCA-GrepSpectrum` |
| ISC / ESD | `5` | `isc` | `ORCA-Input`, `ISC.sh`, `isc_submit`, `ORCA-GrepVC`, `grep_data_isc` |
| RISC / ESD | `6` | `risc` | `ORCA-Input`, `ISC.sh`, `ORCA-GrepVC`, `grep_data_risc` |
| Phosphorescence / ESD | `7` | `phosph` | `ORCA-Input`, `phosp.sh`, `ORCA-submission -esd` |
| Spin-orbit coupling | `8` | `soc` | `ORCA-Input`, `ORCA-GrepSOCME` |

## Common Naming Convention

Most generated files follow this pattern:

```text
<molecule>_<functional>_<basis>_<state>_<method>_<job>.inp
```

Example:

```text
1TD4_M062X_svp_s1_Dscf_optfreq.inp
```

This naming convention is the backbone of the repository. Many scripts infer related files by rebuilding names such as:

- `s0_scf_optfreq`
- `s1_Dscf_optfreq`
- `t1_scf_optfreq`
- `s1_esd_fluo`
- `t1_esd_isc_T1`
- `t1_esd_risc_T2`

## Assumptions and Environment

These scripts assume some or all of the following:

- ORCA is installed and `ORCA_ROOT` is set
- PBS commands such as `qsub` are available
- `qjlist` exists in your shell environment
- `calculate_rmsd` exists in your shell environment
- helper programs such as `bc`, `awk`, `sed`, `grep`, and `orca_2mkl` are available
- optional external tools such as `Multiwfn` and `TheoDORE` are installed for analysis scripts

Several scripts also hard-code behavior based on host names such as:

- `atlas` -> `hpc`
- `asp2a` -> `nscc`
- `Precision` -> local workstation
- `stdct` -> `vanda`

## Core Workflow

### 1. Generate an input file

`ORCA-Input` is the main input generator. It can be used interactively or by passing positional arguments.

Current positional format:

```bash
ORCA-Input <molecule> <xyz> <state_index> <job_index> <functional_index> [method_index] [es_state] [ts_state] [custom_name]
```

Important index mappings:

- states: `1=s0`, `2=s1`, `3=t1`
- jobs: `1=Optfreq`, `2=Optimisation`, `3=Frequency`, `4=Fluorescence`, `5=ISC`, `6=RISC`, `7=Phosphorescence`, `8=SOC`
- functionals: `1=O3LYP`, `2=pbe0`, `3=camB3LYP`, `4=M062X`, `5=wB97XD4`

Examples:

```bash
ORCA-Input 1TD4 1TD4.xyz 1 1 4
ORCA-Input 1TD4 1TD4.xyz 2 1 4 2
ORCA-Input 1TD4 1TD4_M062X_svp_s0_scf_optfreq.xyz 3 5 4
```

Notes:

- input blocks come from `block_input`
- basis is currently hard-coded to `def2-SVP`
- fluorescence, ISC, RISC, and phosphorescence jobs may trigger extra helper scripts automatically

### 2. Generate a PBS submission script

`ORCA-submission` creates a host-specific `.sh` file from one of the templates in this repository.

Typical usage:

```bash
ORCA-submission -n 1TD4_M062X_svp_s0_scf_optfreq
ORCA-submission -n 1TD4_M062X_svp_s1_esd_fluo -esd -m 240 -t 72
ORCA-submission -n 1TD4_M062X_svp_s1_Dscf_optfreq -d
```

Useful options:

- `-n <name>`: input name without `.inp`
- `-c <cores>`: number of cores
- `-m <memory>`: memory
- `-t <hours>`: walltime
- `-w <gbw>`: restart from another wavefunction
- `-esd`: use ESD template
- `-ci`: use CI-optimization template
- `-d`: use Dscf template

### 3. Submit

Once the `.sh` file exists:

```bash
qsub <jobname>.sh
```

Some scripts automate this step:

- `auto_submit`
- `ORCA-TADF-auto`
- `isc_submit`

### 4. Check job status

Typical commands:

```bash
ORCA-chkjob <jobname>.inp
ORCA-chkout <jobname>.inp
ORCA-chkCI <jobname>.out
```

### 5. Extract data

After jobs finish, the `ORCA-Grep*` and `grep_data_*` scripts summarize outputs into shell output or CSV files.

## Script Catalogue

### Input and submission

| Script | What it is for | General usage |
| --- | --- | --- |
| `ORCA-Input` | Main ORCA input generator. Builds the standard file name and injects job-specific blocks. | `ORCA-Input <molecule> <xyz> <state> <job> <functional> [method]` |
| `block_input` | Library of reusable ORCA input blocks used by `ORCA-Input`. | Not run directly; sourced by `ORCA-Input`. |
| `ORCA-submission` | PBS submission-script generator with host-specific templates. | `ORCA-submission -n <jobname> [-esd|-ci|-d] [-c cores] [-m memory] [-t hours]` |
| `template_orca_default_hpc` | Default PBS template for host `hpc`. | Used by `ORCA-submission`. |
| `template_orca_default_nscc` | Default PBS template for host `nscc`. | Used by `ORCA-submission`. |
| `template_orca_default_vanda` | Default PBS template for host `vanda`. | Used by `ORCA-submission`. |
| `template_orca_esd_hpc` | PBS template for ESD jobs on `hpc`. | Used with `ORCA-submission -esd`. |
| `template_orca_esd_nscc` | PBS template for ESD jobs on `nscc`. | Used with `ORCA-submission -esd`. |
| `template_orca_ciopt_hpc` | PBS template for CI optimization jobs. | Used with `ORCA-submission -ci`. |
| `template_orca_Dscf_hpc` | PBS template for Dscf jobs. | Used with `ORCA-submission -d`. |
| `auto_submit` | Bulk generator/submission helper for molecule directories. Supports generating opt jobs, submitting opt jobs, and submitting some ESD jobs. | `auto_submit -g`, `auto_submit -opt`, `auto_submit -esd` |
| `ORCA-TADF-auto` | Older end-to-end automation for generating and submitting a TADF-style job set. | `ORCA-TADF-auto <molecule_dir> [more_dirs]` |
| `isc_submit` | Batch submit helper for existing ISC input files. | `isc_submit <pattern>` or `isc_submit -t <pattern>` |
| `fluo.sh` | Older hard-coded fluorescence job builder. | Edit variables inside the script before use. |
| `ISC.sh` | Helper that expands one ISC/RISC template into `T1/T2/T3` variants and rewrites hessian/xyz references. | Called automatically by `ORCA-Input` for ISC/RISC jobs. |
| `phosp.sh` | Helper that expands one phosphorescence template into `T1/T2/T3` variants. | Called by `ORCA-Input` for phosphorescence jobs. |
| `ORCA-ESD` | Checks optimization jobs inside molecule directories before ESD work. | `ORCA-ESD <molecule_dir> [more_dirs]` |

### Job checking and queue follow-up

| Script | What it is for | General usage |
| --- | --- | --- |
| `ORCA-chkjob` | Check whether an ORCA job terminated normally, locally or in scratch. | `ORCA-chkjob <jobname>.inp`, `ORCA-chkjob -q <jobname>.inp`, `ORCA-chkjob -loc <jobname>.out` |
| `ORCA-chkout` | Print the last optimization cycle and tail of an ORCA output file from scratch. | `ORCA-chkout <jobname>.inp` |
| `ORCA-chkCI` | Summarize CI optimization cycles, IROOT changes, multiplicities, and energies. | `ORCA-chkCI <jobname>.out` or `ORCA-chkCI -loc <jobname>.out` |

### Small extraction helpers

| Script | What it is for | General usage |
| --- | --- | --- |
| `ORCA-GrepSPE` | Extract final single-point energy from `.out`. | `ORCA-GrepSPE <file.out>` or `ORCA-GrepSPE -q <file.out>` |
| `ORCA-Grepk` | Extract rate constant from ESD output. | `ORCA-Grepk <file.out>` or `ORCA-Grepk -q <file.out>` |
| `ORCA-GrepSpectrum` | Return the strongest peak position from a `.spectrum` file. | `ORCA-GrepSpectrum <file.spectrum>` |
| `ORCA-GrepSOCME` | Extract SOC matrix elements and total SOCME magnitude. | `ORCA-GrepSOCME [-q] [-t triplet] [-s singlet] <file.out>` |
| `ORCA-GrepExcitation` | Infer dominant orbital excitation window for a selected state. | `ORCA-GrepExcitation [-q] [-i|-f] [-s state] <file.out>` |
| `ORCA-GrepHOMO` | Determine HOMO index from `.gbw`/`.molden.input`. | `ORCA-GrepHOMO <jobname>` or `ORCA-GrepHOMO -q <jobname>` |
| `ORCA-chkOrbital` | Estimate the orbital pair used in a Dscf excitation. | `ORCA-chkOrbital <file.out>` or `ORCA-chkOrbital -q <file.out>` |
| `ORCA-GrepAdDiff` | Compute adiabatic energy difference between two outputs. | `ORCA-GrepAdDiff file1.out file2.out` |
| `ORCA-GrepVC` | Extract vibrational-coupling data blocks into `.data` files. | `ORCA-GrepVC <esd_file.out>` |

### Data aggregation and rate analysis

| Script | What it is for | General usage |
| --- | --- | --- |
| `grep_data_rate` | Main batch CSV exporter for energies, rates, SOC, emission, and reference mapping across molecule folders. | Run from repo root: `grep_data_rate` or `grep_data_rate <prefix>` |
| `grep_data_isc` | Batch CSV exporter focused on ISC mode-by-mode analysis, HR factors, lambda, and SOC references. | Run from repo root: `grep_data_isc` |
| `grep_data_risc` | Batch analysis for RISC-oriented descriptors and Marcus-type estimates. | Run from repo root: `grep_data_risc` |
| `MarcusCTRate` | Executable front-end for Marcus charge-transfer rate calculation. | `MarcusCTRate -from <E_from> -to <E_to> -Hab <SOC_cm-1> -vlam <E_lambda>` |
| `MarcusCTRate.py` | Python source for `MarcusCTRate`. | `python MarcusCTRate.py ...` |

### Excited-state, TheoDORE, and Multiwfn analysis

| Script | What it is for | General usage |
| --- | --- | --- |
| `theo_ana` | Batch TheoDORE analysis for e-h distance, CT metrics, excitation character, and Dscf orbital mapping. | Run from repo root: `theo_ana` or `theo_ana <molecule>` |
| `test.py` | Python prototype/rewrite of the TheoDORE analysis workflow. | `python test.py [molecule]` |
| `multiwfn_ana` | Run a scripted Multiwfn hole-electron analysis and extract summary metrics. | `multiwfn_ana <jobname> [state]` |
| `dens_ana_template` | TheoDORE input template used by `theo_ana` and `test.py`. | Not run directly. |
| `atom_list` | Tab-separated fragment definitions used by TheoDORE analysis. | Input data file, not a program. |

### Geometry and vibrational utilities

| Script | What it is for | General usage |
| --- | --- | --- |
| `ORCA-displacement` | Generate a displaced geometry along vibrational mode 6 from a `.hess` file. | `ORCA-displacement <jobname>` |
| `ORCA-pltvib` | Thin wrapper around `orca_pltvib`. | `ORCA-pltvib <jobname> [mode]` |
| `dihedral.py` | Compute a dihedral angle from an `.xyz` file. | `python dihedral.py [-q] file.xyz i1 i2 i3 i4` |

### Repo and system maintenance

| Script | What it is for | General usage |
| --- | --- | --- |
| `pullgit.sh` | Pull this repo and a sister repo from the expected install location. | `pullgit.sh` |
| `auto_git_push.sh` | Convenience helper to add, commit, and push everything. | `auto_git_push.sh ["commit message"]` |
| `orca_install/install_orca.sh` | Placeholder install helper; currently only creates `$HOME/Software`. | Not useful yet as a full ORCA installer. |
| `check_directory.sh` | Empty placeholder. | No current function. |

## Recommended General Usage

For a manual single-job flow:

```bash
ORCA-Input 1TD4 1TD4.xyz 1 1 4
ORCA-submission -n 1TD4_M062X_svp_s0_scf_optfreq -m 240 -t 48
qsub 1TD4_M062X_svp_s0_scf_optfreq.sh
ORCA-chkjob 1TD4_M062X_svp_s0_scf_optfreq.inp
```

For a bulk directory-based flow:

```bash
auto_submit -g
auto_submit -opt
auto_submit -esd
grep_data_rate
```

## What Should Be Extracted During Cleanup

There is substantial duplicated logic across the repository. The main reusable pieces are:

- host detection and scratch-path selection
- ORCA file naming and name parsing
- job catalog definitions: states, methods, functionals, job labels
- template selection for PBS submission
- “finished or not finished” output checks
- queue checks and dependency submission
- common output parsers for energy, SOC, rates, and spectral data

These should ideally become shared functions or a single small library instead of being repeated in many shell scripts.

## Suggested Generalized Workflow

If this repository is cleaned up into a more general tool, the top-level interface should probably look like:

```text
orca-workflow generate --host <host> --xyz <file.xyz> --molecule <name> --state <s0|s1|t1> --job <optfreq|fluo|isc|risc|phosph|soc> --functional <O3LYP|pbe0|camB3LYP|M062X|wB97XD4> [--method ...]
orca-workflow submit --host <host> --input <jobname.inp> [--esd] [--cores N] [--memory GB] [--walltime H]
orca-workflow check --input <jobname.inp>
orca-workflow collect --mode <rate|isc|risc|theodore>
```

The repository already contains most of the logic needed for this. What is missing is mainly consolidation:

- one shared host configuration table
- one shared job specification table
- one shared name builder/parser
- one shared submission generator
- one shared output parser layer

## Caveats

- many scripts assume you run them either from the repo root or from inside a molecule folder
- several analysis scripts only work if molecule directories start with a digit
- some scripts are older prototypes and contain hard-coded assumptions
- not every script is portable across hosts without editing paths

## Summary

If you only keep three entry points in mind, they are:

- `ORCA-Input` for generating inputs
- `ORCA-submission` for generating PBS scripts
- `grep_data_*` / `ORCA-Grep*` for collecting results

Everything else in the repository is mostly a helper around those steps.
