---
description: Thanx data pipeline architecture — Fivetran replication, Snowflake warehouse, dbt transformation layers, Looker BI. Reference for QA dashboard metrics, bug tracking data sources, and engineering analytics.
last_verified: "2026-03-12"
---

# Thanx Data Pipeline

Reference for QA agents that need to understand where engineering metrics come from, how bug data flows into dashboards, and which repos own each layer.

## Pipeline Architecture

```text
External DBs (MySQL, PostgreSQL)
        |
   Fivetran (log-based replication)
        |
   Snowflake RAW database (12 schemas)
        |
   dbt (staging views -> intermediate views -> mart tables)
        |
   Snowflake ANALYTICS database
        |
   Looker (LookML explores, dashboards)
```

## Repositories

| Repo | Purpose |
|------|---------|
| `thanx/thanx-dbt` | dbt models for staging, transformation, and marts (~300 models across 7 domains) |
| `thanx/thanx-looker` | LookML views, explores, and dashboards |
| `thanx/thanx-snowflake` | Terraform-managed Snowflake infrastructure (databases, warehouses, roles) |
| `thanx-ai/data-team-workflows` | Hudson plugin: external analytics workflow orchestrator (see Related Tools below) |

## Fivetran-Replicated Schemas (Snowflake RAW)

`NUCLEUS_THANX_PRODUCTION`, `POS`, `SENDGRID`, `ORDERING_THANX_ORDERING`, `OFFER_PUBLIC`, `JIRA`, `ZENDESK`, `GITHUB`, `GOOGLE_PLAY_THANX`, `STRIPE`, `SMS_PUBLIC`, `FIVETRAN_LOG`

**Not replicated:** Keystone PostgreSQL (no Fivetran connector).

## dbt Model Layers

| Layer | Naming | Materialization | Purpose |
|-------|--------|----------------|---------|
| Staging | `stg_<domain>__<entity>.sql` | View | One source per model, soft-delete filtering |
| Intermediate | `int_<concept>.sql` | View | Joins and business logic |
| Marts | No `fct_`/`dim_` prefix | Incremental (delete+insert) | BI-ready tables consumed by Looker |

### QA-Relevant dbt Domains

| Domain | Key Models | Dashboard |
|--------|-----------|-----------|
| Engineering | `qa_stability` (dynamic table), `bugs_velocity_daily` (incremental), `bugs_resolver_stats`, `bugs_custom_fields` | [BUGS Dashboard (Looker 467)](https://thanx.cloud.looker.com/dashboards/467) |
| Engineering | QA coverage, defect leakage, cycle time models | QA Dashboard (in progress) |

### Seed Cascade Risk

Modified seeds cascade through ALL downstream models via `state:modified+`. One row added to `seed_jira_custom_field_options` triggers builds/tests on `stg_jira__custom_field_options` -> `int_jira__bugs_metrics` -> all bugs mart models.

## Snowflake Structure

| Component | Values |
|-----------|--------|
| Databases | RAW (source), ANALYTICS (transformed) |
| Warehouses | LOADING, TRANSFORMING, REPORTING, DEVELOPING, KEYSTONE |
| Roles | ADMIN, LOADER, TRANSFORMER, REPORTER, DEVELOPER, KEYSTONE |
| Service accounts | Fivetran (LOADER), DBT (TRANSFORMER), Looker (REPORTER), Keystone (PII-safe AI proxy) |

## Looker Layer

- **Main model:** `thanx.model.lkml` (primary business semantics)
- **QA-relevant views:** bug tracking (velocity, stability, custom fields), product adoption, campaign analytics
- **Deployment:** Branch to master via PR, validate in IDE, Content Validator check, deploy via thanx-cli

## Related Tools (External)

**Hudson Plugin** — External analytics workflow orchestrator in `thanx-ai/data-team-workflows`. Not part of this repository. Provides `/hudson:*` commands for dbt/LookML/Looker workflows with 2-tier access:

- **Tier 1 (Explorer):** Search Looker/dbt modeling, build dashboards from existing fields, peer review, documentation
- **Tier 2 (Builder):** All T1 + create/modify dbt and LookML models, PRs, CI monitoring

External commands: `/hudson:hudson` (orchestrator), `/hudson:implement`, `/hudson:pr`, `/hudson:bug-ticket`, `/hudson:trace` (data lineage)

## CI Pipeline (thanx-dbt)

- **Strategy:** `dbt run --select state:modified+1 --defer` then `dbt test --select state:modified+ --defer`
- **Master manifest:** Stored in S3 for comparison
- **Linting:** SQLFluff
- **Branch drift:** Rebase onto `origin/master` before pushing to avoid false modifications
