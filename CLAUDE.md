# CLIF — Common Longitudinal ICU Format
## Complete Reference Guide

**Version:** CLIF 2.1.0 (April 21, 2026)  
**License:** Apache 2.0  
**Website:** https://clif-icu.com/  
**GitHub Org:** https://github.com/Common-Longitudinal-ICU-data-Format/  
**Contact:** clif_consortium@uchicago.edu  

---

## 1. Project Overview

CLIF is an open-source data standard for **longitudinal ICU data** enabling high-quality, privacy-preserving multicenter critical care research. It allows analysis across medical institutions without centralizing patient-level data.

**Consortium Statistics (as of 2026):**
- 808,749 unique critically ill patients
- 1,029,400 total hospitalizations (933,058 with ICU stays)
- 62 hospitals across 12 active institutions (15 total, 3 incoming)
- 10 billion+ data points
- Data spanning 2011–2025

**Key Differentiator vs. OMOP:** CLIF captures granular ICU-specific data (ventilator settings, vasoactive drug titrations) that general CDMs omit. CLIF is exclusively for hospitalized adults; no outpatient data.

**Architecture:** Federated — each site retains patient-level data locally. Lead PI develops code, sites execute and return aggregates only (minimum cell count ≥10 for privacy).

---

## 2. Data Model — All 28 Tables

**Current Version:** CLIF 2.1.0  
**Encounter key:** `hospitalization_id`  
**Patient key:** `patient_id`  
**All timestamps:** UTC timezone-aware (`YYYY-MM-DD HH:MM:SS+00:00`)

### 2.1 Beta Tables (16) — Production-Ready

#### `patient`
Demographics (stable across hospitalizations).
```
patient_id, race_name, race_category, ethnicity_name, ethnicity_category,
sex_name, sex_category, birth_date, death_dttm, language_name, language_category
```

#### `hospitalization`
Encounter-level record.
```
patient_id, hospitalization_id, hospitalization_joined_id,
admission_dttm, discharge_dttm, age_at_admission,
admission_type_name, admission_type_category,
discharge_name, discharge_category,        ← "Expired"/"Hospice" = mortality flag
zipcode_nine_digit, zipcode_five_digit,
census_block_code, census_block_group_code, census_tract,
state_code, county_code, fips_version
```

#### `adt`
Admission, Discharge, Transfer — movements within the hospital.
```
hospitalization_id, hospital_id, hospital_type,
in_dttm, out_dttm,
location_name, location_category, location_type
```
*location_category values: ed, ward, icu, stepdown, etc.*

#### `vitals`
One vital per row (long format).
```
hospitalization_id, recorded_dttm,
vital_name, vital_category, vital_value, meas_site_name
```
*vital_category values: temp_c, heart_rate, sbp, dbp, map, spo2, respiratory_rate, height_cm, weight_kg*

#### `labs`
```
hospitalization_id, lab_order_dttm, lab_collect_dttm, lab_result_dttm,
lab_order_name, lab_order_category,
lab_name, lab_category,
lab_value, lab_value_numeric, reference_unit,
lab_specimen_name, lab_specimen_category, lab_loinc_code
```
*Values in CLIF reference units; non-numeric values go in `lab_value`*

#### `medication_admin_continuous`
Rate-based medications (vasopressors, sedation, insulin drips).
```
hospitalization_id, med_order_id, admin_dttm,
med_name, med_category, med_group,
med_route_name, med_route_category,
med_dose, med_dose_unit,              ← rate units (mcg/kg/min, units/hr, etc.)
mar_action_name, mar_action_category, mar_action_group
```
*End of infusion recorded as `med_dose = 0`*

#### `medication_admin_intermittent`
Fixed-dose medications (antibiotics, steroids, boluses).
```
hospitalization_id, med_order_id, admin_dttm,
med_name, med_category, med_group,
med_route_name, med_route_category,
med_dose, med_dose_unit,
mar_action_name, mar_action_category, mar_action_group
```

#### `respiratory_support`
Wide format; ventilator settings and observed parameters.
```
hospitalization_id, recorded_dttm,
device_name, device_id, device_category,     ← IMV, NIPPV, CPAP, HFNC, etc.
vent_brand_name, mode_name, mode_category,
tracheostomy,
fio2_set, lpm_set, tidal_volume_set, resp_rate_set,
pressure_control_set, pressure_support_set, flow_rate_set,
peak_inspiratory_pressure_set, inspiratory_time_set, peep_set,
tidal_volume_obs, resp_rate_obs,
plateau_pressure_obs, peak_inspiratory_pressure_obs,
peep_obs, minute_vent_obs, mean_airway_pressure_obs
```

