# RNA002 and RNA004 preprocessing commands

Replace `/path/to/` placeholders with local paths. Use the reference that matches your samples. Run Guppy, Tombo, Dorado, Remora and NanoRemix in their compatible environments.

## 1. RNA002 FAST5 to Step4

The template uses Guppy 6.3.8, ont-fast5-api 4.1.1, Tombo 1.0 and GraphMap 0.5.2. It runs basecalling, single-read splitting, resquiggling, a corrected-event readiness check and 5-mer feature extraction in that order. Tombo operates on the single-read files created for this run.

```bash
set -euo pipefail

TASK_PACKAGE_ROOT=/path/to/NanoRemix
TASK002_RAW=/path/to/RNA002_multi_fast5
TASK002_RUN=/path/to/NEW_RNA002_run
TASK002_REF=/path/to/matching_transcriptome.fa
TASK002_PIPE="$TASK_PACKAGE_ROOT/upstream_reference/RNA002"
TASK002_GUPPY=/path/to/guppy-6.3.8/bin/guppy_basecaller
TASK002_FAST5=/path/to/ont-fast5-api-4.1.1-env/bin/multi_to_single_fast5
TASK002_TOMBO=/path/to/tombo1.0-env/bin/tombo
TASK002_GRAPHMAP=/path/to/graphmap-0.5.2/bin/Linux-x64/graphmap
TASK002_PY=/path/to/RNA002-feature-env/bin/python

test -d "$TASK002_RAW"
test -s "$TASK002_REF"
test ! -e "$TASK002_RUN"
mkdir -p "$TASK002_RUN/01_guppy_rna002"

"$TASK002_GUPPY" -i "$TASK002_RAW" -s "$TASK002_RUN/01_guppy_rna002" \
  --flowcell FLO-MIN106 --kit SQK-RNA002 --recursive --fast5_out \
  --device cuda:0 --num_callers 4 --cpu_threads_per_caller 1

mkdir -p "$TASK002_RUN/02_single_fast5"
"$TASK002_FAST5" -i "$TASK002_RUN/01_guppy_rna002/workspace" \
  -s "$TASK002_RUN/02_single_fast5" -t 8 --recursive

mkdir -p "$TASK002_RUN/03_reference"
cp "$TASK002_REF" "$TASK002_RUN/03_reference/transcriptome.fa"
"$TASK002_TOMBO" resquiggle \
  --basecall-group Basecall_1D_000 --basecall-subgroups BaseCalled_template \
  --failed-reads-filename "$TASK002_RUN/03_failed_reads.tsv" \
  "$TASK002_RUN/02_single_fast5" "$TASK002_RUN/03_reference/transcriptome.fa" \
  --graphmap-executable "$TASK002_GRAPHMAP" --processes 8 --align-processes 8 \
  --overwrite --include-event-stdev

"$TASK002_PY" "$TASK002_PIPE/python/validate_tombo_fast5.py" \
  --fast5-dir "$TASK002_RUN/02_single_fast5" --sample-size 2000 \
  --min-ready 10 --min-ready-fraction 0.01

mkdir -p "$TASK002_RUN/04_step4"
rg --files "$TASK002_RUN/02_single_fast5" -g '*.fast5' \
  | LC_ALL=C sort > "$TASK002_RUN/04_step4/fast5.list"
"$TASK002_PY" "$TASK002_PIPE/python/extract_tombo_step4.py" \
  --fast5-list "$TASK002_RUN/04_step4/fast5.list" \
  --output-prefix "$TASK002_RUN/04_step4/sample.step4" \
  --workers 8 --clip 10 --progress-every 10000 --rna
```

The output prefix produces a `.feature.tsv` table and a synchronous `.coordinates.tsv` sidecar. Inspect the failed-read and feature extraction summaries before conversion. Install NanoRemix in a separate environment, then run:

```bash
nanoremix prepare --input /path/to/NEW_RNA002_run/04_step4/sample.step4.feature.tsv \
  --format step4 --chemistry RNA002 --output /path/to/sample002.prepared.csv
```

For large inputs, split by complete reads to preserve adjacent events used in candidate construction.

## 2. RNA002 reference coordinates for fusion

Join the coordinate sidecar to the candidate's retained middle-row `id`. This example is for a shard that fits in memory. Use the same reference FASTA and record IDs as the resquiggle step.

```python
from pathlib import Path
import pandas as pd

prepared = pd.read_csv('/path/to/sample002.prepared.csv')
coords = pd.read_csv('/path/to/sample.step4.coordinates.tsv', sep='\t')
columns = ['id', 'read_id', 'mapped_strand', 'transcript_id', 'transcript_position0']
if coords['id'].isna().any() or coords['id'].duplicated().any():
    raise ValueError('coordinate id must be nonempty and unique')

payload = coords[columns].rename(columns={
    'read_id': 'context_read_id',
    'mapped_strand': 'context_strand',
    'transcript_id': 'context_reference_name',
    'transcript_position0': 'context_center0',
})
joined = prepared.merge(payload, on='id', how='left', validate='many_to_one', indicator=True)
if not joined['_merge'].eq('both').all():
    raise ValueError('unmatched prepared rows')
joined = joined.drop(columns='_merge')
joined['record_key'] = 'sample002:' + joined['id'].astype(str)
if joined['record_key'].duplicated().any():
    raise ValueError('duplicate record_key')
output = Path('/path/to/sample002.prepared_with_coordinates.csv')
if output.exists():
    raise FileExistsError(output)
joined.to_csv(output, index=False)
```

