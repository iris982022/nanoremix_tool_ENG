# Software environments

## NanoRemix inference

Use Python 3.10 or 3.11 for the original model objects. The recommended versions are listed in `constraints-models.txt`. The basic install supports precomputed probabilities through NumPy and pandas. The `models` extra enables third-generation feature inference. The `second` extra enables second-generation sequence inference. [Installation instructions](../USER_GUIDE.md#4-install-and-check-the-weights)

## RNA002 upstream tools

The documented workflow uses Guppy 6.3.8 with the RNA002 HAC model, ont-fast5-api 4.1.1, Tombo 1.0 and GraphMap 0.5.2. The historical Tombo job used Python 2.7. The feature extractor used Python 3.6.9, NumPy 1.19.2 and h5py 2.10.0.

## RNA004 upstream tools

The documented workflow uses Dorado 1.3.0 and `rna004_130bps_hac@v5.2.0`. The Remora environment recorded Python 3.9.20, ont-remora 3.3.0, pod5 0.3.15 and pysam 0.22.1. Record your samtools version when running the command template.

The preprocessing tools have their own environments and installation requirements. [Upstream commands](../docs/guide_20261104/PREPROCESSING.md) · [Historical provenance](../docs/guide_20261104/provenance.json)
