# Reproducing the LAVrefiner heritability analysis

Please Download the complete reproduction package from Releases, extract it, and follow the instructions below. Run all commands from the extracted package directory. This package provides a subset of the paper's heritability analysis using variants from variable number tandem repeat (VNTR) regions on chromosomes 1–10 and expression phenotypes on chromosome 1. It includes the input data, LAVrefiner source, analysis scripts, and PLINK/GCTA executables needed to reproduce this example.

The input variants are provided in `inputs/LAV_VNTR.chr1_10.vcf.gz`.

The analysis compares four genetic relationship matrix (GRM) models: LAV, rLAV+nLAV, VNTR copy number (CN), and a combined model containing VNTR CN and CN-pruned rLAV+nLAV. Reference counts and heritability estimates for this subset are provided in [EXPECTED_OUTPUT.md](EXPECTED_OUTPUT.md).

## Requirements

- Linux x86-64 with Bash and Conda or Miniforge.
- Python 3.12, LAVrefiner 0.3.1 and MAFFT; installation commands are provided below.
- Recommended resources: 4 CPU cores, 16 GiB RAM and at least 10 GiB of free disk space for the environment and analysis outputs.

PLINK 1.9.0-b.7.7 and GCTA 1.94.4 are included in `bin/`. The pipeline always uses these package-local executables through absolute paths resolved from the package directory. LAVrefiner and MAFFT are used from the activated Conda environment. Tool versions and binary SHA256 values are recorded in [tool_manifest.json](tool_manifest.json).

Runtime depends on the available CPUs, storage and parallelism. The first run also builds a TR database index. A runtime estimate for the bundled Linux executables has not been established.

## Installation

Extract the package and enter its root directory:

```bash
cd demo_resources_chr1_10_linux_x86_64

conda create -n LAVrefiner python=3.12
conda activate LAVrefiner

cd LAVrefiner
pip install .
cd ..

conda install conda-forge::mafft
```

## Run the analysis

From the package root, with the Conda environment active:

```bash
export OMP_NUM_THREADS=1
export OPENBLAS_NUM_THREADS=1
export MKL_NUM_THREADS=1
export BLIS_NUM_THREADS=1
export NUMEXPR_NUM_THREADS=1

python run_pipeline.py --threads 4 --workers 4
```

This command uses four LAVrefiner threads and runs up to four REML fits concurrently. Each REML fit uses one GCTA thread. Adjust these two options to match the CPUs allocated to the analysis. The environment variables limit additional threads used by numerical libraries.

The pipeline performs the following steps:

| Step | Analysis |
|---|---|
| 00 | Check input files, environment and sample order |
| 01 | Generate rLAV/nLAV variants and VNTR CN dosage |
| 02 | Convert variants to PLINK files and merge rLAV+nLAV |
| 03 | Build the LAV, rLAV+nLAV and VNTR CN GRMs |
| 04 | Apply CN-based pruning and build the combined GRM |
| 05 | Estimate heritability with GCTA REML |

The combined model removes an rLAV or nLAV when its maximum r² with VNTR CN exceeds 0.05 within a ±1,000,000 bp window. GRM construction uses variant-mean imputation, excludes zero-variance variants, and standardizes with `ddof=1` before calculating `G = Z @ Z.T / P`.

## Results

| Output | Contents |
|---|---|
| `results/mean_hsq.tsv` | Mean heritability and valid-fit counts for each model |
| `results/summary_REML.tsv` | Per-phenotype REML estimates and fit status |
| `results/reml/` | Individual GCTA logs and output files |
| `work/` | Intermediate variants, CN dosage, PLINK files, GRMs and pruning reports |

Compare the generated counts and estimates with [EXPECTED_OUTPUT.md](EXPECTED_OUTPUT.md). Some phenotypes do not yield valid REML estimates because of variance-component constraints; the reference lists the expected valid-fit counts for each model.

Two warnings are expected for this dataset:

- PLINK may report multiple variants at the same position because distinct LAVs can share a genomic position.
- LAVrefiner may retain complex LAVs as nLAV rather than include them in rLAV refinement.

## Package contents

```text
demo_resources_chr1_10_linux_x86_64/
├── bin/
│   ├── plink
│   └── gcta64
├── inputs/
├── phenotype/
│   ├── phenotype_chr1.tsv
│   ├── discrete_covar.txt
│   └── quantitative_qcovar.txt
├── LAVrefiner/
│   ├── LAVrefiner/
│   ├── LICENSE
│   ├── README.md
│   ├── requirements.txt
│   └── setup.py
├── pipeline/
│   ├── 00_check_env.sh
│   ├── 01_run_lavrefiner.sh
│   ├── 02_make_bfiles.sh
│   ├── 03_build_grms.sh
│   ├── 04_build_combined_grm.sh
│   ├── 05_run_reml.sh
│   ├── build_grms.py
│   ├── build_combined_grm.py
│   └── run_reml.py
├── run_pipeline.py
├── README.md
├── EXPECTED_OUTPUT.md
├── tool_manifest.json
└── checksums.sha256
```

`work/`, `results/` and the TR index cache are generated during execution.

## File integrity

To verify the extracted files, run from the package root:

```bash
sha256sum -c checksums.sha256
```

If executable permissions were lost during extraction, restore them with:

```bash
chmod +x bin/plink bin/gcta64 pipeline/*.sh
```
