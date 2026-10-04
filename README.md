# Published VASP data for MLIP training

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Python 3.10+](https://img.shields.io/badge/python-3.10%2B-blue.svg)](pyproject.toml)

Code and analysis for collecting the VASP calculations that research groups publish in open data
repositories (Zenodo, NOMAD and the Materials Cloud Archive) and turning them into one consistent,
documented training dataset for machine-learning interatomic potentials (MLIPs).

This repository is the outcome of a summer research project (July–October 2026) supervised by
Dr Seán Kavanagh.

## Why this data

MLIPs learn how atoms interact from DFT calculations. The widely used training sets (MPtrj,
OMat24, sAlex) come from a few large high-throughput databases. They contain almost only bulk
crystals, each computed with one set of settings. At the same time, hundreds of research groups
publish the raw output of their own VASP calculations alongside their papers: surfaces and
interfaces, molecules, defects, and molecular dynamics at finite temperature. This data is
scattered across repositories, packed inside archives and rarely reused. This project finds it,
extracts the energy, forces and stress of every ionic step, and keeps the full settings and the
source of every calculation.

## Results at a glance

All three harvests are complete, and each passes an exact integrity check: every stored frame
matches its metadata record. Numbers as of October 2026.

| Source | Records | Calculations | Frames (ionic steps) |
|---|---:|---:|---:|
| Zenodo | 629 | 386,425 | 18,243,690 |
| Materials Cloud Archive | 102 | 75,751 | 2,545,669 |
| NOMAD, individual uploads | 1,976 uploads | 869,960 | 10,087,877 |
| **Published by individual research groups** | **2,707** | **1,332,136** | **30,877,236** |
| NOMAD, Alexandria database runs ¹ | 1,719 uploads | 6,203,632 | 42,371,188 |

¹ Most NOMAD calculations turned out to be the Alexandria database's own high-throughput runs,
uploaded as ordinary NOMAD uploads. Alexandria already feeds sAlex and OMat24, so these runs are
kept separate and left out of the analysis below.

The data published by individual groups was compared, with the same code, against MPtrj, OMat24,
sAlex, the Materials Project and Alexandria ([full evaluation](docs/DATASET_EVALUATION.md)):

- **Size.** 30.9M frames with 3.41 billion per-atom forces, from more than 700 first authors.
  After sAlex's subsampling rule (keep a frame only if its energy changed by more than
  10 meV/atom), 2.67M frames remain: 3.8 times MPtrj under the same rule.
- **Different structures.** 35% of frames are bulk crystals, 52% slabs or 2D systems and 12%
  molecules or clusters. Every reference set is at least 97.7% bulk.
- **Different calculations.** 38% of frames come from ab initio molecular dynamics and 54% from
  relaxations. Half of the frames use something other than plain PBE: RPBE, PBEsol, dispersion
  corrections, hybrids, r2SCAN or +U.
- **New chemistry.** 26,771 of the 55,189 chemical systems, holding 12.9% of the frames, occur in
  neither the Materials Project nor Alexandria.
- **Quality.** 96.2% of frames pass the default quality filters. 5.49M frames use exactly the
  Materials Project settings, so they can be combined with MPtrj directly.
- **Finding hidden data.** Zenodo's search sees only the text of a record, not the files inside
  its archives. A census of all 583,930 Zenodo records that hold archives found 317 VASP records
  that keyword search had missed, which more than doubled the Zenodo calculations
  ([details](docs/ZENODO_CENSUS.md)).

## How it works

The pipeline runs in five resumable stages, which pass work to each other through JSONL
manifests:

