# NHS Organisation Data Service (ODS) – AWS Ingestion

This repository contains code to ingest **NHS Organisation Data Service (ODS)** reference data into an AWS data lake (Bronze/Silver).

ODS is the national reference for organisation and site codes (e.g. trusts, ICBs, GP practices) used across health and social care in England.  [oai_citation:4‡NHS England Digital](https://digital.nhs.uk/services/organisation-data-service?utm_source=chatgpt.com)

---

## Source dataset

- **Name:** Organisation Data Service (ODS) – reference data  
- **Owner:** NHS England (Organisation Data Service)  
- **Service:** ODS Data Search and Export  [oai_citation:5‡odsdatasearchandexport.nhs.uk](https://www.odsdatasearchandexport.nhs.uk/?utm_source=chatgpt.com)  
- **Licence:** Open Government Licence (OGL v3.0) – confirm on the ODS site.  

Typical ODS extracts include:

- NHS Trust codes and sites  [oai_citation:6‡odsdatasearchandexport.nhs.uk](https://www.odsdatasearchandexport.nhs.uk/referenceDataCatalogue/NHS-Trusts-and-NHS-Trust-Sites_565791609.html?utm_source=chatgpt.com)  
- ICBs / CCGs and their boundaries  
- GP practices and their parent organisations  
- Local authority and other health/social-care organisations

---

## Geographic coverage & granularity

- **Coverage:** England (plus some cross-border and UK-level organisations).  
- **Granularity:** Organisation-level (codes and attributes, not individual patients).

---

## Time coverage & refresh

- ODS reference data is refreshed regularly (often weekly or monthly depending on the pack).  [oai_citation:7‡NHS England Digital](https://digital.nhs.uk/services/organisation-data-service/data-search-and-export/csv-downloads/other-nhs-organisations?utm_source=chatgpt.com)  
- This repo assumes periodic pulls of the latest CSV extracts so Bronze contains a snapshot per load date.

---

## This project’s data model

- **Bronze layer**  
  - Raw ODS CSV files stored with minimal changes.  
  - File-level metadata added (load date, file type).  

- **Silver layer**  
  - Cleaned dimension tables such as:
    - `dim_organisation` – ODS code, name, org type, parent org, start/end dates.  
    - `dim_site` – site code, name, address, parent organisation.  
  - Standardised field types, trimmed codes, consistent date formats.

---

## Repository contents

- **`Bronze_pipeline.ipynb`** – Notebook to ingest ODS CSVs into Bronze S3 paths.  
- **`Untitled*.ipynb`** – Work-in-progress transformation notebooks; can be refactored into final Silver Glue scripts.

(Replace the filenames with your final script names when you clean them up.)

---

## How to use

1. Download the latest ODS CSVs from the ODS Data Search and Export service.  [oai_citation:8‡odsdatasearchandexport.nhs.uk](https://www.odsdatasearchandexport.nhs.uk/?utm_source=chatgpt.com)  
2. Run the Bronze notebook to upload the files into your S3 Bronze layer.  
3. Build and run Silver scripts/notebooks to create curated dimension tables used by all downstream datasets (ASCOF, Oversight, GP datasets, Adult Social Care, etc.).

---

## Attribution

When using ODS data, please follow attribution guidance under the Open Government Licence on the ODS and NHS England websites.  [oai_citation:9‡NHS England Digital](https://digital.nhs.uk/services/organisation-data-service?utm_source=chatgpt.com)
