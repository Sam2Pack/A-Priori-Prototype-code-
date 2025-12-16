<h1>Provider Data Validation with NPI Registry</h1>
This project is an agentic pipeline that validates and enriches healthcare provider data using the public NPPES NPI Registry API, and surfaces the results in an interactive Streamlit dashboard. It supports both existing provider directories and new‑provider onboarding flows.

<h2>Features</h2>
Validates provider records against the official NPI Registry using NPI number.​

Detects mismatches in name and practice address between internal records and NPI data.

Computes a confidence score per provider and routes records into AUTO_ACCEPT, REVIEW, or REJECT buckets.

Exposes three CSV queues and a Streamlit dashboard for operations teams and demo.

Optional Flow 2: validates new providers collected via Google Forms / Sheets using the same pipeline.

<h1>Architecture</h1>
<h2>Agents</h2>
</br>
<h2>Data Validation Agent</h2>

Performs structural checks on each row (NPI format, presence of practice address, basic completeness).

Outputs: npi_format_valid, address_present, raw_data_quality.

<h2>NPI Validation Agent</h2>

Calls the NPPES NPI Registry API with the NPI from the CSV row:
https://npiregistry.cms.hhs.gov/api/?number={npi}&version=2.1.​

Extracts official name, practice address, and taxonomy from the JSON.

Compares CSV vs NPI for names and addresses using tolerant string matching.

Outputs:

npi_valid, npi_status (ACTIVE, NOT_FOUND, INVALID_FORMAT, API_ERROR)

npi_name, npi_address, npi_taxonomy_code, npi_taxonomy_desc

name_match, address_match.

<h2>Quality Assurance (QA) Agent</h2>

Combines structural checks and NPI comparison into a numeric confidence_score in 
[0,1].

Heavily rewards valid NPI and matching name, penalizes mismatches.

Maps the score into three buckets:

AUTO_ACCEPT (high confidence)

REVIEW (medium confidence)

REJECT (low confidence).

Outputs: confidence_score, qa_bucket (plus propagated name_match, address_match).

<h2>Public Data Sources Enrichment Agent</h2>

Uses an LLM (Gemini) to infer additional attributes (e.g., human‑friendly specialty, provider type, risk flags) using NPI JSON + CSV context.​

Outputs: provider_type, likely_specialty, risk_flags.

<h2>Information Enrichment Agent </h2>

Merges LLM outputs with NPI taxonomy into final directory‑friendly fields.

Outputs: final_specialty, enriched_flags, and replicated taxonomy fields.

<h2>Directory Management Agent</h2>

Splits the validated dataframe by qa_bucket into three queues:

directory_auto_accept.csv

directory_review.csv

directory_reject.csv

These CSVs feed both downstream processes and the dashboard.

<h2>Master Orchestrator Agent</h2>

Drives the dataset through all agents in order for each row.

Takes a pandas dataframe as input and returns the enriched dataframe plus the three bucket dataframes.

<h1>Workflows</h1>
<h2>Flow 1: Existing Provider Directory Validation</h2>
Input: NPI_Extract.csv (internal provider list with NPI, names, and practice addresses).

Process:

Orchestrator runs Data Validation → NPI Validation → QA → (optional enrichment) → Directory Management.

Output:

Enriched master dataframe.

directory_auto_accept.csv, directory_review.csv, directory_reject.csv for the existing directory.

<h2>Flow 2 : New Provider Credential Verification</h2>
Input: Google Form responses of new providers (NPI, name, practice address, etc.), read from the linked Google Sheet as CSV.

Column headers are normalized to match the pipeline’s expected schema (e.g., Provider NPI → npi).

Process: same orchestrator as Flow 1, run on the responses dataframe.

Output: new providers appended to the three directory CSVs, with an optional source column set to "new".

<h1>Dashboard</h1>
A Streamlit app provides a visual front‑end over the validation results.

Data source:

Reads directory_auto_accept.csv, directory_review.csv, and directory_reject.csv.

Top metrics:

Total providers = sum of rows across the three CSVs.

Counts of Auto‑accept, Review, Reject.

Views and filters:

Radio button to switch between AUTO_ACCEPT / REVIEW / REJECT views.

Sidebar slider to filter by confidence_score.

Table showing NPI, original name columns, npi_name, qa_bucket, confidence_score, npi_status, name_match, address_match.

Charts:

Confidence score distribution for the selected bucket.

NPI status counts (ACTIVE, NOT_FOUND, etc.) to show reasons for rejects.

ngrok exposure (Colab)
When running in Google Colab, Streamlit listens on localhost:8501, which is not directly accessible from the internet. A pyngrok tunnel is used to expose the local port through a temporary HTTPS URL:

pyngrok connects local port 8501 to a public URL like https://xxxx.ngrok-free.app.

Judges and stakeholders access this URL to interact with the live dashboard while the app and data remain inside the Colab runtime.

<h1>Setup and Usage</h1>

<h2>Environment</h2>

Python 3.x

Required libraries: pandas, requests, tqdm, streamlit, pyngrok (for Colab), and optionally google-generativeai.

Run the pipeline (notebook)

Place NPI_Extract.csv (and optionally the Google Form responses CSV) in the working directory.

Open the Colab notebook.

Run all cells that define the agents and master_orchestrator_agent.

Execute the pipeline cell.

Link to google sheets taking input of new providers: https://docs.google.com/spreadsheets/d/e/2PACX-1vQPUXTUYda3P2Y2IZzcWD7hMvtU9JyBkGt5uL0OKiR-Enr37dXFRvJMnOyjwegLvb0pv2Kud7VpSIIC/pub?gid=256121927&single=true&output=csv