The joined coordinate is zero-based within the named reference record. Use the mapped strand when the FASTA is in reference orientation. A FASTA already oriented in the read's transcription direction uses its own coordinates and does not need a strand argument.

```bash
nanoremix predict --input /path/to/sample002.prepared_with_coordinates.csv \
  --output /path/to/NEW_002_fused \
  --task multi --chemistry RNA002 --mods all --data-type biological --strategy s2 \
  --context-fasta /path/to/matching_transcriptome.fa \
  --fasta-id-column context_reference_name --center-column context_center0 \
  --coordinate-base 0 --strand-column context_strand \
  --model-root models --model-checksums models/SHA256.json --trust-model-files \
  --on-invalid mark
```

## 3. RNA004 POD5 to Step6

This template uses Dorado 1.3.0 with `rna004_130bps_hac@v5.2.0` and ont-remora 3.3.0. Organize POD5 files under `pod5/split0/`. For multiple splits, repeat the extraction for each split before merging. Use the same reference for the aligner, regions and any later FASTA context extraction.

```bash
set -euo pipefail

TASK_PACKAGE_ROOT=/path/to/NanoRemix
TASK004_RUN=/path/to/NEW_RNA004_run
TASK004_REF=/path/to/matching_genome.fa
TASK004_REGIONS=/path/to/candidate_kmer_start_regions.csv
TASK004_DORADO=/path/to/dorado-1.3.0-linux-x64/bin/dorado
TASK004_MODEL=/path/to/dorado-models/rna004_130bps_hac@v5.2.0
TASK004_LEVELS="$TASK_PACKAGE_ROOT/upstream_reference/RNA004/assets/9mer_levels_v1.txt"
TASK004_PY=/path/to/remora-3.3.0-env/bin/python
TASK004_FEATURE="$TASK_PACKAGE_ROOT/upstream_reference/RNA004/preprocess_pipeline/feature_extraction_rna004.py"
TASK004_MERGE="$TASK_PACKAGE_ROOT/upstream_reference/RNA004/preprocess_human_pipeline/merge_step6.py"

test -d "$TASK004_RUN/pod5/split0"
test -s "$TASK004_REF"
test -s "$TASK004_REGIONS"
test ! -e "$TASK004_RUN/basecall"
mkdir -p "$TASK004_RUN/basecall" "$TASK004_RUN/aligned" "$TASK004_RUN/features" "$TASK004_RUN/step6"

samtools --version > "$TASK004_RUN/samtools.version.txt"

"$TASK004_DORADO" basecaller --device cuda:0 --batchsize 512 --chunksize 2000 \
  --emit-moves --recursive "$TASK004_MODEL" "$TASK004_RUN/pod5/split0" \
  > "$TASK004_RUN/basecall/split0.bam"
samtools quickcheck -v -u "$TASK004_RUN/basecall/split0.bam"

"$TASK004_DORADO" aligner "$TASK004_REF" "$TASK004_RUN/basecall/split0.bam" \
  | samtools view -@ 8 -bh -F 2308 -q 10 - \
  | samtools sort -@ 8 -o "$TASK004_RUN/aligned/split0_sorted.bam" -
samtools quickcheck -v "$TASK004_RUN/aligned/split0_sorted.bam"
samtools index "$TASK004_RUN/aligned/split0_sorted.bam"

"$TASK004_PY" "$TASK004_FEATURE" \
  --pod5_dir "$TASK004_RUN/pod5" --bam "$TASK004_RUN/aligned" \
  --reg "$TASK004_REGIONS" --level "$TASK004_LEVELS" \
  --output "$TASK004_RUN/features" --threads 8 --run-id sample004 \
  --splits 0 --kmer 9 --step 1 --resquiggle True

"$TASK004_PY" "$TASK004_MERGE" --input-dir "$TASK004_RUN/features" \
  --output "$TASK004_RUN/step6/data_df.csv"
```

Regions are a CSV with `seqnames,start,end,strand`. The zero-based interval `[start,end)` enumerates 9-mer left starts. For a target center at position `p`, the nine neighboring 9-mer starts used to build one 17 nt context span `[p-8,p+1)` when all nine are within range. Confirm the coordinate convention and candidate coverage for your reference.

## 4. Convert Step6 and prepare fusion inputs

For explicit positive-strand Step6 rows, run:

```bash
nanoremix prepare --input /path/to/positive_step6.csv \
  --chemistry RNA004 --format step6 --output prepared004.csv
```

The legacy converter rejects negative and mixed strands. The historical `prepare_model_input.py` and `add_splice_aware_context.py` scripts were designed around known catalogs and truth labels. A validated adapter for other samples must preserve candidate identity and orient the 17 nt context and signal slots together. If 81 nt context crosses a splice junction, use a matching transcript or verified CIGAR-derived path. [Known limitations](../KNOWN_LIMITATIONS.md) · [Prediction commands](../../USER_GUIDE.md#run-predictions)