#### `patient_assessments`
Clinical assessments (sedation, pain, neurological, withdrawal).
```
hospitalization_id, recorded_dttm,
assessment_name, assessment_category, assessment_group,
numerical_value, categorical_value, text_value
```
*assessment_group values: Sedation, Neurological, Pain, Withdrawal*

#### `microbiology_culture`
```
patient_id, hospitalization_id, organism_id,
order_dttm, collect_dttm, result_dttm,
fluid_name, fluid_category,
method_name, method_category,
organism_name, organism_category, organism_group,
lab_loinc_code
```

#### `microbiology_nonculture`
PCR and other non-culture methods.
```
patient_id, hospitalization_id,
order_dttm, collect_dttm, result_dttm,
fluid_name, fluid_category,
method_name, method_category,
micro_order_name, organism_category, organism_group,
result_name, result_category,
reference_low, reference_high, result_units, lab_loinc_code
```

#### `microbiology_susceptibility`
Links to `microbiology_culture` via `organism_id`.
```
organism_id,
antimicrobial_name, antimicrobial_category,
sensitivity_name, susceptibility_name, susceptibility_category
```

#### `hospital_diagnosis`
Post-discharge billing diagnoses (for comorbidity indexing only).
```
hospitalization_id,
diagnosis_code, diagnosis_code_format,   ← ICD10CM or ICD9CM
diagnosis_primary, poa_present
```

#### `code_status`
```
patient_id, start_dttm,
code_status_name, code_status_category   ← DNR, DNI, AND, Full, etc.
```

#### `patient_procedures`
Completed procedures (billing codes).
```
hospitalization_id,
billing_provider_id, performing_provider_id,
procedure_code, procedure_code_format,   ← CPT / ICD10PCS / HCPCS
procedure_billed_dttm
```

#### `position`
Body positioning for proning analysis.
```
hospitalization_id, recorded_dttm,
position_name, position_category         ← prone / not_prone
```

### 2.2 Concept Tables (12) — Developmental

#### `crrt_therapy`
Continuous Renal Replacement Therapy.
```
hospitalization_id, device_id, recorded_dttm,
crrt_mode_name, crrt_mode_category,      ← scuf, cvvh, cvvhd, cvvhdf, avvh
dialysis_machine_name,
blood_flow_rate, pre_filter_replacement_fluid_rate,
post_filter_replacement_fluid_rate, dialysate_flow_rate,
ultrafiltration_out
```

#### `ecmo_mcs`
ECMO / Mechanical Circulatory Support.
```
hospitalization_id, recorded_dttm,
device_name, device_category, mcs_group,
ecmo_configuration_category,
control_parameter_name, control_parameter_category, control_parameter_value,
flow, sweep_set, fdO2_set
```

#### `intake_output`
Fluid balance (long format).
```
hospitalization_id, intake_dttm,
fluid_name, amount,                      ← mL
in_out_flag                              ← 1=in, 0=out
```

#### `invasive_hemodynamics`
```
hospitalization_id, recorded_dttm,
measure_name, measure_category,          ← cvp, ra, rv, pa_*, pcwp, cardiac_output
measure_value                            ← mmHg
```

#### `key_icu_orders`
PT/OT and rehabilitation orders.
```
hospitalization_id, order_dttm,
order_name, order_category,
order_status_name                        ← sent / completed
```

#### `medication_orders`
Links to `medication_admin_*` via `med_order_id`.
```
hospitalization_id, med_order_id,
order_start_dttm, order_end_dttm, ordered_dttm,
med_name, med_category, med_group,
med_order_status_name, med_order_status_category,
med_route_name, med_dose, med_dose_unit,
med_frequency, prn
```

#### `patient_diagnosis`
Problem list / encounter diagnoses (timestamped).
```
patient_id, hospitalization_id,
diagnosis_code, diagnosis_code_format,
source_type,                             ← problem_list / medical_history / encounter_dx
start_dttm, end_dttm
```

#### `place_based_index`
```
hospitalization_id,
index_name, index_value, index_version
```

#### `provider`
Care team continuity (start-stop longitudinal).
```
hospitalization_id, provider_id,
start_dttm, stop_dttm,
provider_role_name, provider_role_category
```

