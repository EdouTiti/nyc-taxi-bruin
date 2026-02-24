# nyc-taxi-bruin

### Project overview
This hands-on tutorial guides you through building a complete NYC Taxi data pipeline from scratch using Bruin - a unified CLI tool for data ingestion, transformation, orchestration, and governance.

We will create a pipeline with 3 layers:
1. Ingestion (Python + seed): fetch monthly NYC TLC parquet files + load a small lookup CSV
2. Staging (SQL): clean + deduplicate + enrich (join lookup)
3. Reports (SQL): aggregate metrics + run quality checks
Bruin runs assets in dependency order and supports validation (dry-run style) before executing.

### Project sources
NYC Taxi dataset (2019-2020)

### tools used: 
- Bruin for Data platform & pipeline orchestration
- SQL for Querying and transformations
- Python for Data processing
- R for Analytics & visualization
- GitHub Codespaces for Cloud development environment

### Steps
- Use GitHub Codespaces to do the project.
- Create a new repository named as "nyc-taxi-bruin"
- Install Bruin in Codespaces in terminal bash of docker and verify the version
```bash curl -LsSf https://getbruin.com/install/cli | sh
bruin version

- Initialize the project zoomcamp folder
```bash bruin init zoomcamp my-taxi-pipeline
      cd my-taxi-pipeline 
      ls -la
- Then, the pipeline is created. You should get started
  ```bash bruin validate my-taxi-pipeline
