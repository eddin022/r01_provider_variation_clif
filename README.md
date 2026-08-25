# Provider Variation in Mechanical Ventilation Management (VFD-28, CLIF)

Federated analysis pipeline estimating, for each day of a 28-day window after mechanical
ventilation initiation, the number of at-risk encounters and VFD-28 (ventilator-free days at 28
days) summary statistics — stratified by ICU type and by the eligible-provider pool — to power a
study of provider-level practice variation in ventilator management (R01, JAMA-bound).

VFD-28 methodology follows Yehya et al., *Reappraisal of Ventilator-Free Days in Critical Care
Research*, Am J Respir Crit Care Med. 2019 ([PMC6812447](https://pmc.ncbi.nlm.nih.gov/articles/PMC6812447/)).

This is a **site-run** pipeline: each site runs it against its own local CLIF-formatted data and
shares back only the aggregate outputs described below — no patient-level data leaves the site.

## Requirements

- Python ≥ 3.12 with [uv](https://docs.astral.sh/uv/)
- R with `Rscript` available (used for one step, proning classification — see below). No R package
  installation needed ahead of time; the R script installs any packages it's missing on first run.
- A local CLIF-formatted dataset (Parquet or CSV) — see [Data Requirements](#data-requirements) below
  for the specific tables/columns this pipeline reads.

## Setup

1. Clone the repo and install the Python environment:
   ```bash
   uv sync
   ```
   This installs the pinned dependencies (`clifpy`, `duckdb`, `pandas`, `polars`, `pyarrow`,
   `numpy`, `matplotlib`, `seaborn`) plus `ipykernel` for running the notebook.

2. Create your own site config from the template — `config.json` is gitignored (site-specific,
   never committed):
   ```bash
   cp config_template.json config.json
   ```
   Edit `config.json`:
   ```json
   {
     "site_name": "your_site_name",
     "data_directory": "/absolute/path/to/your/clif/rclif",
     "filetype": "parquet",
     "timezone": "America/Chicago",
     "rscript_path": "C:\\path\\to\\Rscript.exe"
   }
   ```
   - `data_directory` — absolute path to the directory holding your CLIF tables (Parquet or CSV).
   - `filetype` — `"parquet"` or `"csv"`, matching what's in `data_directory`.
   - `timezone` — your site's local timezone (IANA name, e.g. `"America/Chicago"`); all CLIF
     timestamps are UTC and get converted to this timezone for local-day/calendar-date logic.
   - `rscript_path` — **Windows only**. Full path to `Rscript.exe` (e.g. inside a conda env). On
     macOS/Linux, leave it out or ignore it — the pipeline just calls `Rscript` on `PATH`.

3. Confirm R is reachable:
   ```bash
   Rscript --version
   ```
   On Windows, R needs to be a conda-installed R (the proning script relies on the conda env's
   timezone database — `CONDA_PREFIX`). If `Rscript` isn't on `PATH`, set `rscript_path` in
   `config.json` instead.

## Running the pipeline

Open `vfd28.ipynb` (VS Code's Jupyter extension, JupyterLab, or `uv run jupyter lab`) and run all
cells top to bottom (**Restart & Run All**). The notebook is a single linear pipeline — later
steps depend on variables/tables registered by earlier ones, so it isn't safe to run cells out of
order or skip around after an edit.

Right after **File paths**, a pre-flight check confirms every required CLIF table and column is
present and readable — if it fails, fix your `data_directory`/`filetype` config or your local CLIF
files before continuing; nothing downstream will run correctly otherwise.

**Pipeline steps** (in the order they run):

| Step | What it does |
|---|---|
| 0 | Encounter stitching — links related hospitalizations (e.g. ED→inpatient, quick readmission) into one `encounter_block` via clifpy's `stitch_encounters` |
| 1 | Cohort identification — first ICU IMV episode per encounter, plus all exclusions (age, ECMO/trach-at-onset, cardiac arrest/anoxic injury POA, DNI-before-MV, death dated before MV start) |
| 2 | VFD-28 computation per encounter |
| 3 | Provider roster — eligible-provider identification and counts, by (hospital, ICU type, year) and by (hospital, ICU stratum, calendar day) |
| 4 | Daily at-risk landmark loop, Day 1–28 |
| 5 | Daily summary aggregation and output |
| 6 | QC summary — every diagnostic count/statistic from Steps 0–4 |
| 7 | Evidence-based practice indicators (proning, LPV, hypoglycemia avoidance, SBT) |

Step 7's proning indicator (7c) runs `ards_classifier.R` as a subprocess — this is the one step
that needs R. It reads `config.json`'s `data_directory` directly and is scoped to just this
notebook's own analytic cohort (via `cohort_hospitalization_ids.csv`, written just before the R
call). If it fails, its stdout/stderr and `status.jsonl` progress log are printed inline in the
notebook cell — check there first.

A full run can take a while depending on dataset size; there's no checkpointing between steps, so
plan to run it start-to-finish in one sitting (or per-step in separate sessions holding the same
kernel).

## Outputs

- **`output_no_share/`** — patient-level data (the analytic cohort, with dates and IDs) and the
  DuckDB working database. Stays local. Never share, never commit (gitignored).
- **`output_to_box/`** — pure aggregate results, safe to share with the coordinating center:
  - `provider_roster.csv` — eligible providers by (year, ICU type)/(year, hospital)
  - `daily_summary.csv` — N at risk, eligible-provider counts, VFD-28 stats, by (day 1–28) × stratum
  - `ebp_indicators.csv` — pooled evidence-based-practice adherence rates
  - `qc_summary.csv` — cohort/exclusion counts and diagnostics from every step
  - `drop_reasons.md` — day-over-day at-risk attrition diagnostic

Small-cell suppression follows the CLIF federated convention: strata with fewer than 10 at-risk
encounters are flagged (`suppressed_lt10`) rather than removed, so recipients can apply their own
suppression policy.

## Data requirements

The pipeline reads from these CLIF tables (column subset actually used, not the full table
schema) — the pre-flight check in the notebook verifies all of these are present before anything
else runs:

| Table | Columns used |
|---|---|
| `hospitalization` | `hospitalization_id`, `patient_id`, `admission_dttm`, `discharge_dttm`, `age_at_admission`, `admission_type_category`, `discharge_category` |
| `adt` | `hospitalization_id`, `hospital_id`, `in_dttm`, `out_dttm`, `location_category`, `hospital_type`, `location_type` |
| `respiratory_support` | `hospitalization_id`, `recorded_dttm`, `device_category`, `tracheostomy`, `tidal_volume_set`, `tidal_volume_obs`, `fio2_set`, `peep_set`, `peep_obs`, `mode_category`, `mode_name`, `pressure_support_set` |
| `patient` | `patient_id`, `death_dttm`, `sex_category` |
| `ecmo_mcs` | `hospitalization_id`, `recorded_dttm`, `device_category` |
| `hospital_diagnosis` | `hospitalization_id`, `diagnosis_code`, `poa_present`, `diagnosis_primary` |
| `code_status` | `patient_id`, `start_dttm`, `code_status_category` |
| `provider` | `hospitalization_id`, `recorded_date`, `recorded_hour`, `prov_npi` |
| `labs` | `hospitalization_id`, `lab_category`, `lab_value_numeric`, `lab_collect_dttm` |
| `vitals` | `hospitalization_id`, `vital_category`, `vital_value`, `recorded_dttm` |
| `position` | `hospitalization_id`, `recorded_dttm`, `position_category` |
| `medication_admin_continuous` | `hospitalization_id`, `med_category`, `admin_dttm`, `med_dose` |

## Notes

- All tunable thresholds (exclusion windows, provider-eligibility minimums, EBP indicator cutoffs)
  live in one place in the notebook — the **Pipeline Constants** cell — so they can be adjusted
  without hunting through the analysis code.
- Prior to a consortium run, environment and os needs to be less constrained