#### `therapy_details`
Physical/occupational therapy sessions.
```
hospitalization_id, session_start_dttm,
therapy_element_name, therapy_element_category, therapy_element_value
```

#### `transfusion`
```
hospitalization_id,
transfusion_start_dttm, transfusion_end_dttm,
component_name, attribute_name,
volume_transfused, volume_units,
product_code                             ← ISBT 128
```

#### `clinical_trial`
```
participant_id, patient_id, hospitalization_id,
trial_id, trial_name, arm_id,
consent_dttm, enrollment_dttm, randomized_dttm, withdrawal_dttm
```

#### `intermittent_dialysis`
*(Schema in full specification — referenced in data dictionary header)*

---

## 3. mCIDE Controlled Vocabularies

**Definition:** Minimum Common ICU Data Elements — 1,400+ standardized clinical variables.  
**Location:** `/mCIDE/` in the CLIF main repo  
**Purpose:** Provides CSV mapping files for local-to-standard value conversion; aligns with FAIR principles and NIH Data Management and Sharing Policy.

**16 Domains:**
- Patient demographics (race, ethnicity, sex, language)
- Location types (ed, icu, ward, stepdown, …)
- Vital categories
- Laboratory names and reference units
- Medication names and routes
- Respiratory devices and modes
- Assessment types (sedation, pain, neurological)
- Diagnoses
- Procedures
- Microbiology organisms and methods
- Transfusion components
- Code status values
- Position categories
- Hemodynamic measures
- CRRT modes
- ECMO/MCS configurations

**Format (per entry):**
```
category | clinical_description | example_1 | example_2 | example_3 | group (optional)
```

**Interactive Explorer:** Available at https://clif-icu.com/ (mCIDE Explorer tool)

---

## 4. Core Python Package — clifpy

**Install:** `pip install clifpy`  
**Docs:** https://common-longitudinal-icu-data-format.github.io/clifpy/  
**Repo:** https://github.com/Common-Longitudinal-ICU-data-Format/clifpy  
**Dev setup:** `uv sync && uv run pytest`

### ClifOrchestrator — Main API

```python
from clifpy import ClifOrchestrator

# Initialize
orch = ClifOrchestrator(
    data_directory='/path/to/clif',
    timezone='US/Eastern'
)

# Validate all tables against schemas
orch.validate_all()

# Access individual tables
vitals_df  = orch.vitals.df
labs_df    = orch.labs.df
meds_df    = orch.medication_admin_continuous.df

# Convert to wide hourly format
wide_df = orch.create_wide_dataset(time_resolution='hourly')

# Clinical calculations
sofa_df        = orch.compute_sofa_scores()
comorbidities  = orch.compute_comorbidities()       # Elixhauser, Charlson
standardized   = orch.standardize_medication_units()

# Encounter stitching (link related ICU stays within time window)
stitched = orch.stitch_encounters(time_window_hours=48)
```

### Key Modules

| Module | Purpose |
|--------|---------|
| `ClifOrchestrator` | Main entry point — load, validate, process |
| `BaseTable` | Foundation class for all table implementations |
| `Tables` | Table-specific implementations |
| `DQA` | Data Quality Assessment / validation |
| `Utilities` | Helper functions |

### Built-In Clinical Calculations

- SOFA scores
- Comorbidity indices (Elixhauser, Charlson)
- Medication unit conversion/standardization
- CDC Adult Sepsis Event (ASE) identification
- Respiratory support analysis (waterfall algorithm)
- MDRO (multidrug-resistant organism) flagging
- Encounter stitching across ICU stays

### Backends
- **DuckDB** — SQL-based high-performance queries
- **Polars** — columnar DataFrame operations
- Parquet files as primary storage format

---

## 5. CLIF-MIMIC ETL Pipeline

**Repo:** https://github.com/Common-Longitudinal-ICU-data-Format/CLIF-MIMIC  
**Version:** 1.2.0 (March 2026)  
**Converts:** MIMIC-IV 3.1 → CLIF 2.1.0  
**License:** PhysioNet Credentialed Health Data License 1.5.0

### Requirements
- PhysioNet credentials with MIMIC-IV access
- MIMIC-IV 3.1 CSV files (~30 GB compressed)
- Python 3.10.5+
- ~46 GB total disk space (15 GB Parquet output + 1 GB logs)

