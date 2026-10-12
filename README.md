# modelangelo-builds

Paired dataset of **64 raw ModelAngelo builds** and the **deposited PDB structures** for the
same cryo-EM maps. Each build is unedited, un-pruned ModelAngelo output; each pairing gives a
machine-built model and a human-curated model of the same density, so the two can be compared
directly or used as a before/after pair for refinement and validation work.

The dataset was generated as the machine-built input class for a study of map-versus-geometry
weighting in real-space refinement, but it is self-contained: nothing here depends on that
study's conventions, and the manifest records the exact inputs so any build can be reproduced
or re-derived independently.

## Contents

```
manifest.csv                           one row per structure (columns below)
builds/<pdb>_modelangelo_raw.cif.gz    raw ModelAngelo output, unedited, un-pruned
deposited/<pdb>.cif.gz                 the deposited PDB entry (wwPDB mmCIF)
```

`manifest.csv` columns:

| column | meaning |
|---|---|
| `pdb_id` | PDB entry of the deposited structure |
| `emdb_id` | EMDB entry of the map ModelAngelo was run on |
| `resolution_A` | reported resolution (Å) |
| `batch` | generation batch (1–3; three successive cohorts, see *Selection*) |
| `build_file`, `deposited_file` | relative paths in this repo |
| `map_url`, `fasta_url` | the exact inputs fetched for the build |
| `build_date` | date the ModelAngelo build started; the map and FASTA were fetched minutes earlier, the same day |
| `deposited_retrieved` | date the deposited mmCIF was downloaded from a wwPDB rsync mirror |
| `deposited_wwpdb_file_date` | the wwPDB file's own modification date at retrieval (preserved by rsync) — identifies which revision of the entry this is |

# Methods

## Software and versions

