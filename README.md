# UFall paper analyses

Jupyter notebooks for analysing gait variability, environmental exposure, scene embeddings, and eye tracking in the UFall study. The notebooks generate the paper's analysis tables and figures from processed Parquet datasets.

The raw-sensor processing, model inference (SAM3/CLIP), and synchronisation pipeline are not included. This copy contains the notebooks and Python requirements, but **does not include the data or the `helpers/` package imported by several notebooks**. Obtain these from the project maintainers before running the relevant analyses.

## Installation

Use **Python 3.10**, the version recorded in `requirements.txt`. Open a terminal in this repository's root directory, then create and activate an isolated environment:

```bash
python3.10 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
python -m pip install statsmodels ipykernel
python -m ipykernel install --user --name ufall-paper --display-name "Python (UFall paper)"
```

On Windows, use `py -3.10 -m venv .venv` to create the environment and `.venv\Scripts\Activate.ps1` to activate it in PowerShell.

`statsmodels` is needed for the Holm-adjusted sensitivity analyses in notebook 4 but is currently absent from `requirements.txt`. `ipykernel` provides the notebook kernel. The final command registers this environment so it can be selected in JupyterLab.

## Where to place data

Place the processed files directly in the following directories, alongside the notebooks. Keep the filenames unchanged; paths in the notebooks are relative to the repository root.

```text
PaperRepo/
├── README.md
├── requirements.txt
├── 0.build_data.ipynb
├── 1.baseline_walking.ipynb
├── 2.environmental_exposure.ipynb
├── 3.embeddings.ipynb
├── 4.within_participant_environment_gait_variability.ipynb
├── 5.eye_tracking.ipynb
├── 6.case_study.ipynb
├── derived/
│   ├── participants.parquet
│   ├── bouts.parquet
│   ├── frames.parquet
│   └── frame_embeddings.parquet
├── derived_eye_tracking/
│   ├── bouts.parquet
│   ├── frames.parquet
│   └── frame_objects.parquet
├── helpers/                     # Obtain the original project package
│   ├── bout_features.py
│   ├── relational.py
│   └── paper_figures.py
└── results/                     # Created automatically when analyses run
```

Use the original processed datasets, with their existing column names and identifiers. `Participant` and `BoutIndex` link bout records; frame records also use `FrameIndex`. The dense eye-tracking dataset has locally renumbered bout IDs, so its files must stay together in `derived_eye_tracking/`; do not substitute the similarly named files from `derived/`.

The `helpers/` tree above shows the modules imported directly by the notebooks. Obtain the complete package, including any dependencies and supporting files, rather than creating empty replacements. Notebook 0 uses `helpers.bout_features` and `helpers.relational`; notebooks 3, 5, and 6 import `helpers.paper_figures`. These are project modules, not packages supplied by `requirements.txt`.

Notebook 6 also calls `show_video_frames` to create context images. That step requires the external source video/mask assets and the paths expected by the original helper. Those assets and the helper's path configuration are not included here; the Parquet files alone are insufficient for that panel.

## How to run

With the environment activated, launch JupyterLab **from the repository root**:

```bash
python -m jupyterlab
```

Open a notebook, select **Python (UFall paper)** as its kernel, and run all cells from top to bottom. Use a fresh kernel for each notebook. Start with notebook 1; notebook 0 is optional and is only needed to rebuild the Parquet inputs.

Notebooks 1–6 read their own inputs and can be run independently once the required data and helper modules are available. Numerical order is a convenient way to work through the analyses; no analysis notebook consumes another notebook's results.

| Notebook | Required Parquet files | Outputs in `results/` |
| --- | --- | --- |
| `1.baseline_walking.ipynb` | `derived/participants.parquet`, `derived/bouts.parquet` | `table2.csv` |
| `2.environmental_exposure.ipynb` | `derived/participants.parquet`, `derived/bouts.parquet`, `derived/frames.parquet` | `table3.csv`, `table3_participant_exposure.csv` |
| `3.embeddings.ipynb` | `derived/frame_embeddings.parquet`, `derived/frames.parquet`, `derived/bouts.parquet` | `figure3A.png`, `figure3B.png`, `figure3_clusters.csv`, `figure3_representatives.csv` |
| `4.within_participant_environment_gait_variability.ipynb` | `derived/bouts.parquet` | `table4.csv`, `table4_min5_steps.csv`, `table4_min10_steps.csv` |
| `5.eye_tracking.ipynb` | All three files in `derived_eye_tracking/` | `figure4.png`, `figure4_selected_bouts.csv`, `figure4_bout_fractions.csv`, `figure4_participant_fractions.csv` |
| `6.case_study.ipynb` | `derived/bouts.parquet`, `derived/frames.parquet` | `figureS1_metrics.csv`, `figureS1_gait.png`, `figureS1_environment.png`; frame selections and context image when source frames/assets are available |