| Stage | What it does |
|---|---|
| discover | Find candidate records: keyword search or a full census on Zenodo, an indexed query on NOMAD, a complete listing of the Materials Cloud Archive. |
| triage | Read each record's file list and, through HTTP range requests, the directory of each remote zip file, to confirm VASP outputs without downloading them. |
| fetch | Download only the VASP files: pull single files out of remote zips, extract VASP files from other archives (nested ones included), verify checksums and resume broken transfers. |
| parse | Read each calculation with pymatgen (`vasprun.xml`, `vaspout.h5`) or ASE (`OUTCAR`): one frame per ionic step with energy, forces, stress, SCF convergence, net magnetic moment and net charge. |
| store | Write compressed extxyz files and one metadata record per calculation, then check that the two match exactly. |

A few design choices made the full harvests possible:

- **It fits a fixed quota.** CSD3's scratch space allows 1 TB and 1 million files. The fetch
  stage counts every byte and every file as it is written and pauses when a budget is reached.
  The parse stage then turns the staged files into frames and frees the space. Several terabytes
  of archives passed through this way.
- **It survives job time limits.** Jobs end after 12–36 h, so every stage resumes where it
  stopped, and long runs resubmit themselves.
- **Nothing is dropped silently.** Every rejected record, file and calculation is logged with a
  reason.
- **All sources share one format.** The NOMAD and Materials Cloud adapters reuse the fetch, parse
  and store stages of the Zenodo pipeline.

## Data format

Each source becomes a directory of `shard-NNNNN.extxyz.gz` files (about 10,000 frames each) and
one `metadata.jsonl`, joined by `calc_id` and `frame_id`.

Per frame, in ASE's `atoms.info` and `atoms.arrays`:

| Key | Content |
|---|---|
| `REF_energy` | total energy extrapolated to σ → 0 (eV) |
| `REF_forces` | force on every atom (eV/Å) |
| `REF_stress` | stress in ASE's Voigt order and sign (eV/Å³), where VASP computed it |
| `E_free` | free energy F, the energy consistent with the forces (eV) |
| `electronic_converged`, `scf_dE` | whether this step's SCF loop converged, and its last energy change |
| `total_magnetization`, `total_charge` | net magnetic moment (μB) and net charge (e) of the cell |
| `calc_id`, `frame_id`, `ionic_step` | links to the calculation's metadata record |

The `REF_*` names are MACE's default keys, so the files can be used for training as they are.

Per calculation, `metadata.jsonl` records the source (repository, record ID, DOI, licence,
citation, file path), the full settings (INCAR as written and as resolved by VASP, k-points,
POTCARs and a hash of the POTCAR set, functional), quality flags (SCF and ionic convergence, frame
counts), and which heavy outputs exist at the source but are not stored (charge density, DOS,
eigenvalues, wavefunctions).

```python
from ase.io import read

frames = read("shard-00000.extxyz.gz", index=":")
energy = frames[0].info["REF_energy"]    # eV
forces = frames[0].arrays["REF_forces"]  # eV/Å, shape (n_atoms, 3)
```

## Installation

Python 3.10 or newer.

```bash
git clone https://github.com/yuxie432/open-vasp-harvest.git
cd open-vasp-harvest
python -m venv .venv && source .venv/bin/activate
pip install -e ".[parse,archives]"   # pymatgen, ASE, h5py; .7z, .rar and .zst archives
pip install -e ".[dev]"              # optional: pytest, mypy, ruff
```

- `.rar` files also need an `unrar` or `bsdtar` binary on `PATH`.
- For Zenodo, put a personal access token in an untracked `.env` file as `ZENODO_TOKEN=…`. It
  raises the search page size from 25 to 100. NOMAD and the Materials Cloud Archive need no token.
- The statistics of the reference sets (`python -m dataset_stats.cli ref-fetch`) also need
  `pip install ase-db-backends`.
- Every tool runs from the repository root as `python -m <package>.cli`, and each command
  explains its options with `--help`.

## Quick start

A small trial on Zenodo:

