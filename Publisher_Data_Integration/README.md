# Publisher_Data_Integration

**Purpose:** ETL and integration layer for the Publisher system. Ingests partner files, validates, stages data, and kicks off rendering jobs.

## Components

- `PDI_Azure_Function` — watches file ingress and enqueues processing.
- `PDI_Database` — staging tables and stored procedures for ETL.
- `Publisher_Data_Operations` — core transformation, validation, orchestration.
- `Publisher_ETL` — SSIS/ETL packages.

## Quick links

- [Architecture Overview](docs_ontology/01_Overview/ARCHITECTURE_OVERVIEW.md)
- Components: [PDI_Azure_Function](docs_ontology/02_Components/PDI_Azure_Function.md), [PDI_Database](docs/02_Components/PDI_Database.md)
- [Business Vocabulary](docs_ontology/03_Glossary/BUSINESS_VOCABULARY.md)
- [Runbooks](docs_ontology/05_Runbooks/Reprocess_File.md)

## How to review

Open `docs/` for component READMEs, glossary, ETL sequence and runbooks.
For Previewing of markdowns, click on CTRL + Shift + V on Visual Studio Code.

## Contact

- Author: <Kenneth_Tablang>
