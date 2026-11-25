# NanoRemix User Guide

This English edition uses NanoRemix v0.3.1 and the original published v0.3.0 weights. The workflow below starts with FAST5 or POD5. If you already have a prepared feature table or compatible probabilities, start at [Prepare the input table](#3-prepare-the-input-table).

1. [Choose a model and evidence route](#1-choose-a-model-and-evidence-route)
2. [Preprocess raw reads](#preprocess-raw-reads)
3. [Prepare the input table](#3-prepare-the-input-table)
4. [Install and check the weights](#4-install-and-check-the-weights)
5. [Run predictions](#run-predictions)
6. [Interpret the output](#interpret-the-output)
7. [CLI parameters and fusion calculations](#7-cli-parameters-and-fusion-calculations)

<a id="model-coverage"></a>

## 1. Choose a model and evidence route

Choose RNA002 or RNA004 according to the sequencing kit used for the sample. Binary compares the three dedicated pairs A/m6A, C/m5C and U/pseudoU. You can request several pairs in one run. Multi compares the selected labels within each candidate's reference-base family.

| Reference base | RNA002 Multi modifications | RNA004 Multi modifications |
| --- | --- | --- |
| A | m6A, m1A, Inosine, Am | m6A, m1A, Am |
| C | m5C, hm5C, ac4C, FormylC | m5C, hm5C, ac4C |
| G | m7G, o8G | o8G, Gm |
| U | pseudoU, m1psi, m5U, Um | pseudoU, m1psi, m5U, Um |

The four unmodified classes are A, C, G and U. `--mods all` selects all modifications supported by the current route with their matching unmodified controls. Multi also accepts exact label lists such as `--targets A,m6A,m1A`. Run `nanoremix catalog --chemistry RNA004` to check class names.

| Route | Select with | Inputs |
| --- | --- | --- |
| Signal only (third generation) | `--mode thirdgen` | Prepared RNA002/RNA004 features and matching weights, or compatible precomputed third-generation probabilities |
| Signal and sequence fusion | `--data-type biological` | The signal-branch input plus a same-center 81 nt sequence and the shared SecondGen weights, or complete second-generation probabilities |
| Sequence only (second generation) | `--mode secondgen` | Candidate reference-base identity plus an 81 nt sequence and the shared model, or complete second-generation probabilities |

“Second generation” and “third generation” identify NanoRemix model branches. The sequence branch can use reference or event sequence aligned to the same RNA candidate. Using `--data-type biological --also-thirdgen --also-secondgen` writes fusion and both component outputs. S2 is the default third-generation strategy. Use `--strategy both` to write S1 and S2 separately.

<a id="preprocess-raw-reads"></a>

## 2. Preprocess raw reads

Run the sequencing tools in their own environments. Keep the reference version, strand, candidate coordinates and read IDs available for joining the generated features. The versions below describe the documented upstream workflows.

### 2.1 RNA002 FAST5 to Step4

1. Basecall with Guppy 6.3.8 and the RNA002 HAC configuration `rna_r9.4.1_70bps_hac.cfg`. Use `--fast5_out` to retain basecalled FAST5.
2. Use `multi_to_single_fast5` from ont-fast5-api 4.1.1 to split the FAST5 files.
3. Resquiggle against a matching cDNA reference using Tombo 1.0 with GraphMap 0.5.2. Verify corrected Events and Alignment and inspect failed-read counts.
4. Use `extract_tombo_step4.py` with `--rna --clip 10` to export the 5-mer feature table and synchronous coordinate sidecar. The extractor's Python environment used Python 3.6.9, NumPy 1.19.2 and h5py 2.10.0.
5. Run `prepare`. Every valid window of five consecutive, overlapping 5-mers produces a 9 nt context and retains the middle row's 20 signal values.

```bash
nanoremix prepare --input sample002.step4.feature.tsv \
  --chemistry RNA002 --format step4 --output prepared002.csv
```

Step4 TSV begins with `id, base, sequence_input, mean, std, median, length` in that order, followed optionally by `kmer_raw`. Each statistic has five pipe-separated values. The alternative `--format extracted` expects `id`, `sequence_input`, `fast5_prefix`, `fast5_order` and the 20 expanded `*_outputN` fields. For fusion, join the retained candidate `id` to the coordinate sidecar before extracting a same-center 81 nt window. The sidecar's reference coordinate identifies the window within the corresponding FASTA record. [Step4 example](docs/guide_20261104/examples/RNA002_step4.tsv) · [Extractor source](upstream_reference/RNA002/python/extract_tombo_step4.py)

<a id="rna004-pod5-to-step6"></a>

### 2.2 RNA004 POD5 to Step6

1. Basecall with Dorado 1.3.0 and `rna004_130bps_hac@v5.2.0` using `--emit-moves` to keep the move table.
2. Align to the matching reference, filter for primary mapped reads, and sort and index the BAM. The documented filter uses `samtools view -F 2308 -q 10`.
3. Extract 9-mer signal features using the bundled extractor, ont-remora 3.3.0 and `9mer_levels_v1.txt`. The template uses `--kmer 9 --step 1 --resquiggle True`. Candidate regions provide `seqnames,start,end,strand`, with zero-based, half-open intervals over 9-mer **start positions**.
4. Check NPZ and coordinate-sidecar row alignment, then merge them into the Step6 CSV. The feature table includes `read_id`, `ref_start`, `sequence_input`, `strand` and 36 statistics. Preserve `ref_name` and `ref_end` for provenance. For a Step6 9-mer, the reference center is `ref_start + 4`.
5. Convert explicit positive-strand Step6 rows. Nine consecutive, overlapping 9-mers produce a 17 nt context and retain the middle row's 36 signal values.

```bash
nanoremix prepare --input sample004.positive.step6.csv \
  --chemistry RNA004 --format step6 --output prepared004.csv
```

The legacy Step6 converter rejects negative or mixed strands, missing or invalid strand values, and reads assigned to multiple references in the same table. To score negative-strand candidates, use a validated adapter to construct an RNA-oriented prepared table with aligned sequence and signal slots. For fusion across splice junctions, obtain the contiguous 81 nt context from the matching transcript or verified CIGAR path. The bundled historical catalog adapters use known truth/catalog fields. [Step6 example](docs/guide_20261104/examples/RNA004_step6_positive.csv) · [Extractor source](upstream_reference/RNA004/preprocess_pipeline/feature_extraction_rna004.py)

The upstream command templates show full calls, intermediate filenames and coordinate joins. [RNA002 and RNA004 command templates](docs/guide_20261104/PREPROCESSING.md)

## 3. Prepare the input table

Use UTF-8 CSV or TSV with a header. Gzip is supported for prediction tables. Each row is one observation of a candidate center on a particular read. Keep a stable, globally unique `record_key` where possible. Retain `sample_id`, `read_id`, `site_id`, `reference_name`, `pos` and `strand` as needed.

### 3.1 Prepared signal features

| Field | RNA002 | RNA004 |
| --- | --- | --- |
| `9mer_final` | 9 nt, center position 5 | 17 nt, center position 9 |
| `mean_output1` through `mean_outputK` | K=5 | K=9 |
| `median_output1` through `median_outputK` | K=5 | K=9 |
| `std_output1` through `std_outputK` | K=5 | K=9 |
| `length_output1` through `length_outputK` | K=5 | K=9 |

The signal values follow the released models' feature and normalization definitions. Values must be finite, with nonnegative standard deviations and lengths. If present, `center_ref_base` must agree with the central base in `9mer_final`. [RNA002 feature example](docs/guide_20261104/examples/RNA002_prepared_with81.csv) · [RNA004 feature example](docs/guide_20261104/examples/RNA004_prepared_with81.csv)

### 3.2 Precomputed probabilities and sequence input

- With `--input-kind probabilities`, Binary needs one probability field per requested strategy and pair, such as `s2_prob_m6A`. Multi needs the complete chemistry-native axis for each requested strategy, 18 classes for RNA002 or 16 for RNA004. Probabilities are finite in [0,1]. Multi rows sum to 1 within an absolute tolerance of `1e-6`.
- For on-the-fly sequence inference, use `--sequence-column sequence81` or a FASTA with explicit identifiers and coordinates. The sequence must have 40 nucleotides on each side, a matching central base and a central 9/17 nt region consistent with the prepared context.
- For precomputed sequence evidence, supply all 15 `second_prob_<label>` columns inline or use `--second-probabilities second.tsv --join-on record_key`. The independent table needs unique, nonempty identity keys. Use the same candidate definition and reference orientation for both sources. [Exact column names and FASTA rules](docs/INPUTS.md)

## 4. Install and check the weights

Use Python 3.10 or 3.11. Install only the dependencies your input route needs.

```bash
python -m venv .venv
source .venv/bin/activate
python -m pip install -e '.[models]' -c constraints-models.txt
```

Use `'.[models,second]'` when you also compute sequence probabilities from 81 nt inputs. To run from precomputed probabilities alone, use `python -m pip install -e .`. On Windows PowerShell, activate the environment with `.\.venv\Scripts\Activate.ps1`.

| Task | RNA002 ZIP | RNA004 ZIP |
| --- | --- | --- |
| Binary | [RNA002 Binary](https://github.com/iris982022/nanoremix_tool_ENG/releases/download/v0.3.0/NanoRemix_v0.3.0_models_RNA002_Binary_20261104.zip) | [RNA004 Binary](https://github.com/iris982022/nanoremix_tool_ENG/releases/download/v0.3.0/NanoRemix_v0.3.0_models_RNA004_Binary_20261104.zip) |
| Multi | [RNA002 Multi](https://github.com/iris982022/nanoremix_tool_ENG/releases/download/v0.3.0/NanoRemix_v0.3.0_models_RNA002_Multi_20261104.zip) | [RNA004 Multi](https://github.com/iris982022/nanoremix_tool_ENG/releases/download/v0.3.0/NanoRemix_v0.3.0_models_RNA004_Multi_20261104.zip) |
| Sequence inference | [Shared SecondGen](https://github.com/iris982022/nanoremix_tool_ENG/releases/download/v0.3.0/NanoRemix_v0.3.0_models_SecondGen_20261104.zip) | Same shared ZIP |

Extract the selected ZIP files at the repository root. They populate `models/` and contain the same `models/SHA256.json` index. To check the RNA004 Binary S2 assets used in the homepage example, run:

```bash
nanoremix doctor --model-root models --model-checksums models/SHA256.json \
  --chemistry RNA004 --task binary --mods m6A --strategy s2 \
  --data-type synthetic --strict
```

For `doctor`, `synthetic` scopes the inventory to third-generation assets. To check fusion with precomputed second-generation probabilities, choose `--data-type biological --second-source probabilities`. For a biological prediction using the signal branch alone, set `--mode thirdgen` on `predict`. [Model layout and compatibility](docs/MODELS.md)

<a id="run-predictions"></a>

## 5. Run predictions

The feature format depends on chemistry. The weight ZIP depends on **both** chemistry and task. Use the combinations in this table to select files and arguments.

| Configuration | Prepared input | `--chemistry` | `--task` | Example `--mods` |
| --- | --- | --- | --- | --- |
| RNA002 Binary | `prepared002.csv` | `RNA002` | `binary` | `m6A,m5C,pseudoU` |
| RNA002 Multi | `prepared002.csv` | `RNA002` | `multi` | `all` |
| RNA004 Binary | `prepared004.csv` | `RNA004` | `binary` | `m6A,m5C,pseudoU` |
| RNA004 Multi | `prepared004.csv` | `RNA004` | `multi` | `all` |

### 5.1 Signal-only example

```bash
nanoremix predict --input prepared002.csv --output results/rna002_binary \
  --chemistry RNA002 --task binary --mods m6A,m5C,pseudoU \
  --mode thirdgen --strategy s2 \
  --model-root models --model-checksums models/SHA256.json --trust-model-files
```

### 5.2 Signal and sequence fusion example

This example reads an aligned `sequence81` column and writes fusion and both component outputs. It requires the RNA004 Multi and SecondGen packages.

```bash
nanoremix predict --input prepared004_with81.csv --output results/rna004_fusion \
  --chemistry RNA004 --task multi --mods all --strategy s2 \
  --data-type biological --sequence-column sequence81 \
  --also-thirdgen --also-secondgen \
  --model-root models --model-checksums models/SHA256.json --trust-model-files
```

To use a separate table of second-generation probabilities, replace `--sequence-column sequence81` with `--second-probabilities second.tsv --join-on record_key`. To perform a trial inference with either command, replace `predict` with `validate` and omit `--output`. For a small trial, point `--input` at a representative subset of already prepared candidate rows. Use a fresh output directory for each prediction.

<a id="interpret-the-output"></a>

## 6. Interpret the output

1. Start with `run_manifest.json`. Confirm the chemistry, task, strategy, evidence modes, class sets, candidate counts and input/model hashes.
2. Read each route's `selection_audit.tsv`. Count accepted `ok` rows, out-of-scope reference families and rejections. For example, `81nt_context_mismatch` indicates that the short and long contexts disagree.
3. Read `predictions.tsv` in the same route folder. `predicted_label` is the accepted call, `prob_<label>` is conditional on the selected classes within the candidate's family, and `evidence_source` identifies the branch or fusion evidence. Rejected rows can retain a `candidate_label` with an empty accepted label.
4. Compare matched routes on `record_key` with a common strategy and label scope. S1/S2 output appears in separate folders. Pure second-generation output is written once under `secondgen/none/`. For site-level summaries, group accepted read-level observations using explicit sample and candidate coordinates and report the coverage and denominator used.

Optionally supply a `--truth-column` for evaluation counts and metrics in the run manifest. The packaged artificial examples check formats and command execution. Biological accuracy and across-tool comparisons require independent evaluation data. [Output and evaluation options](docs/OPTIONS.md)

## 7. CLI parameters and fusion calculations

- [Full CLI reference](docs/OPTIONS.md) covers global flags, input/output controls, preprocessing, model checks, fusion, filtering and all runtime options.
- [Input contract](docs/INPUTS.md) documents FASTA coordinates, probability axes, identity matching and invalid-row behavior.
- [Model assets](docs/MODELS.md) documents the model layout, hash index, dependency versions and scope of preflight checks.

Binary fusion applies a weighted sum of second- and third-generation log-odds, adds the fixed Binary bias, then applies the sigmoid. With both weights set to 1 and zero bias, probabilities of 0.8 and 0.6 combine to approximately 0.857. Multi conditions on the selected reference family and multiplies the weighted probabilities of classes shared by the two branches. The default `extended` mode retains the third-generation mass for classes unique to that branch. `common` restricts selection to classes supported by both. The internal Multi model weights and temperatures are defined by the released manifests.

[Source notice](NOTICE.md) · [License status](LICENSE)