```bash
python -m zenodo_harvest.cli discover --query VASP --query OUTCAR --max-records 200 \
    --out data/manifests/candidates.jsonl
python -m zenodo_harvest.cli triage --in data/manifests/candidates.jsonl \
    --out data/manifests/keep.jsonl --min-rank 3
python -m zenodo_harvest.cli fetch --in data/manifests/keep.jsonl --max-bytes 500000000 --workers 4
python -m zenodo_harvest.cli parse --in data/manifests/fetched.jsonl
python -m zenodo_harvest.cli verify --dataset-dir data/dataset
```

Everything is written under `data/` (gitignored), or under `$ZENODO_HARVEST_DATA` if it is set.

## Full harvests on CSD3

The full harvests ran as SLURM jobs on CSD3. Each part has its own job scripts and runbook:

| Directory | Contents |
|---|---|
| [`scripts/csd3/`](scripts/csd3/README.md) | Zenodo harvest and recovery jobs, and the cluster limits they are built around |
| [`scripts/csd3/census/`](scripts/csd3/census/README.md) | Zenodo census: census, scoring, triage and recovery |
| [`scripts/csd3/nomad/`](scripts/csd3/nomad/README.md) | NOMAD harvest |
| [`scripts/csd3/materials_cloud/`](scripts/csd3/materials_cloud/README.md) | Materials Cloud harvest |
| [`scripts/csd3/stats/`](scripts/csd3/stats/README.md) | dataset statistics and comparison with the reference sets |
| [`scripts/csd3/test/`](scripts/csd3/test/README.md) | smoke tests of every stage and every safety limit |

On the cluster, fetch, parse and store run as one overlapped command: batch *i + 1* downloads
while batch *i* is parsed and its staged files are removed.

```bash
python -m zenodo_harvest.cli pipeline --in data/manifests/keep.jsonl \
    --parts 40 --workers 4 --max-bytes 0 --max-member-bytes 30000000000 \
    --max-disk-bytes 800000000000 --max-disk-files 800000 --max-primary-bytes 2000000000
```

| Option | Meaning |
|---|---|
| `--max-bytes` | skip any single download larger than this; `0` means no limit (the production setting) |
| `--max-member-bytes` | largest file extracted from an archive, a guard against decompression bombs |
| `--max-disk-bytes`, `--max-disk-files` | staging budget in bytes and in files plus directories, charged as data is written |
| `--max-primary-bytes` | largest `vasprun.xml` or `OUTCAR` to parse; pymatgen needs about 10 times the file size in RAM |
| `--workers`, `--parse-workers` | number of parallel downloads and parallel parses |
| `--no-zip-stream` | download whole zip files instead of pulling single files out of them |

How the staging budget is enforced (details in [`docs/DESIGN.md`](docs/DESIGN.md) §6):

- **Charge before writing.** Every chunk of about 1 MB, and every new file or directory, is charged
  before it is written and refunded when it is deleted. Nothing is predicted from an archive's
  declared size or compression ratio, so staging stays within the budget whatever an archive
  expands to.
- **Pause and resume.** A record that would cross the budget is rolled back whole and the fetch
  stops. The pipeline parses and removes what is staged, then fetches the same batch again. A
  record too large for the whole budget is reported instead of being retried forever.
- **Survive a full disk.** If the filesystem itself fills up (`ENOSPC`, `EDQUOT`), the affected
  records count as a temporary failure and the next run retries them. They are never recorded as
  holding no VASP data.

Filling the budget to about 98% is normal, because safety comes from the check before each write,
not from spare space. Each run reports its peak usage (`peak_staged_bytes`, `peak_staged_files`).
In 570 randomised end-to-end tests (different compression ratios, wrong declared sizes, 1–4
workers, both limits) the staging area never went over its budget.

## Repository layout

```
zenodo_harvest/           core pipeline: discover, triage, fetch, parse, store, verify (Zenodo)
nomad_harvest/            NOMAD adapter (discover and fetch; reuses parse and store)
materials_cloud_harvest/  Materials Cloud Archive adapter
zenodo_census/            census of every archive-bearing Zenodo record, to find the VASP data
                          keyword search misses
dataset_stats/            statistics of the datasets and of the reference training sets
scripts/csd3/             SLURM job scripts and runbooks for CSD3
scripts/estimate/         measurement tools from the initial survey
tests/                    offline test suite
docs/                     design notes, results and evaluation
CLAUDE.md                 detailed code map and conventions, kept as guidance for AI coding
                          assistants (each package has its own)
```

