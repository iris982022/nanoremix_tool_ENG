# Known limitations

1. **RNA004 Step6 conversion.** The legacy converter requires explicit positive strands and rejects reads assigned to multiple references. A validated, strand-aware adapter is needed to prepare negative-strand candidate tables. See the [preprocessing guide](../USER_GUIDE.md#rna004-pod5-to-step6).
2. **RNA004 unknown samples.** The bundled catalog adapters use known labels and/or reference catalogs. Use an adapter that derives candidate coordinates and orientation from the sample and reference. For fusion across splice junctions, use the matching transcript or verified CIGAR path.
3. **RNA002 coordinates.** Join the Step4 coordinate sidecar by candidate ID. Reference and event indexes describe different coordinate systems. Use the appropriate coordinates for the selected FASTA.
4. **Preprocessing compatibility.** The released weights expect the original context and signal definitions. Keep neighboring events when constructing windows. RNA004 regions identify zero-based 9-mer start positions.
5. **Weight coverage.** The release includes full-feature Binary weights. Binary ablations require the corresponding weights. For supported labels, use the exact `catalog` output.
6. **Interpretation.** Confidence is a conditional probability within the selected reference family. Use an explicit read-to-site aggregation procedure to estimate site-level fractions. S1 and S2 outputs remain separate.
7. **Memory and environments.** `--chunk-size` controls main prediction-table batches. `prepare`, FASTA and independent probability tables can still require memory proportional to their inputs. Raw preprocessing tools use separate environments.
8. **Validation.** Bundled fixtures check software and input formats. The documentation and smoke tests do not establish independent biological accuracy. [Verification scope](../verification/README.md)
