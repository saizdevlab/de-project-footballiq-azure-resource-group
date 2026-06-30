# FootballIQ — Azure Data Engineering Pipeline

End-to-end data pipeline that ingests 12 Transfermarkt football datasets (1.9M+ records) into Azure Data Lake, transforms them through a medallion architecture in Databricks Unity Catalog, and serves analyst-ready KPIs to Power BI.

![Architecture Diagram](https://github.com/saizdevlab/de-project-footballiq-azure-resource-group/blob/Dev/architecture/footballiq_complete_architecture.png)

## Overview

This project simulates a production-grade analytics pipeline for a football data platform. Raw CSVs land in Azure Data Lake Storage Gen2, get validated and modeled into a proper star schema in the Silver layer, and aggregate into business-ready Gold tables — all orchestrated through Databricks Workflows and visualized in Power BI.

## Tech Stack

| Layer | Technology |
|---|---|
| Storage | Azure Data Lake Storage Gen2 (Hierarchical Namespace) |
| Compute | Azure Databricks (Serverless + Unity Catalog) |
| Processing | PySpark, Delta Lake |
| Security | Azure Key Vault, Managed Identity (Access Connector) |
| Orchestration | Databricks Workflows |
| Serving | Databricks SQL Warehouse, Power BI |
| Governance | Unity Catalog (3-schema medallion: bronze / silver / gold) |



## Dataset

Source: [Football Data from Transfermarkt](https://www.kaggle.com/datasets/davidcariboo/player-scores) (Kaggle, davidcariboo) — 12 CSV files, weekly-updated, 756 MB total.

| File | Type | Purpose |
|---|---|---|
| `competitions.csv`, `clubs.csv`, `countries.csv`, `players.csv` | Dimensions | League, club, country, and player reference data |
| `games.csv`, `club_games.csv` | Facts | Match results and per-club outcomes |
| `appearances.csv`, `game_events.csv`, `game_lineups.csv` | Facts | Player-level performance per match (1.88M+ rows) |
| `player_valuations.csv`, `transfers.csv` | Facts | Market value history and transfer records |
| `national_teams.csv` | Dimension | Player-to-national-team mapping |

## Key Engineering Decisions

**SCD Type 2 on dimensional tables.** `dim_players` and `dim_clubs` track historical changes — when a player transfers clubs, the old record is closed out (`valid_to`, `is_current = False`) and a new one opens, rather than overwriting history. This preserves point-in-time accuracy for any historical query.

**FK validation with quarantine, not silent drops.** Every fact table is validated against its parent dimensions before being written to Silver. Records that fail validation (e.g. an `appearance` referencing a `player_id` that doesn't exist in `dim_players`) are routed to a `quarantine` table with a `reason` column, rather than being dropped or, worse, loaded with broken joins.

**Unity Catalog External Locations over legacy mounts.** Instead of `dbutils.fs.mount()`, ADLS access is governed through an Access Connector (managed identity) + Storage Credential + External Location — the current Unity Catalog–native pattern, which is auditable and doesn't require storing credentials anywhere in code.

**Idempotent fact loads via `MERGE INTO`.** Fact tables use Delta Lake's `MERGE` for upserts, so re-running the pipeline (e.g. after a schema fix) doesn't duplicate data.

**Three-task orchestration with explicit dependencies.** A single Databricks Workflow runs Bronze → Silver → Gold as dependent tasks, with email alerts on failure — mirroring how a real pipeline would be scheduled and monitored.

![Workflows](https://github.com/saizdevlab/de-project-footballiq-azure-resource-group/blob/Dev/screenshots/workflow-run-success.jpeg )
![Notebooks](https://github.com/saizdevlab/de-project-footballiq-azure-resource-group/blob/Dev/screenshots/notebooks.jpeg)

## Pipeline Layers

**Bronze** — raw ingestion, append-only, all columns kept as strings to protect against upstream schema drift. Adds `_load_ts`, `_source_file`, `_batch_id` for lineage.

**Silver** — type casting, null handling, FK validation, SCD Type 2 on dimensions, bad records quarantined with reason codes.

**Gold** — business aggregates with window functions: `player_season_stats` (goals/assists/cards per player per season, ranked within competition), `club_league_table` (points, goal difference, league position), `top_transfers`, `match_summary`.

## Dashboard

![Power BI stats](https://github.com/saizdevlab/de-project-footballiq-azure-resource-group/blob/Dev/screenshots/powerbi.jpeg)


Three report pages connected via DirectQuery to the Gold layer:
- **League Table** — standings by competition and season
- **Player Stats** — top scorers/assists with competition and season slicers
- **Transfer Market** — transfer fee vs. market value, by position

## Challenges & Lessons Learned

The Databricks Compute UI on new free-trial workspaces now provisions with only a SQL Warehouses tab visible — no classic "Create compute" option by default. Rather than fighting workspace permissions, I used Serverless compute attached directly from the notebook, which required switching ADLS access from the legacy mount-based pattern to Unity Catalog External Locations — a better practice anyway, and one that doesn't depend on a long-running cluster.

Implementing SCD Type 2 correctly required thinking through the `MERGE` logic carefully: the first run creates the table outright, but subsequent runs need to expire the previous "current" record before inserting the new one, otherwise you end up with two rows both marked `is_current = True` for the same player.

## Project Structure

```
├── notebooks/
│   ├── 01_bronze/      12 ingestion notebooks (one per source CSV)
│   ├── 02_silver/      validation, typing, SCD2, FK checks
│   └── 03_gold/        business KPI aggregations
├── architecture/        architecture diagram
├── screenshots/         catalog explorer, workflow runs, Power BI pages
└── docs/
    └── setup-manual.md  full step-by-step Azure + Databricks setup guide
```

## Setup

Full step-by-step instructions — from creating the Azure Resource Group through connecting Power BI — are in [`FootballIQ_Azure_Setup_Manual.pdf`](FootballIQ_Azure_Setup_Manual.pdf).

## Future Improvements

- CI/CD for notebook deployment via Databricks Asset Bundles
- dbt-style data quality tests on Gold tables
- Incremental (rather than full overwrite) Bronze loads using Auto Loader
- Parameterize season/competition filters for a self-service Power BI experience

---

Built as a portfolio project to demonstrate production-style data engineering patterns on Azure. Feedback welcome — open an issue or reach out on [LinkedIn](https://www.linkedin.com/in/sai-sde).
