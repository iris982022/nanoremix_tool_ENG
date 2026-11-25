# Artificial input examples

These fixtures illustrate formats and exercise inference paths. They contain artificial sequences and probabilities.

| Files | Format |
| --- | --- |
| `RNA002_prepared_with81.csv` and `RNA004_prepared_with81.csv` | Prepared feature tables with short context, signal descriptors and `sequence81` |
| `RNA002_binary_both_probabilities.csv` and `RNA004_binary_both_probabilities.csv` | Binary precomputed probabilities and the full second-generation class axis |
| `RNA002_both_probabilities.csv` and `RNA004_both_probabilities.csv` | Multi precomputed probabilities with both branches' complete class axes |
| `RNA002_step4.tsv` | RNA002 Step4 input for `prepare` |
| `RNA004_step6_positive.csv` | Positive-strand RNA004 Step6 input for `prepare` |
| `RNA004_step6_negative_limitation.csv` | Negative-strand fixture for testing the legacy converter's rejection |
| `contexts.fa` | Reference sequences matching the example identities and zero-based centers |

[Input contracts](../../INPUTS.md) · [Prediction examples](../../../USER_GUIDE.md#run-predictions)
