# coloc plugin

Migrated from the legacy `coloc_abf_container` wrapper in nodes-io. One
directory = one plugin family = one git-able unit. The family holds a
single node kind, `coloc_abf`, backed by the official R `coloc` package
(Wallace, CRAN 5.2.3, GPL-3.0-or-later).

## Layout

- `manifest.toml` — node kind `coloc_abf`: 21 flattened params, ports,
  image provenance
- `scripts/abf.sh` — the execution script (R source despite the `.sh`
  name, matching the family convention; `interpreter = "Rscript"` in the
  manifest governs execution), inlined by the loader at startup
- `Dockerfile` — rocker/r-ver:4.5.1 + coloc 5.2.3 from CRAN Archive
  (moved verbatim from `containers/coloc/`)
- `fixtures/coloc_abf_fixture.tsv` — 10-SNP two-dataset smoke fixture
- `test_coloc_abf.sh` — build + smoke test (repointed to this directory)

The image contains only the official R package and its runtime; no
reference panels, no LD matrices, no GWAS inputs are baked in.
Reference: <https://chr1swallace.github.io/coloc/>

## End-to-end workflow

```sh
./test_coloc_abf.sh     # builds localhost/atc/coloc:5.2.3 and smoke-tests it
```

## Migration parity

The golden test (`container-plugin/tests/coloc_migration.rs` in the
autonomics workspace) compares the compiled `ContainerCommandSpec`
against the legacy Rust wrapper's output: image, command, outputs,
resources, timeout, artifact prefix and panels are equal; the script
differs structurally but preserves the exact invocation semantics.

Deltas introduced by the plugin architecture, all following the
documented v0 patterns:

- **Flattened dataset params.** The legacy nested `ColocDatasetSpec`
  objects (`dataset1.beta`, ...) are scalar params (`dataset1_beta`,
  ...); the v0 param DSL has no object or enum types. The trait-type
  enum becomes a string param documented as `quant` or `cc`.
- **Params travel via env.** The legacy wrapper baked values into the R
  source with `__PLACEHOLDER__` substitution; the plugin renders
  `COLOC_*` env values and the script reads them with `Sys.getenv`
  (optional fields gated by `nzchar()`, numbers via `as.numeric()`).
  The legacy wrapper had an empty env map; the compiled spec now
  carries the 21 rendered values.
- **Cross-field validation moved into the script.** Exactly one of
  (beta+varbeta) or pvalues, maf/n required with pvalues, s required
  for cc + pvalues, non-empty snp, and type in {quant, cc} were
  enforced by Rust `validate()` before the container started; the DSL
  cannot express them, so the script enforces the same rules with the
  same messages. Failures now surface inside the container instead of
  at node-build time. The p1/p2/p12 range check moved to DSL bounds
  (`min = 0.0`, `max = 1.0`).
- **kind and artifact prefix drop the `_container` suffix**
  (`coloc_abf_container` -> `coloc_abf`, `/artifacts/coloc_abf`).
  DAG specs referencing the old kind must be regenerated.
