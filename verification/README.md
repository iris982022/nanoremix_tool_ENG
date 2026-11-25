# Verify the source distribution

From the repository root, run:

```bash
python verification/verify_distribution.py
python -m pip install -e '.[dev]'
python -m pytest -q
```

The verifier uses the Python standard library and checks the source manifest, expected paths and SHA256 values. The tests check CLI behavior, output contracts, model adapters and packaging. For a model execution check, install verified Release weights and follow the [model instructions](../docs/MODELS.md).

To rebuild a source ZIP:

```bash
python tools/build_source_release.py --output-dir /path/to/new-source-release
python /path/to/new-source-release/NanoRemix/verification/verify_distribution.py
```

The builder selects the paths in `MANIFEST.json` and regenerates integrity metadata. The source ZIP excludes model weights and local test artifacts. Software and package checks cover command behavior. Biological accuracy requires independent validation.