### Setup & Execution
```bash
# 1. Copy template config
cp config/config_template.json config/config.json

# 2. Edit config.json — set MIMIC path, Parquet conversion settings, table selection

# 3. Run (recommended)
uv run python main.py

# 3. Alternative (traditional venv)
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt
python main.py
```

### Output (15 Parquet files)
| Table | Approx Size |
|-------|-------------|
| labs | ~457 MB |
| vitals | ~265 MB |
| medication_admin_continuous | — |
| medication_admin_intermittent | — |
| patient_assessments | — |
| adt | — |
| hospitalization | — |
| patient | — |
| … + 7 more | — |

Logs written to `info/` and `error/` for monitoring.

---

## 6. CLIF-TableOne (Validation & Cohort Summary)

**Repo:** https://github.com/Common-Longitudinal-ICU-data-Format/CLIF-TableOne  
**Purpose:** Validate CLIF 2.1 tables and generate Table One cohort summaries  
**Replaced:** CLIF-Lighthouse (deprecated as of v2.1)

### Workflow
```bash
# Full run with sampling and ECDF distributions
uv run python run_project.py --sample --no-summary --get-ecdf

# Options
# --sample        Fast validation on data subset (~10–15 min runtime)
# --no-summary    Skip summary generation
# --get-ecdf      Generate empirical cumulative distribution functions
```

### Outputs (`output/final/`)
- Validation reports (error classification: accepted/rejected/pending)
- Table One (demographics/clinical characteristics)
- ECDF distributions per variable
- PDF and CSV formats
- Interactive Streamlit web interface available

### Validates All 18 CLIF 2.1 Tables (Parquet format)

---

## 7. CLIF-WorkBench (Containers)

**Repo:** https://github.com/Common-Longitudinal-ICU-data-Format/CLIF-WorkBench  
**Purpose:** Pre-built containerized environments for reproducible CLIF analysis

### ML Image (`clif-workbench:ml`)
- Base: Python 3.12 on Debian Bookworm  
- Size: ~1 GB  
- Packages: pandas, polars, duckdb, scikit-learn, statsmodels, matplotlib, Streamlit

### AI Image (`clif-workbench:ai`)
- Base: NVIDIA CUDA 12.8 on Ubuntu 22.04  
- Size: ~10 GB  
- Adds: PyTorch, transformers, DeepSpeed, XGBoost, Optuna, Weights & Biases

### Usage (Apptainer / Singularity)
```bash
# Pull images
apptainer pull clif-ml.sif docker://clifconsortium/clif-workbench:ml
apptainer pull clif-ai.sif docker://clifconsortium/clif-workbench:ai

# Run ML analysis
apptainer exec --bind /data:/data --bind /project:/project \
    clif-ml.sif bash /project/run.sh

# Run AI/GPU analysis
apptainer exec --nv --bind /data:/data --bind /project:/project \
    clif-ai.sif bash /project/run.sh
```

Releases include SHA256 checksums; offline transfer supported.

---

## 8. FLAIR (Federated Learning Framework)

**Repo:** https://github.com/Common-Longitudinal-ICU-data-Format/FLAIR  
**Purpose:** Privacy-first benchmarking across 17+ hospital sites using CLIF

### Seven Prediction Tasks
| Type | Task |
|------|------|
| Binary | Discharged home |
| Binary | LTACH transfer |
| Binary | Hospital mortality |
| Binary | ICU readmission |
| Multiclass | 72-hour respiratory outcomes |
| Regression | Hypoxic proportion |
| Regression | ICU length of stay |

### Privacy Architecture
- Network requests blocked at Python socket level
- Output scanned for PHI before return
- Cell counts <10 suppressed (HIPAA safe harbor)
- Site PI approval required before any code execution
- Submitter bans for exfiltration attempts

### Workflow
1. Develop and validate on MIMIC-CLIF (public MIMIC in CLIF format)
2. Submit code for federated execution — no code modification allowed at validation
3. Each site outputs: hospitalization IDs, time windows, labels, demographics, train/test splits
4. Results aggregated at coordinating center

---

## 9. ELF & CLIF_MEDS (Interoperability)

### ELF — Event Language Format
**Repo:** https://github.com/Common-Longitudinal-ICU-data-Format/ELF  
**Purpose:** Standardized hierarchical coding for clinical events; enables cross-site model deployment

**Code Format:**
```
{domain}//{level_1}//{level_2}//{level_3}
```

