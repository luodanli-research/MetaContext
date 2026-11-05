# MetaContext-CMM

**MetaContext-CMM** is a meta-omics contextualised community metabolic modelling workflow that builds context-specific community-scale metabolic models (CMMs) by integrating MAG-derived network topology, metatranscriptome-informed taxon scaling and reaction constraints, and geochemistry-based environmental constraints.

This repository contains the reproducible MetaContext-CMM implementation for hot-spring microbial communities. Reusable code lives under `workflow/`. The shipped `example/` tree is an example community for learning and testing.

The workflow builds paired baseline and context-specific CMMs for each sample, computes flux-derived metrics—the Elemental Allocation Index (EAI), Anabolic Reallocation Bias (ARB), and Association Coefficient (AC)—to characterise community metabolic states, and runs sensitivity analyses over medium bounds, cooperative trade-off settings, and reaction constraints to test the robustness of model-derived conclusions.

## Installation

From the project root:

```bash
conda env create -f workflow/config/environment.yml
conda activate metacontext
```

Verify the installation:

```bash
python -c "import cobra, cplex, micom, riptide; print('OK')"
```

CPLEX is distributed by IBM and may require a separate license if the PyPI
package cannot be used on the target system.

## Workflow

![MetaContext-CMM workflow](https://github.com/user-attachments/assets/777e4e98-b2ac-406d-a7af-98530b7dbabb)

## Inputs

Each input type below has a corresponding format example under `example/inputs/`.

**Genome-scale metabolic model (GEM) of MAGs:**

```text
01_gems/
├── MAG001.xml
├── MAG002.xml
└── ...
```
Each file should contain one GEM for one MAG in SBML format.

**Taxon-level scaling:**

`01_taxon_ge.csv`: housekeeping-gene-based RNA abundance (TPM), used for taxon scaling in baseline and context CMMs.

`01_taxon_ga.csv`: housekeeping-gene-based DNA abundance (TPM), used only for DNA–RNA concordance quality control (QC).

**Gene expression:**

`02_gene_expression.csv`: gene-level expression profiles for each MAG.

**Medium:**

`03_medium.csv`: medium composition for exchange-reaction constraints, based on measured geochemistry or user-defined settings.

**Sensitivity analysis:**

`04_medium_sensitivity_list.csv`: medium exchange reactions included in medium-bound sensitivity analysis.

`04_reaction_sensitivity_list.csv`: reaction families included in reaction-constraint sensitivity analysis.

**Community guild:**

`04_taxon_guild.csv`: community guild assignment used for Association Coefficient (AC) calculation.

## Run Pipeline

**Example pipeline:**

Three-MAG community (`example/inputs`) for a quick workstation run:

```bash
python pipeline/run_example.py
```
Output layout (under `{example|case_study}/outputs/`; shared by both launchers):

| Stage | Item | Notes |
|---|---|---|
| `01_baseline/` | `<sample>-bsl.pickle` | Baseline CMM |
| `01_baseline/` | `<sample>-manifest.tsv` | Taxon manifest |
| `02_context/` | `context_bounds.csv` | Reaction bounds |
| `02_context/` | `context_species.csv` | QC eligibility taxa |
| `02_context/` | `context_expression.csv` | QC-filtered gene expression data |
| `02_context/` | `context_summary.csv` | QC funnel summary |
| `02_context/` | `<sample>-ctx.pickle` | Context CMM |
| `03_simulation/context/` | `<sample>-{bsl,ctx}-flux.csv` | Formal output flux data |
| `03_simulation/sensitivity_medium/bound<value>/` | `<sample>-bound<value>-{bsl,ctx}-flux.csv` | One folder per medium bound |
| `03_simulation/sensitivity_tradeoff/tradeoff<value>/` | `<sample>-tradeoff<value>-{bsl,ctx}-flux.csv` | One folder per tradeoff fraction |
| `03_simulation/sensitivity_reaction/01_lhs/<sample>/` | `LHS_sample_S*.csv` | Generated Latin Hypercube Sampling (LHS) intervals |
| `03_simulation/sensitivity_reaction/S*/` | `<sample>-reaction-S*-ctx-flux.csv` | One folder per realization |
| `04_analysis/eai/` | EAI CSVs and PDFs | EAI raw data and figure outputs |
| `04_analysis/arb/` | ARB CSVs and PDFs | ARB raw data and figure outputs |
| `04_analysis/ac/` | AC CSVs and PDFs | AC raw data and figure outputs |

**Case-study pipeline:**

Full case-study data and outputs for the hot-spring system are distributed separately via [Zenodo DOI 10.5281/zenodo.20132606](https://doi.org/10.5281/zenodo.20132606).

After download, run `python pipeline/run_case_study.py` from the repo root. Output paths and file formats match the example layout above.

```bash
python pipeline/run_case_study.py
```

**Key options:**

The flags below apply to both `run_example.py` and `run_case_study.py`.

| Option | Role |
|---|---|
| `--sample` | Subset of preset samples to run |
| `--output-root` | Simulation output tree (default: preset `…/outputs`) |
| `--analysis-dir` | Metric output tree |
| `--sensitivity` | `medium` / `tradeoff` / `reaction` (default: all three) |
| `--metrics` | `eai` / `arb` / `ac` (default: `eai arb ac`) |
| `--medium-bounds` | Medium sensitivity uptake bounds (preset differs by launcher) |
| `--tradeoffs` | Tradeoff fractions (preset differs by launcher) |
| `--reaction-realizations` | Latin Hypercube Sampling (LHS) realization count for reaction sensitivity (preset differs by launcher) |
| `--force` | Rebuild baselines, re-infer bounds, rerun simulations and metrics, completed stages are reused unless `--force` is set |
| `--check-only` | Validate inputs and print the plan without solving |
| `--help` | Print the full flag set and defaults |

## Run Individual Modules

Every module is also a standalone CLI under `workflow/`; you can run modules individually or assemble your own pipeline. Run all commands from the repo root. In the **In** / **Out** tables below, bare filenames and stage folders are shorthands for `{example|case_study}/inputs/…` and `{example|case_study}/outputs/…`, respectively. For flags, defaults, and command-line examples, run `python <script> --help` on any module or pipeline launcher.

**1. `workflow/simulation/build_community_model.py`**

Build a baseline CMM from taxon scaling data.

| | Item | Notes |
|---|---|---|
| In | `01_gems/<mag_id>.xml` | GEM per MAG |
| In | `01_taxon_ge.csv` | Default taxon scaling data|
| In | `--sample` | Sample ID |
| Out | `01_baseline/<sample>-bsl.pickle` | Baseline CMM |
| Out | `01_baseline/<sample>-manifest.tsv` | Taxon manifest |

**2. `workflow/simulation/infer_context_bounds.py`**

Run DNA–RNA discordance QC and infer sample-specific reaction bounds for qualified taxa.

| | Item | Notes |
|---|---|---|
| In | `01_gems/<mag_id>.xml` | GEM per MAG  |
| In | `02_gene_expression.csv` | Gene expression data |
| In | `01_taxon_ge.csv`, `01_taxon_ga.csv` | Discordance QC inputs |
| In | `--sample` | Sample ID |
| Out | `02_context/context_bounds.csv` | Reaction bounds |
| Out | `02_context/context_species.csv` | QC eligibility taxa |
| Out | `02_context/context_expression.csv` | QC-filtered gene expression data |
| Out | `02_context/context_summary.csv` | QC funnel summary |

**3. `workflow/simulation/simulate_community_model.py`**

Run simulations using both baseline and context CMMs.

| | Item | Notes |
|---|---|---|
| In | `01_baseline/<sample>-bsl.pickle` | Baseline CMM |
| In | `02_context/context_bounds.csv` | Reaction bounds |
| In | `03_medium.csv` | Exchange constraints |
| In | `--sample` | Sample ID |
| Out | `03_simulation/context/<sample>-{bsl,ctx}-flux.csv` | Formal output flux data |
| Out | `02_context/<sample>-ctx.pickle` | Context CMM |

**4. `workflow/simulation/sensitivity_medium.py`**

Medium-bound exchange sensitivity analysis.

| | Item | Notes |
|---|---|---|
| In | `02_context/<sample>-ctx.pickle` | Context CMM |
| In | `03_medium.csv` | Exchange constraints |
| In | `04_medium_sensitivity_list.csv` | Exchanges to perturb |
| In | `<sample>-ctx-flux.csv` | Formal output flux data |
| In | `--medium-bounds` | Scenario bounds |
| In | `--sample` | Sample ID |
| Out | `03_simulation/sensitivity_medium/bound<value>/<sample>-bound<value>-{bsl,ctx}-flux.csv` | Output flux data, one folder per medium bound |

**5. `workflow/simulation/sensitivity_tradeoff.py`**

Cooperative-tradeoff fraction sensitivity analysis.

| | Item | Notes |
|---|---|---|
| In | `01_baseline/<sample>-bsl.pickle` | Baseline CMM (`bsl` mode) |
| In | `02_context/<sample>-ctx.pickle` | Context CMM (`ctx` mode) |
| In | `--tradeoffs` | Fraction grid |
| In | `--sample` | Sample ID |
| Out | `03_simulation/sensitivity_tradeoff/tradeoff<value>/<sample>-tradeoff<value>-{bsl,ctx}-flux.csv` | Output flux data, one folder per tradeoff fraction |

**6. `workflow/simulation/sensitivity_reaction.py`**

Reaction bound Latin Hypercube Sampling (LHS) reaction-constraint sensitivity analysis.

| | Item | Notes |
|---|---|---|
| In | `02_context/<sample>-ctx.pickle` | Context CMM |
| In | `02_context/context_bounds.csv` | Reaction bounds |
| In | `04_reaction_sensitivity_list.csv` | Reaction families to perturb |
| In | `--reaction-realizations` | LHS realization count |
| In | `--sample` | Sample ID |
| Out | `03_simulation/sensitivity_reaction/01_lhs/<sample>/LHS_sample_S*.csv` | Generated LHS intervals |
| Out | `03_simulation/sensitivity_reaction/S*/<sample>-reaction-S*-ctx-flux.csv` | Output flux data, one folder per realization |

**7. `workflow/analysis/metrics_eai.py`**

Calculate and visualize the Elemental Allocation Index (EAI).

| | Item | Notes |
|---|---|---|
| In | `03_simulation/context/` | `--single-dir`; Formal output flux data |
| In | `03_simulation/sensitivity_*/` | `--batch-dir`; Optional sensitivity batches |
| In | `--sample` | Sample ID |
| Out | `04_analysis/eai/` | EAI raw data and figure outputs |

**8. `workflow/analysis/metrics_arb.py`**

Calculate and visualize the Anabolic Reallocation Bias (ARB).

| | Item | Notes |
|---|---|---|
| In | flux directories | `--single-dir` / `--batch-dir` |
| In | `01_taxon_ge.csv` | `--ge-file` |
| In | `--sample` | Sample ID |
| Out | `04_analysis/arb/` | ARB raw data and figure outputs |

**9. `workflow/analysis/metrics_ac.py`**

Calculate and visualize Association Coefficient (AC).

| | Item | Notes |
|---|---|---|
| In | flux directories | `--single-dir` / `--batch-dir` |
| In | `02_context/<sample>-ctx.pickle` | `--context-cmm-dir` |
| In | `01_taxon_ge.csv`, `04_taxon_guild.csv` | Scaling and guild map |
| In | `--sample` | Sample ID |
| Out | `04_analysis/ac/` | AC raw data and figure outputs |
| Out | `*-ctx-interactions.{metacontext,micom}.csv` | Interaction cache beside flux tables |

## License

This project is licensed under the GNU General Public License v3.0. See [LICENSE](LICENSE) for details.
