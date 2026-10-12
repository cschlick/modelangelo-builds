# modelangelo-builds

Paired dataset of **71 raw ModelAngelo builds** and the **deposited PDB structures** for
the same cryo-EM maps. Built as the machine-built input class for a study of
map-versus-geometry weighting in real-space refinement; 64 of the 71 form that study's
pooled build corpus (`usable_in_study` in the manifest).

## Contents

```
manifest.csv                      one row per structure (see columns below)
builds/<pdb>_modelangelo_raw.cif.gz   raw ModelAngelo output, unedited, un-pruned
deposited/<pdb>.cif.gz                the deposited PDB entry (wwPDB mmCIF)
```

`manifest.csv` columns:

| column | meaning |
|---|---|
| `pdb_id` | PDB entry of the deposited structure |
| `emdb_id` | EMDB entry of the map ModelAngelo was run on |
| `resolution_A` | reported resolution (Å) |
| `batch` | generation batch (1–3; three successive cohorts) |
| `usable_in_study` | `yes` for the 64 builds in the pooled study corpus (`no` = failed a box-size/usability screen downstream, build still provided) |
| `build_file`, `deposited_file` | relative paths in this repo |
| `map_url`, `fasta_url` | exact inputs fetched for the build |
| `build_date` | date the ModelAngelo build ran (maps and FASTAs were fetched the same day) |
| `deposited_retrieved` | date the deposited mmCIF was downloaded from wwPDB (rsync mirror sync) |
| `deposited_wwpdb_file_date` | the wwPDB file's own modification date at retrieval (preserved by rsync) — identifies which revision of the entry this is |

## How the builds were made

Each build is the unedited output of

```
model_angelo build -v map.map -pf seq.fasta -o modelangelo_out
```

(default settings) where `map.map` is the **deposited primary EMDB map** (`map_url`) and
`seq.fasta` the protein sequence from RCSB (`fasta_url`).

**Versions and dates.** ModelAngelo was installed via an unpinned
`git clone https://github.com/3dem/model-angelo && ./install_script.sh`; the tip of
`main` at build time was commit `994945bd` (2026-06-15), identical to the
**v1.0.18** release, so the builds are v1.0.18. The three batches ran on
2026-09-07, 2026-09-08, and 2026-09-10 (`build_date`); maps and FASTAs were fetched
fresh the same day. Deposited mmCIFs come from a wwPDB rsync mirror synced
2026-07-18/19 (`deposited_retrieved`), with each file's upstream modification date
preserved in `deposited_wwpdb_file_date`.

The file shipped here is
`modelangelo_out_raw.cif` — the raw model **before** ModelAngelo's own confidence
pruning, so low-confidence regions are retained; per-residue confidence is stored in the
B-factor field (confidence × 100), as ModelAngelo writes it.

Entries were drawn from EMDB/PDB depositions with both half maps available,
2,000–13,000 atoms, passing a model-on-density registration screen (CC ≥ 0.30), with a
relatedness cap of ≤ 2 entries per 3-character PDB-id prefix.

## Why this pairing is useful

A raw machine build is, measured against its own deposited structure, *a model in the
right place wearing the wrong chemistry*: median matched-Cα distance to the deposit
~0.22 Å (97% of Cα match within 3 Å), while bond RMSD is ~0.066 Å and MolProbity ~3.8
against ~1.4 for the deposit. That combination (near-deposit coordinates, unregularised
geometry) is exactly the regime where refinement defaults mis-behave — e.g.
`phenix.real_space_refine`'s weight scan reads the pre-existing strain as map pull and
collapses its weight toward zero — making these pairs a ready-made benchmark for
refinement procedures on builder output: refine the build, score against the map, the
deposit, and geometry.

## Licensing and attribution

Deposited structures are wwPDB data (CC0). Maps and sequences are referenced by URL, not
redistributed. Builds were generated with
[ModelAngelo](https://doi.org/10.1038/s41586-024-07215-4) (Jamali et al., *Nature* 628,
2024) — cite it if you use the builds. This dataset itself is released under CC0.

A companion dataset of planted-error structures from the same study:
[planted-error-structures](https://github.com/cschlick/planted-error-structures).