## Documentation

| Document | Contents |
|---|---|
| [DESIGN.md](docs/DESIGN.md) | data model, storage format and pipeline design |
| [survey-findings.md](docs/survey-findings.md) | initial survey: how much VASP data Zenodo holds |
| [HARVEST_RESULT.md](docs/HARVEST_RESULT.md), [EVALUATION.md](docs/EVALUATION.md) | Zenodo harvest: result, quality checks and limitations |
| [ZENODO_CENSUS.md](docs/ZENODO_CENSUS.md) | finding the Zenodo VASP data that keyword search misses |
| [NOMAD_HARVEST.md](docs/NOMAD_HARVEST.md), [NOMAD_HARVEST_RESULT.md](docs/NOMAD_HARVEST_RESULT.md) | NOMAD: design and result |
| [MATERIALS_CLOUD_HARVEST.md](docs/MATERIALS_CLOUD_HARVEST.md), [MATERIALS_CLOUD_HARVEST_RESULT.md](docs/MATERIALS_CLOUD_HARVEST_RESULT.md) | Materials Cloud Archive: design and result |
| [EXTERNAL_DATA_SOURCES.md](docs/EXTERNAL_DATA_SOURCES.md) | other databases considered, and why they were not harvested |
| [DATASET_EVALUATION.md](docs/DATASET_EVALUATION.md) | statistics, and comparison with MPtrj, OMat24, sAlex, MP and Alexandria |
| [FURTHER_WORK.md](docs/FURTHER_WORK.md) | next steps |

The result documents are dated working records. Where their numbers differ, the most recent
document, [DATASET_EVALUATION.md](docs/DATASET_EVALUATION.md), is authoritative.

## Tests

```bash
python -m pytest tests/ -q       # 702 offline tests, about 30 s, no network needed
python -m mypy zenodo_harvest/   # type check
ruff check zenodo_harvest/       # lint
```

## Status

- **Done:** the Zenodo harvest (keyword search plus the census), the NOMAD and Materials Cloud
  harvests, and the comparison with the reference training sets.
- **Not done:** merging the three datasets into one cleaned dataset (removing duplicates,
  improving the energy filters, grouping by functional and POTCAR set), and measuring the value of
  the data for MLIP training, for example by fine-tuning MACE with and without it. Both are
  planned in [FURTHER_WORK.md](docs/FURTHER_WORK.md) and in §7 of
  [DATASET_EVALUATION.md](docs/DATASET_EVALUATION.md).
- The datasets themselves (about 140 GiB of compressed extxyz files) are stored on CSD3. They are
  not part of this repository and have not been released publicly.

## Licence

The code and documentation in this repository are released under the [MIT Licence](LICENSE).

The harvested data are not covered by this licence. Every calculation keeps the licence of the
record it came from, stored in its metadata. By frames, Zenodo is 97.8% CC-BY-4.0, NOMAD is 100%
CC-BY-4.0, and the Materials Cloud Archive is 65.6% CC-BY-SA-4.0, 26.6% CC-BY-4.0 and 7.8% MIT. A
few records are CC-BY-NC, and one Zenodo record without a stated licence (`10579527`) was added at
the supervisor's direction. Anyone who redistributes the data must credit the original depositors
and respect share-alike and non-commercial terms where they apply.

## Acknowledgements

Supervised by Dr Seán Kavanagh. The data were published by the researchers who deposited them on
Zenodo, NOMAD and the Materials Cloud Archive; every calculation keeps a link to its source.
Computations used the Cambridge Service for Data Driven Discovery (CSD3), operated by the
University of Cambridge Research Computing Service.