Builds were produced with **ModelAngelo v1.0.18**, upstream commit
`994945bdfa6e5368e0d62349a47792f4864eebc3` (2026-06-15,
[3dem/model-angelo](https://github.com/3dem/model-angelo)). ModelAngelo was installed from an
unpinned `git clone` followed by its own `install_script.sh`; the tip of `main` at install
time was that commit, and `git describe --exact-match --tags` places the `v1.0.18` release tag
exactly on it, so the clone and the tagged release are the same code.

| component | version |
|---|---|
| ModelAngelo | 1.0.18 (commit `994945bd`) |
| Python | 3.11.16 |
| PyTorch | 2.9.1+cu128 (CUDA 12.8, cuDNN 9.10.2) |
| NVIDIA driver | 550.163.01 |
| GPU | NVIDIA GeForce RTX 3050, 8 GB |
| weights bundle | `nucleotides` (`model_angelo_v1.0`) |
| language model | `esm1b_t33_650M_UR50S` (fair-esm 1.0.3) |
| other | pyhmmer 0.7.1, numpy 1.26.4, scipy 1.17.1, biopython 1.88, mrcfile 1.5.4, einops 0.8.2 |

PyTorch ships its own CUDA 12.8 runtime in the `+cu128` wheel and ran against a driver
advertising CUDA 12.4 via minor-version compatibility.

## Inputs

For each entry, two files were fetched immediately before the build and used without any
preprocessing:

- the **deposited primary EMDB map** (`map_url`), gunzipped — no masking, sharpening,
  resampling, or box adjustment;
- the **protein sequence FASTA from RCSB** (`fasta_url`), passed whole.

Both URLs are recorded per row, so the inputs are addressable rather than redistributed. No
half maps, nucleotide sequences, or user-supplied masks were used.

## Build command

Every build is the output of exactly:

```
model_angelo build -v map.map -pf seq.fasta -o modelangelo_out
```

All settings are ModelAngelo defaults. No flags were passed beyond those three, so box size
(64), stride (16), C-alpha threshold (0.05), three GNN refinement rounds, crop length (200),
batch size, and `fp16` (off) are all at their shipped values. The compute device was
auto-selected, resolving to `cuda:0` on every run. The pipeline therefore ran its full
default course for each entry: initial C-alpha prediction, three rounds of GNN refinement with
ESM-1b sequence attention, then HMM-based sequence assignment and confidence pruning.

## Hardware and runtime

All 64 builds ran on a single RTX 3050 (8 GB). Peak GPU memory was ~5.9 GB and effectively
independent of map size, so default settings fit the card with headroom and no batch or
box-size reduction was needed. Per-structure wall-clock was **7 to 88 minutes** (median 23),
totalling ~32 GPU-hours. Builds ran in three batches on 2026-09-07 through 2026-09-10; the
per-entry date is in `build_date`.

## What is shipped

The file provided per entry is `modelangelo_out_raw.cif` — the **raw model before
ModelAngelo's own confidence pruning**, so low-confidence regions are retained rather than
discarded. Per-residue confidence is in the B-factor field as confidence × 100, which was
checked to lie in [0, 100] for all 64 builds. Files are gzipped but otherwise bit-for-bit as
ModelAngelo wrote them; no renaming, re-ordering, chain relabelling, or geometry regularisation
was applied.

Deposited structures are wwPDB mmCIF files taken from an rsync mirror, retrieved 2026-07-18
and 2026-07-19. Because PDB entries are revised over time, each file's own modification date
at retrieval is recorded in `deposited_wwpdb_file_date` (spanning 2023-08-25 to 2026-07-10),
which identifies the revision used.

## Selection

Entries are EMDB/PDB pairs where the deposition provides both a primary map and a
corresponding atomic model. They were drawn in three successive batches — 13, 28, and 23
structures — across three resolution bands:

| band | n | resolution (Å) |
|---|---|---|
| high-resolution | 10 | 1.67 – 1.76 |
| intermediate | 28 | 1.92 – 2.36 |
| classic | 26 | 2.55 – 3.84 |

Overall span is **1.67 – 3.84 Å** (median 1.98). Two further constraints were applied: a
relatedness cap of at most 2 entries per 3-character PDB-id prefix (verified: 4 prefixes
contribute 2 entries, 56 contribute 1), and a model-on-density screen requiring CCmask ≥ 0.30
between the deposited model and the map, which all 64 entries clear (observed range
0.34 – 0.79).

**This is not a random sample of the PDB**, and three things narrow what it generalises to.
The resolution bands were chosen deliberately rather than sampled. Candidate pools were
assembled from local mirrors, so availability on those mirrors shaped the pool. And seven
builds that were generated are **not** included here: three that failed a map-coverage screen
(the fraction of the deposited model falling inside the scored box measured 0.50, 0.18 and
0.24 against a 0.60 threshold — the deposition models only part of the map), two where the
build itself fell below the CCmask 0.30 on-density threshold (0.297 and 0.048), and two
dropped by the same screening without a per-entry reason being recorded. Those seven are absent from both the
manifest and the file tree, so the set is the subset that passed usability screening, not
everything the builder produced.

## Verification

The dataset was audited against the machine that produced it. All 64 builds are
**byte-identical** (md5 of the decompressed content) to the ModelAngelo output files as
originally written, so nothing was altered in packaging. Also checked: all 128 gzip files pass
`gzip -t`; manifest row count matches the file counts in both directories with no orphans or
duplicate `pdb_id`/`emdb_id`; every `build_file` and `deposited_file` path resolves; each
`emdb_id` is consistent with its `map_url` and each `fasta_url` with its `pdb_id`; every
deposited file's own `data_` block matches its `pdb_id`; resolutions agree with the source
generation manifests; and `build_date` was set from the timestamp inside each run's own
ModelAngelo log rather than from the batch it belonged to.

## Dataset characteristics

Measured directly from the files in this repository:

| quantity | value |
|---|---|
| atoms per build | 4,259 – 71,196 (1,336,722 total) |
| atoms per deposited model | 3,528 – 95,536 (1,702,926 total) |
| deposited Cα atoms compared | 161,614 |
| median distance, deposited Cα → nearest build Cα | **0.201 Å** pooled; per-structure median 0.222 Å (range 0.096 – 1.110) |
| deposited Cα within 3.0 Å of a build Cα | **98.7%** pooled; per-structure median 99.1% (range 93.9 – 100) |
| deposited Cα within 1.0 Å of a build Cα | 93.8% pooled |

The Cα figures are nearest-neighbour distances computed here with a KD-tree over all Cα
atoms, with no chain or sequence correspondence imposed; they are a positional-agreement
measure, not a sequence-aware alignment, and they are reported that way because that
definition is reproducible from these files alone.

Note that builds and deposited models do not cover identical regions. ModelAngelo builds into
whatever density supports a backbone, while a deposition may model more (ordered ligands,
alternate conformers) or less (one component of a larger map). Atom counts differ accordingly
in both directions and should not be read as a completeness score.

### Figures from the companion analysis

These were measured with Phenix/cctbx in the study this dataset was built for, and are
**not** recomputed from the files here, so treat them as approximate context rather than
dataset-verified values: bond RMSD ≈ 0.066 Å for the raw builds, and MolProbity score ≈ 3.8
for builds against ≈ 1.4 for the deposited models.

## Why this pairing is useful

Measured against its own deposited structure, a raw machine build is *a model in roughly the
right place wearing the wrong chemistry*: Cα positions agree with the deposit to ~0.2 Å in the
median while the geometry is unregularised. That combination — near-deposit coordinates with
strained bonds and angles — is exactly the regime where refinement defaults misbehave. In the
companion analysis, `phenix.real_space_refine`'s weight scan read the pre-existing strain as
map pull and collapsed its weight toward zero. These pairs are therefore a ready-made
benchmark for refinement and validation procedures on builder output: refine the build, then
score it against the map, against the deposit, and on geometry.

## Reproducing a build

```bash
# pick a row from manifest.csv, then:
curl -sL "<map_url>"   -o map.map.gz && gunzip map.map.gz
curl -sL "<fasta_url>" -o seq.fasta
model_angelo build -v map.map -pf seq.fasta -o modelangelo_out
# compare modelangelo_out/modelangelo_out_raw.cif with builds/<pdb>_modelangelo_raw.cif.gz
```

Exact reproduction requires ModelAngelo v1.0.18 and the same weights bundle; results may also
shift slightly with GPU model, CUDA version, and cuDNN nondeterminism. Note also that EMDB and
RCSB serve current files — if an entry has been revised since the `build_date` in the
manifest, the inputs you fetch may differ from the ones used here.

## Licensing and attribution

Deposited structures are wwPDB data (CC0). Maps and sequences are referenced by URL, not
redistributed. Builds were generated with
[ModelAngelo](https://doi.org/10.1038/s41586-024-07215-4) (Jamali et al., *Nature* 628, 2024)
— cite it if you use the builds. This dataset itself is released under CC0.

A companion dataset of planted-error structures from the same study:
[planted-error-structures](https://github.com/cschlick/planted-error-structures).