**Example Codes (15 mCIDE domains):**
```
VITAL//spo2
LAB//bicarbonate//mmol/L//bmp
MED_CON//dexmedetomidine//mcg/kg/hr//start
HOSP_DX//ICD10CM//N186
```

### CLIF_MEDS — MEDS Bridge
**Repo:** https://github.com/Common-Longitudinal-ICU-data-Format/CLIF_MEDS  
**Purpose:** Convert CLIF → Medical Event Data Standard (MEDS) with ELF vocabulary

```
# Domain-specific YAML configs in config/
# Maps CLIF source columns to ELF-formatted codes
# Output: domain Parquet files, code metadata, MEDS schema info
```

### CLIF-C2D2 — C2D2 Bridge
**Repo:** https://github.com/Common-Longitudinal-ICU-data-Format/CLIF-C2D2  
**Version:** 0.1.0-alpha (March 2026)  
**Purpose:** Transform CLIF → SCCM C2D2 format  
**Config:** YAML-based with pyproject.toml dependencies

---

## 10. Project Template & Standard Workflows

**Repo:** https://github.com/Common-Longitudinal-ICU-data-Format/CLIF-Project-Template

### Standard Directory Structure
```
project/
├── config/
│   └── config.json           # Site-specific settings (data path, table selection)
├── code/
│   ├── 01_load_cohort.py    # Cohort identification (inclusion/exclusion → hospitalization IDs)
│   ├── 02_descriptive.py    # Table One generation
│   └── 03_analysis.py       # Primary statistical analysis
├── utils/
│   └── clinical_calcs.py    # SOFA, comorbidity, custom utilities
├── output/                   # Aggregate results ONLY — never patient-level data
├── docs/
└── README.md
```

### Setup
```bash
# R environment
Rscript 00_renv_restore.R

# Python environment (preferred)
uv init
cd code && source run.sh
```

### Federated Analysis Pattern (Standard Workflow)
1. Lead PI develops code on local CLIF instance or MIMIC-CLIF
2. Code reviewed and released via pull request with peer review
3. Each consortium site PI approves code before local execution
4. Sites execute locally → generate aggregate statistics (no patient-level export)
5. Aggregates submitted to coordinating center
6. Results combined; all cell counts ≥10 enforced

### Three-Stage Code Structure
1. **QC scripts** — Verify data availability, formats, completeness
2. **Cohort identification** — Inclusion/exclusion criteria → output `hospitalization_id` list
3. **Analysis scripts** — Statistical methods, visualizations, output tables

---

## 11. Quality Control & Outlier Thresholds

**Location:** `/outlier-handling/` in main CLIF repo  
**Location:** `/reference_ranges/` in main CLIF repo

### Outlier Threshold Files
| File | Coverage |
|------|---------|
| `outlier_thresholds_adults_vitals.csv` | Heart rate, BP, temperature, SpO2, respiratory rate |
| `outlier_thresholds_labs.csv` | Electrolytes, glucose, renal function, etc. |
| `outlier_thresholds_respiratory_support.csv` | FiO2, PEEP, tidal volumes |
| `outlier_thresholds_ecmo_mcs.csv` | ECMO flow/sweep rates |
| `outlier_thresholds_crrt_modes.csv` | CRRT blood flow, ultrafiltration |

### ETL Data Quality Standards
- Composite key deduplication on all tables
- Calculate field missingness per table
- Validate date ranges (e.g., `age_at_admission` 18–120 years)
- Remove records with missing critical identifiers (`hospitalization_id`, `patient_id`)
- All identifiers must be VARCHAR strings
- All timestamps must be UTC-converted
- Blood pressure: split MAP/SBP/DBP into separate vital rows
- Vital units: Celsius, percent, mmHg, cm, kg

---

## 12. Governance

**Meetings:** Weekly Thursdays 2 PM CST  
**Quorum:** 50% of steering committee  
**Votes:** Majority for standard decisions; 2/3 supermajority for schema changes or new tables  
**Governance Repo:** https://github.com/Common-Longitudinal-ICU-data-Format/clif-governance

### Leadership
| Role | Person |
|------|--------|
| Steering Co-Chair | Nicholas Ingraham, MD (U Minnesota) |
| Steering Co-Chair | Catherine Gao, MD (Northwestern) |
| Vice-Chair, New Members | Chad Hochberg, MD (Johns Hopkins) |
| Vice-Chair, Software | Kaveri Chhikara, MS |
| Technical Lead | Vaishvik Chaudhari, MS |
| Founding Executive Director | William Parker, MD, PhD (UChicago) |

