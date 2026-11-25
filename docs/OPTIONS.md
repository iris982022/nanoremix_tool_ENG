# NanoRemix CLI Reference

[Homepage](../README.md) · [User guide](../USER_GUIDE.md) · [Input contract](INPUTS.md) · [Model assets](MODELS.md)

## Global options

| Option | Value or default | Meaning |
| --- | --- | --- |
| `--help`, `-h` | Flag | Show usage and commands. After a subcommand, show help for that command. |
| `--version` | Flag | Print the installed NanoRemix version. |

## prepare

Convert a supported legacy feature table into a prepared input table.

| Option | Value or default | Meaning |
| --- | --- | --- |
| `--input` | Required path | Step4 TSV, extracted feature table, or Step6 table. For extracted and Step6, use CSV or a file ending in `.tsv` or `.tab` for tab separation. |
| `--output` | Required new path | Prepared table. A `.tsv` or `.tab` suffix selects tabs. Other suffixes select commas. |
| `--chemistry` | Required: `RNA002` or `RNA004` | Select the context constructor. |
| `--format` | Required: `step4`, `extracted`, or `step6` | `step4` and `extracted` apply to RNA002. `step6` applies to RNA004 and requires explicit positive strands. |
| `--help`, `-h` | Flag | Show preparation options. |

## catalog

| Option | Value or default | Meaning |
| --- | --- | --- |
| `--chemistry` | `RNA002`, `RNA004` (default: both) | List Binary, Multi, fusion and sequence class sets for the selected chemistry. |
| `--help`, `-h` | Flag | Show catalog options. |

## doctor

Check files, model metadata, hashes and dependencies for the selected route. Use `validate` to execute model inference.

| Option | Value or default | Meaning |
| --- | --- | --- |
| `--model-root` | `models` | Directory containing the weights. |
| `--chemistry` | `RNA002`, `RNA004` (default: both) | Limit the check to one chemistry. |
| `--task` | `binary`, `multi` (default: both) | Limit the check to one task. |
| `--mods` | `all` or comma-separated modifications (default: `all`) | Select Binary pairs. For Multi, all components of each checked model are required. |
| `--strategy` | `s1`, `s2`, `both` (default: `both`) | Check selected strategy assets. `s3` aliases `s2`. |
| `--data-type` | `biological` or `synthetic` (default: `biological`) | `synthetic` checks only third-generation models. `biological` honors `--second-source`. |
| `--second-source` | `sequence` or `probabilities` (default: `sequence`) | For `biological`, `sequence` includes the shared network and checkpoints, while `probabilities` omits them. |
| `--condition` | `full`, `mean`, `std`, `length` (default: `full`) | Select matching feature weights. Single-channel checks require `--task binary` and the corresponding ablation weights. |
| `--model-checksums` | Optional JSON path | Check required files against trusted SHA256 values, for example `models/SHA256.json`. |
| `--strict` | Flag, off | Return status 2 if preflight checks fail. The JSON field `preflight_passed` records the result in either mode. |
| `--help`, `-h` | Flag | Show preflight options. |

## predict and validate

These commands share 41 long options. `predict` saves outputs. `validate` executes the same checks and inference and returns a summary. Specify exactly one of `--mods` and `--targets` and exactly one of `--data-type` and `--mode`.

### Input, class selection and routing

| Option | Value or default | Meaning |
| --- | --- | --- |
| `--input` | Required path | Main CSV or TSV, optionally gzip-compressed. |
| `--output` | Required for `predict`, optional for `validate` | New output directory. |
| `--task` | Required: `binary` or `multi` | Dedicated modified/unmodified pairs or reference-family multiclass inference. |
| `--chemistry` | `auto`, `RNA002`, `RNA004` (default: `auto`) | Auto uses the declared chemistry or a consistent short-context length. Explicitly specify chemistry when the input does not identify it. |
| `--mods` | `all` or comma-separated modifications | Select modifications and add unmodified controls according to `--canonical`. |
| `--targets` | Comma-separated labels, Multi only | Exact competitors, including any desired unmodified controls. A modified class requires another class in its family. |
| `--canonical` | `matching` or `all` (default: `matching`) | With `--mods`, add matching unmodified controls or all A/C/G/U controls. |
| `--data-type` | `biological` or `synthetic` | Run fusion or third generation only, respectively. A supplied `data_type` input column must agree. |
| `--mode` | `thirdgen`, `fused`, `both`, `secondgen` | Explicit route. `both` writes third generation and fusion. For signal-only analysis of a biological sample, use `thirdgen`. |
| `--also-thirdgen` | Flag, off | With `--data-type biological`, also save the pure third-generation branch. |
| `--also-secondgen` | Flag, off | With `--data-type biological`, also save the second-generation comparisons supported by the shared sequence model. |
| `--strategy` | `s1`, `s2`, `both` (default: `s2`) | Select the positional or joint third-generation strategy, or write both separately. `s3` aliases `s2`. |
| `--input-kind` | `features` or `probabilities` (default: `features`) | Use signal features or compatible precomputed third-generation probabilities. |
| `--condition` | `full`, `mean`, `std`, `length` (default: `full`) | Select the feature condition. Single-channel modes require Binary and their matching ablation weights. The published ZIPs provide full-feature weights. |