Rerunning an analysis replaces its existing output files. Notebook 3 loads embeddings into memory and performs nearest-neighbour analysis, so its memory use and runtime depend on dataset size.

### Optional: run all analyses from the terminal

After providing the data and helpers, run the following from the repository root with the environment activated. It uses `nbclient` from `requirements.txt`, runs notebooks 1–6 in separate kernels, and saves executed copies under `results/executed_notebooks/`. It stops on the first error and leaves the source notebooks unchanged.

```bash
python - <<'PY'
from pathlib import Path
import nbformat
from nbclient import NotebookClient

root = Path.cwd()
output = root / "results" / "executed_notebooks"
output.mkdir(parents=True, exist_ok=True)

for path in sorted(root.glob("[1-6].*.ipynb")):
    print(f"Running {path.name}", flush=True)
    notebook = nbformat.read(path, as_version=4)
    NotebookClient(
        notebook,
        kernel_name="ufall-paper",
        timeout=None,
        resources={"metadata": {"path": str(root)}},
    ).execute()
    nbformat.write(notebook, output / path.name)
PY
```

### Optional: rebuild the Parquet inputs

Skip `0.build_data.ipynb` when the processed Parquet files are already available. To rebuild them, first supply the complete original `helpers/` package and the trusted upstream `*_processed.pkl` files. The default source locations are relative to this repository:

| Source directory | Destination | Tables written |
| --- | --- | --- |
| `../UFallV4/OUTPUTS/` | `derived/` | `participants`, `bouts`, `frames`, `frame_embeddings` |
| `../UFallV4/OUTPUTS/single_bout_all_frame/` | `derived_eye_tracking/` | `bouts`, `frames`, `frame_objects` |

Edit `DATASETS` in notebook 0 if your source files are elsewhere, set `REBUILD = True`, and run all cells. The builder searches each configured source directory directly for `*_processed.pkl` files and replaces the corresponding destination Parquet files. With the default `REBUILD = False`, no tables are rebuilt, although the helper imports still need to succeed.

## Reproduction notes

The notebooks preserve several source-analysis conventions that affect interpretation and agreement with the manuscript:

- **Table 2:** uses legacy month-first parsing for ambiguous bout dates to reproduce the reported values. Table 3 uses explicit day-first parsing; the semi-scripted time windows also differ between these analyses.
- **Table 3:** corridor occupancy uses `TightCorridorIndex / (1 + TightCorridorIndex)`, excluding frames without visible floor for this metric. The supplementary equation omits this transform.
- **Figure 3:** the high-variability threshold is frame-weighted, with no explicit free-living/cohort time filter. Representatives are selected by proximity to cluster centroids; the exact hand-selected manuscript images and annotations were not saved.
- **Table 4:** the main analysis uses non-TOPAL bouts with steps, without an explicit free-living time filter. Its clutter measure is `EnvLowerCentralObstacleAreaRatioMean`. The later sensitivity analyses apply minimum step thresholds of 5 and 10 and Holm correction.
- **Figure 4:** uses the available dense eye-tracking subset. Its original selection cannot be fully reconstructed, so an exact match to the manuscript figure is not established. The exact frames for Figure 2 are also not identified.
- **Figure S1:** uses participant `UFALL_PD_0005`, bouts 61 and 38. Environmental values are normalised within each bout across five metrics, not across the population.

## Troubleshooting

- **Missing Parquet file:** check the directory layout above and launch JupyterLab from the repository root.
- **`ModuleNotFoundError: helpers`:** obtain the original project helper package and place it at the root as shown above.
- **`ModuleNotFoundError: statsmodels`:** run `python -m pip install statsmodels` in the environment used by your notebook kernel.
- **Packages installed but imports still fail:** select **Python (UFall paper)** and restart the kernel.
- **Missing columns, insufficient eligible bouts, or missing case-study bouts:** confirm that you have the intended processed datasets. The notebooks require the original schemas and analysis-specific records.
- **Context image generation fails:** verify that the external video/mask assets are available and that `helpers.paper_figures` can resolve their paths.