### Member Institutions (17+)
UChicago Medicine, Rush University, Northwestern Medicine, Johns Hopkins University, University of Minnesota, Emory Healthcare, Tufts University, Michigan Medicine, University of Pennsylvania, Cornell University, UCSF, University of Colorado, Yale University, Harvard University, University of Toronto, Oregon Health & Science University, + others.

### Table Stewardship
Each of the 28 CLIF tables has an assigned Steering Committee Point of Contact (POC) responsible for schema standards and data element evolution.

### Membership Requirements
- Must be PI with independent funding OR active data scientist
- Attend weekly meetings
- Maintain independent local IRB approval
- Complete local data extraction
- Serve as POC for ≥1 data table

### Publication Policy
International Committee of Medical Journal Editors (ICMJE) standards. Federated results only; no patient-level data sharing between sites without explicit Data Use Agreement.

---

## 13. Research Projects & Publications

### Key Publications
| Year | Journal | Title |
|------|---------|-------|
| 2026 | Annals ATS | "Federation, Not Centralization: A New Paradigm for EHR-Based Critical Care Research" |
| 2025 | Intensive Care Medicine | "A common longitudinal intensive care unit data format (CLIF) for critical illness research" |
| 2025 | Critical Care Explorations | "The Epidemiology of ICU Readmissions Across Ten Health Systems" |
| 2025 | medRxiv | "Prone Positioning in a North American Cohort of Hypoxemic Patients on Mechanical Ventilation" |

### Active NIH-Funded Projects
| Project | PI | Institution | Grant |
|---------|-----|-------------|-------|
| Crisis allocation triage scoring | William Parker | UChicago | R01 LM014263 |
| Low-tidal volume ventilation variation | Nicholas Ingraham | U Minnesota | K23 HL166783 |
| Sepsis subphenotypes via vital signs | Sivasubramanium Bhavani | Emory | K23 GM144867 |
| Prone positioning implementation | Chad Hochberg | Johns Hopkins | K23 HL169743 |
| Early mobilization eligibility | Bhakti Patel | UChicago | K23 HL148387 |
| Severe pneumonia prediction | Catherine Gao | Northwestern | K23HL169815 |
| Oncology sepsis prediction | Patrick Lyons | OHSU | K08CA270383 |

### Active Research Topics
- Acute respiratory failure & proning
- Sepsis identification and subphenotyping
- Sedation epidemiology
- CRRT epidemiology
- ICU readmission
- Prolonged respiratory failure
- Antimicrobial consumption variation
- Mechanical ventilation practice variation
- ECMO/MCS outcomes
- Transplant recipient studies
- Novel AI/ML methodologies for ICU prediction

---

## 14. GitHub Repository Index

### Core
| Repo | URL | Purpose |
|------|-----|---------|
| CLIF | .../CLIF | Main docs, DDL schemas, mCIDE, outlier thresholds, ETL guides |
| clifpy | .../clifpy | Official Python package |
| CLIF-101 | .../CLIF-101 | Educational intro (Jekyll site) |
| CLIF-Project-Template | .../CLIF-Project-Template | Standard project structure |

**GitHub Org base:** `https://github.com/Common-Longitudinal-ICU-data-Format`

### ETL Pipelines
| Repo | Purpose |
|------|---------|
| CLIF-MIMIC | MIMIC-IV 3.1 → CLIF 2.1.0 |
| EHR-TO-CLIF | Epic Caboodle/Clarity SQL queries & ETL guide |
| CLIF-C2D2 | CLIF → SCCM C2D2 format |
| CLIF_MEDS | CLIF → MEDS standard with ELF vocabulary |

### Validation & QC Tools
| Repo | Purpose |
|------|---------|
| CLIF-TableOne | Schema validation + Table One generation |
| CLIF-Lighthouse | DEPRECATED — replaced by CLIF-TableOne |
| CLIF_cohort_identifier | Cohort identification tool |

### Compute & Infrastructure
| Repo | Purpose |
|------|---------|
| CLIF-WorkBench | Docker/Apptainer ML and AI images |
| FLAIR | Federated learning benchmarking (privacy-first) |
| CLIF-PROJECT-RESULT-AGGREGATION | Multi-site results aggregation |

### Standards & Interoperability
| Repo | Purpose |
|------|---------|
| ELF | Event Language Format — hierarchical clinical coding |
| clif-governance | Steering committee motions/vote tracker (Jekyll) |