### Model paths and sequence evidence

| Option | Value or default | Meaning |
| --- | --- | --- |
| `--model-root` | Optional directory | Parent of the chemistry folders and optional `secondgen/` folder. Required for third-generation inference from features. |
| `--model-checksums` | Optional JSON path | Verify loaded model files using a trusted SHA256 index. Requires `--model-root` when checking loaded assets. |
| `--trust-model-files` | Flag, off | Required when loading verified local pickle, checkpoint or model Python files. |
| `--second-model-root` | Defaults to `<model-root>/secondgen` | Override the shared sequence-model folder. Keep it within `--model-root` when checking hashes. |
| `--second-probabilities` | Optional CSV or TSV path | Separate table containing all 15 second-generation probabilities. Requires `--join-on`. |
| `--join-on` | Optional comma-separated column names | Identity match for an independent probability table, such as `record_key`. Keys must be nonempty and unique in that table. |
| `--sequence-column` | Optional column name | Read the 81 nt sequence from the main table for inference. With `--second-probabilities`, check the sequence column in that separate table. |
| `--context-fasta` | Optional FASTA path, gzip supported | Extract a window from the named record using the following three required coordinate options. |
| `--fasta-id-column` | Required with FASTA | Main-table column matching the first token of each FASTA header. |
| `--center-column` | Required with FASTA | Main-table column with the candidate center within the named record. |
| `--coordinate-base` | Required with FASTA: `0` or `1` | Declare zero-based or one-based candidate coordinates. |
| `--strand-column` | Optional column name | With FASTA, use `+` or `-` to orient the extracted window. `-` reverse-complements the window. Omitting the option uses the FASTA orientation. |

### Fusion, filtering and runtime

| Option | Value or default | Meaning |
| --- | --- | --- |
| `--fusion-scope` | `extended` or `common` (default: `extended`) | Multi fusion preserves third-only categories in `extended` mode and restricts selection to shared categories in `common` mode. |
| `--second-weight` | Finite positive number (default: `1.0`) | Sequence-branch coefficient in fusion. |
| `--third-weight` | Finite positive number (default: `1.0`) | Signal-branch coefficient in fusion. |
| `--binary-bias` | Finite real number (default: `0.0`) | Fixed log-odds bias for Binary fusion. Only Binary fusion accepts nonzero values. |
| `--min-confidence` | [0, 1] (default: `0.0`) | Minimum probability of the winning selected class. |
| `--min-selected-mass` | [0, 1] (default: `0.0`) | Minimum fraction of reference-family probability mass retained by the selected labels. |
| `--min-second-mass` | [0, 1] (default: `0.0`) | Minimum selected family mass from the second-generation branch where used. |
| `--output-filter` | `all`, `retained`, `modified` (default: `all`) | Save all predictions, accepted predictions, or accepted modified calls. The audit retains all input rows. |
| `--on-invalid` | `error` or `mark` (default: `error`) | Stop or record supported row-level errors. Schema, identity and model errors still stop the run. |
| `--truth-column` | Optional column name | Record evaluation metrics using the supplied annotation column. This input is used only for evaluation. |
| `--chunk-size` | Positive integer (default: `50000`) | Rows per main-table batch. FASTA, an independent probability table and sets of observed keys can remain in memory. |
| `--batch-size` | Positive integer (default: `256`) | Batch size for second-generation sequence inference. |
| `--device` | `cpu`, `cuda`, `mps` (default: `cpu`) | Device for the sequence model, subject to the local PyTorch environment. |
| `--keep-features` | Flag, off | Retain extra input columns, excluding input probabilities and internal underscore-prefixed columns. |
| `--help`, `-h` | Flag | Show command help. |

Thresholds reject rows when the value falls strictly below the minimum. Binary `selected_family_mass` is 1 for its pair, and the sequence branch reports its retained pair mass separately. `confidence` is conditional on the selected labels. Use matched candidate sets and an explicit aggregation procedure for site-level estimates.

For Multi, `extended` retains third-generation-only categories. For example, RNA004 hm5C, o8G and m1psi have third-generation evidence only. The shared sequence model class axis is `A,C,G,U,Am,m1A,m6A,Inosine,ac4C,m5C,Gm,m7G,m5U,pseudoU,Um`. [Complete modification lists and fusion calculations](../USER_GUIDE.md)
