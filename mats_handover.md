# Mats - ML Infra Handover (July 8 2026)

## The Landscape

Monorepo, poetry for env mgmt, deps referenced by path.
"Trade-off: some flaky deps due to missing CI/CD coverage (deprioritised due to Databricks complexities.)" What does this mean?
-> Packages live at Head, don't really have setup to check whether deps of packages are impacted by a change. 
-> How often is that a problem? 

Per project pyproject.toml / lock files. Each ML project has different deps.

## Deployment and dev workflow

Docker image for every deploy. ML infra needs OS-level packages that can't be installed by pip alone. Reproducibility between dev and prod.
Docker compose with volume mount for local dev. Hot reloading without rebuilding the image on every change. Files saved automatically get copied to docker image. "Feels like you're working locally."

## Deps

0: chef deploy, data-contracts
1: key-vault, catalog-connector, cheffelo-logging
...

## Design principles:
- Explicit > implicit
- golden path, not cage
- avoid impossible state
- small, single purpose packages
- Pure python. Aligned / data-contracts optional.

## P0: Chef Deploy

Deploy python apps as ACA. Creates Subnet, DNS, KV, sidecar for Datadog agent.
Project owner writes declarative apps.py.

*Why not Terraform:* Terraform is for static ressources. Chef deploy is for dynamic resources. Subnets, DNS + PEs, Secrets and env vars. Things that change per deploy.
I disagree with that. Only the docker image changes. Rest is basically static. Could just have a container app env with all networking setup then just push images to a container registry.

Deploys a container instances if long-running task, otherwise, container app.
apps.py -> StreamlitApp = ACA, GenericApp = ACA, ContainerJob (one-off or scheduled jobs, no ingress) = ACI.
Same interface for secrets, env vars, resources, sidecards, IP rules.

## P0: Data-contracts

Goal: focus on what data you need, not where it lives.
One interface for Unity Catalog, SQL Server, Blob Storage, Redis etc.
Built on Aligned (maintained in-house by Mats) a lightweight feature store.
- Defines features as typed, validated Python objects.
- Materialises to multiple backends (Parquet, Redis, Delta.)

Also, dbt_validate.py check contracts against dbt SQL models. Catches schema mismatches etc. If we remove a column used in a Python project, the CI will notify it on the PR.

Data-contracts only used in Preselector. Data needed for preselector is defined in data contracts. Reci-pick uses data-contracts to materialize data to mloutput schema. That data is used by preselector.

## P1: Key-vault

Unified interface for both databricks and azure chefdp-common-kv secrets, auto-detects runtime.
Loads secrets directly into Pydantic settings.

## P1: catalog-connector

Thin wrapper around databricks connect for reading and writing data.
Lazy value resolution - env vars resolved at connection time, not import time.

# P1: cheffelo-logging
Standardized logging for DataDog and streamlit (st.warning, st.error)
Datadog logs are structured JSON.
Also possible to export metrics to Datadog or setup alerts.

Chef-deploy always create a sidecar for Datadog. But not automatically setup. So we create the sidecar even if we don't use it?
*Datadog -> logs -> source: python*

# P1: model-registry
ML lifecycle interface built on MLflow.
Builder pattern with type hints. 
Auto-loads Unity Catalog features at inference time.
`InMemoryRegistry` for testing without touching MLflow.

# P2: blob-storage, service-bus, embeddings
Thin wrapper around blob storage, service-bus and embeddings.

BLOB STORAGE -> Not as-code. 

# P3: Utilities
pydantic-argparsers: generates argpase CLI from a Pydantic model.
pydantic-form: generate streamlit forms from pydantic.
container-cleaner: CRON job cleaning stale Docker images from Azure Container Registry.

# Alerting:
Datadog -> Monitoring -> Preselector has a few alerts. All setup through UI. One alert making sure the preselector workers are listening for requests.
Currently tagging 

# Agathe: Slow fetching data from Databricks
Currently, very slow to start a job in Databricks (5-10min to start up the compute) because using Docker. When running Docker on Databricks, they need a custom docker flavor, so 2 images, one Azure, one Databricks. Starting a Databricks cluster with a Docker image takes a long time. Provision one cluster per job.
Stephen working on some enablement packages, e.g: using DuckDB for local dev.

# Next big things:
Improve and extend the deploy stuff, make that easy to use.
Reduce diff between data-science and software-engineering. e.g: Making it easy to setup service bus, setting up triggers for CRON job. Setting up monitoring / observability. 