### Research Projects (29+ repos, selected)
| Repo | Purpose |
|------|---------|
| CLIF-proof-of-concept | Federated case studies from ICM 2025 paper |
| CLIF-sipa-model-training | SIPA mortality predictor (R) |
| CLIF-eligibility-for-mobilization | Safe mobilization windows (Python/R) |
| ARF-NIV_TreatmentLocation | NIV treatment location effects |
| CLIF-epi-of-sedation | Sedation epidemiology |
| CLIF_Proning_Incidence_Severe_ARF | Proning incidence analysis |
| Clinical-implications-of-sepsis-definitions | ASE definition comparison |
| Variation-in-High-Flow-Nasal-Oxygen | HFNO post-extubation variation |
| Induction_Variability_RSI | RSI induction variability |
| OHCA-RL | Out-of-hospital cardiac arrest research |
| nippv-pred | NIPPV prediction model |
| clif_vent_variation | Ventilator management variation |
| CLIF_PFvsSF_Performance | Oxygenation severity classification |

---

## 15. Technology Stack

### Python
| Package | Role |
|---------|------|
| clifpy | Official CLIF Python package |
| polars | High-performance DataFrames |
| duckdb | SQL on Parquet files |
| pandas | General data manipulation |
| scikit-learn | Traditional ML |
| statsmodels, scipy | Statistical analysis |
| matplotlib, plotly | Visualization |
| Streamlit | Interactive QC dashboards |
| PyTorch, transformers | Deep learning (AI image) |
| XGBoost, LightGBM | Gradient boosting |
| Optuna | Hyperparameter tuning |
| DeepSpeed | Distributed training |
| Weights & Biases | Experiment tracking |
| uv | Package/environment manager (preferred over pip/venv) |
| pydantic | Schema validation |

### R
| Package | Role |
|---------|------|
| renv | Reproducible environments |
| tidyverse | Data manipulation |
| data.table | High-performance tables |
| survival | Survival analysis |
| ggplot2 | Visualization |
| tableone | Cohort summary tables |
| caret, glmnet, ranger | ML modeling |

### Infrastructure
| Tool | Role |
|------|------|
| Apptainer/Singularity | Container runtime (HPC-compatible) |
| Docker | Container build |
| Parquet | Primary data storage format |
| DuckDB | In-process analytical SQL |
| Jekyll | Static documentation sites |
| GitHub Actions | CI/CD |

### CLIF Data Types
| Type | Use |
|------|-----|
| VARCHAR | All identifiers |
| DATETIME | UTC timestamps |
| FLOAT | Single precision numeric |
| DOUBLE | Double precision numeric |
| INT | Integer values |
| BOOLEAN | 0/1 flags |
| DATE | Date-only fields |

---

## 16. Key Links & Contacts

| Resource | URL |
|----------|-----|
| Website | https://clif-icu.com/ |
| Data Dictionary | https://clif-icu.com/data-dictionary/ |
| GitHub Organization | https://github.com/Common-Longitudinal-ICU-data-Format/ |
| clifpy Docs | https://common-longitudinal-icu-data-format.github.io/clifpy/ |
| CLIF-101 Education | https://github.com/Common-Longitudinal-ICU-data-Format/CLIF-101 |
| Governance Tracker | https://github.com/Common-Longitudinal-ICU-data-Format/clif-governance |
| CLIF Assistant (GPT) | https://chatgpt.com/g/g-h1nk6d3eR-clif-assistant |
| MIMIC-IV on PhysioNet | physionet.org (credentialed access required) |
| Contact | clif_consortium@uchicago.edu |

---

## 17. Quick Reference — Cohort Demographics (Consortium)

| Metric | Value |
|--------|-------|
| Median age | 66 years |
| Female | 45.5% |
| White race | 63.3% |
| Hispanic ethnicity | 5.8% |
| Hospital mortality | 25.8% |
| Median ICU stay | 1.7 days |
| Median hospital stay | 6.4 days |
| Median SOFA score | 3 |
| Median Charlson Index | 2 |
| Invasive mechanical ventilation | 30.6% (315,193 patients) |
| Vasopressor support | 25.3% (260,090 patients) |
| CRRT | 3.4% (34,707 patients) |

---

*Last updated: 2026-05-04. Sources: clif-icu.com, GitHub org Common-Longitudinal-ICU-data-Format.*
