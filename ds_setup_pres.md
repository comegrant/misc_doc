# DS Setup

## Package Management

### We're using `uv` instead of `poetry` now.
- New projects and packages created with `chef create` will use `uv`
- All packages moved to `uv`, existing projects are still on `poetry`
- Docs updated with guides on [setting up uv and custom package feed](https://docs.data.cheffelo.com/setup/installation.html#uv) etc
- Use `uv run` to run commands and `uv add` to add dependencies.

### Custom feed for internal packages
- Merging a PR that touch a package in `sous-chef/packages/` will publish it to the feed.
- When a package is published, the patch version number (`major.minor.patch`) is bumped automatically.
- Major and minor version number are human-managed. Bump them if you push potentially breaking changes to an internal package.
- Dependent projects and packages will not pick up updates until re-locked, which requires a PR.
- You can add dependencies on internal packages with `uv add`.
- Internal packages are now prefixed with `cheffelo-` by default, you need to run `uv add cheffelo-constants`. `uv add constants` will install a random package called `constants` from PyPi. The prefix is here for disambiguation.


## Databricks or Azure

I think the best option is to use a mix of Databricks and Azure.

### Done in Databricks: 

|   | what | why  |
|---|---|---|
| Scheduled jobs  | Any code running on a fixed cron schedule. | <ul><li>We want to keep orchestration in Databricks</li><li>Common use case that should have simple setup</li></ul> |
| Batch Data Processing | Reading, processing and writing data, all in Databricks |  <ul><li>Moving data out of Databricks and back is slow and expensive</li><li>Databricks is really good for batch data processing. You can mix python and SQL (a.k.a: single-node and spark)</li><li>Respect data access control configured in Databricks.</li></ul> |
| Model Lifecycle | MLflow everything | <ul><li>Databricks gives us hosted MLflow and we're generally happy with it</li></ul> |


### Done in Azure: 

|   | what | why  |
|---|---|---|
| Streamlit Apps  | Streamlit apps (ACA). | <ul><li>Databricks Apps are very nice but too expensive</li></ul> |
| Azure Container Apps  | Containerized services (FastAPI, random servers etc.) | <ul><li>Azure is generally more flexible for custom stuff</li><li>Try Databricks ML / Lakebase for generic API use cases</li></ul> |
| Other infra | Serverless functions, cloud storage, service buses, logging for ACAs...  | <ul><li>Other use cases not well supported by Databricks. TBD what.</li></ul> |

### Not sure:

|   | what | why  |
|---|---|---|
| OLTP | Random point read & writes, high concurrency, low latency. Workloads which need a row-oriented database like Postgres. | <ul><li>Databricks offers a managed postgres (lakebase.) No custom infra</li><li>Good integration with lakehouse (regular Databricks.)</li><li>I haven't looked into it enough to know if it will work for us.</li></ul> |
| Real-time inference | Turning an MLflow model into an API endpoint for real-time inference. | <ul><li>Should try Databricks ML</li><li>Models already live in Databricks MLflow</li><li>No existing use case, investigate when it comes up.</li></ul> |


## Databricks Setup:

### Speed up the start time of Databricks jobs:
- No longer using docker in Databricks
- Use serverless compute by default
- Use ML Base Environment with [bunch of pre-installed libs](https://docs.databricks.com/aws/en/release-notes/serverless/environment-version/five#-ml-base-environment) to save install time

### Serverless defaults:
- The project template uses Serverless compute by default for jobs. More specifically, the ML Base Environment.
- Databricks serverless environment comes with a [bunch of libraries installed by default](https://docs.databricks.com/aws/en/release-notes/serverless/environment-version/five#-ml-base-environment)
- You can install extra libraries
- Accidentally upgrading or downgrading a pre-installed library can break stuff
- New projects and packages created with `chef create` come with a list of constraints matching the versions and libraries. So if you run `uv add pandas` in your project or package, it will default to the version installed on the Databricks Serverless environment (2.2.3) instead of pinning the latest version, which would mutate the serverless environment and potentially break things
- The dependencies for the serverless environment are generated automatically from the uv lockfile anytime you run `databricks bundle deploy`.
- `databricks bundle deploy` will fail loudly if it finds a dependency conflict instead of deploying potentially buggy code

### Beyond defaults:
- As long as you use `uv add` to add dependencies without forcing version numbers, you can be confident that your project will run fine on Databricks serverless compute.
- The goal of this setup is to provide **safe defaults**, not prevent people from managing dependencies for their projects. You're free to delete all the constraints we template into `pyproject.toml`, specify exactly which versions you want, use docker etc.
- Databricks serverless compute has a limit of 16GB memory*. If that's a problem, you can configure compute for your project. **Our data is small enough that this really shouldn't be necessary.** Before making the compute bigger, try to:
    - Optimize your code
    - Read & process data in chunks instead of loading everything in memory at once
    - Do the heavy work in spark ([pyspark](https://docs.databricks.com/aws/en/pyspark/basics), [spark SQL](https://docs.databricks.com/aws/en/pyspark/reference/classes/sparksession/sql), [spark pandas](https://docs.databricks.com/aws/en/pandas/pandas-on-spark))

<br>

**Databricks uses Spark for distributed processing.  
Serverless compute on Databricks is made of one computer called the Driver Node which handles running your code and distributing work to other computer (called Worker Nodes or Spark Workers.)*  

*When you do transformation in regular pandas instead of one of the multiple APIs for spark ([pyspark](https://docs.databricks.com/aws/en/pyspark/basics), [spark SQL](https://docs.databricks.com/aws/en/pyspark/reference/classes/sparksession/sql), [spark pandas](https://docs.databricks.com/aws/en/pandas/pandas-on-spark)), or training models with plain scikit-learn instead of using [distributed training](https://docs.databricks.com/aws/en/machine-learning/train-model/distributed-training/), you're making the driver node do all the work. Which kinda defeat the purpose of using a distributed processing platform.*

*The 16GB memory limit only applies to the driver node. For work done in spark, serverless compute can auto-scale by adding more workers node. So if you're using Spark, you don't have to worry about running out of memory.
Doing stuff in plain python on the driver node is simple and it's completely fine for small workloads.*  
*But give distributed data processing a try before creating huge single-node compute.*

## Azure Setup:

I think our general philosophy should be to try and use Databricks features first, and fall back to Azure if Databricks does not satisfy our use cases. I'm very happy to build custom things in Azure. But we need a good reason for doing so, because things done in Databricks will generally require less infra work and break less often.

### Configuring infra in projects

We need Azure infra to be done in Terraform to be manageable. However, we don't want to force everyone to learn terraform and make terraform PRs. So we'll use a setup similar to `databricks asset bundles`. If you want Azure resources for your project, you'll add an `azure.yml` file which will serve as input for terraform to create the necessary infra.

This is what your `projects/my-project/azure.yml` would look like for deploying a streamlit app as an Azure Container App for example:
```yaml
# projects/my-project/azure.yml
apps:
  - name: my-streamlit-app
    kind: streamlit
    entrypoint: app/main.py
    secrets:
      OPENAI_API_KEY: openai-api-key   # fetches secret called "openai-api-key" from Key Vault and sets them as ACA secrets
    env:
      LOG_LEVEL: info                  # plain, non-secret values that will be mapped to env vars
```

The schema of `azure.yml` can easily be extended to support more types of Azure resources & config options for common use cases. Happy to discuss it.

```yaml
apps:
  - name: my-fastapi-server
    kind: fastapi
    entrypoint: project.main:app
  - name: custom-aca
    command: python3 -m http.server 8080

function:
  - name: some-serverless-function
    maximum_instance_count: 50
    instance_memory_in_mb: 2048
```

Merging changes to any `azure.yml` file in a project will trigger a Github Action that will run terraform to create the necessary infra. The Data Science infra will be managed by a separate Terraform deployment to be independent from the rest of our infrastructure as much as possible. 
We'll deploy to test then prod, with approval gates in the middle. We can discuss removing the approval gate if things run smoothly.

Merging changes to a project containing an `azure.yml` file will also build and publish the new image to ACR and update the app.

I think this is a decent way to have everything done in Terraform without forcing everyone to do Terraform.

### Dependency groups

A project will have a single `pyproject.toml` and lockfile. However, one project may contain a mix of Databricks jobs and Container Apps with different dependencies. For example, your project may have a streamlit app and some Databricks jobs which do not need streamlit installed.  
This is handled by having an extra `azure` dependency group for libraries that will get installed on the docker image deployed in Azure Container App but skipped on Databricks compute.

```pyproject.toml
[project]
dependencies = [          # Installed on both Databricks jobs AND docker image for Azure Container App
    "pandas>=2.2",
    "cheffelo-model-registry>=0.2.0",
]

[dependency-groups]
dev = ["pytest", "ruff", "pyright"]             # Installed locally and in CI only
azure = ["streamlit", "fastapi", "uvicorn"]     # Installed in the image for the Azure Container App
```

Use `uv add --group azure streamlit` to add a dependency that's only relevant to container apps (here: Streamlit.)
Having a single dependency group for all the Azure stuff means some unnecessary library installs if you have multiple ACAs in one project, but the simplification is worth it. If you really want to avoid that, you can break a project down into multiple projects & packages.

### local dev

Here's a streamlit app, as defined in `azure.yml`
```yaml
# projects/my-project/azure.yml
apps:
  - name: my-streamlit-app
    kind: streamlit
    entrypoint: app/main.py
    secrets:
      OPENAI_API_KEY: openai-api-key
    env:
      LOG_LEVEL: info
```

To run it locally, create a `.env` file in your project with the necessary secrets and variables then run:
`uv run --env-file .env streamlit run app/main.py`

We can have a chef CLI command that reads `azure.yml` to automatically fetch secrets from KV and run your app etc.

For Databricks, use `databricks bundle deploy` to publish your bundle and run your jobs from Databricks.
As long as your databricks jobs use regular python files and avoid features exclusive to Databricks notebooks, you can also just `uv run` stuff.  
You may use docker for local dev, but I wouldn't recommend it. `uv` maintaining the lockfile and the virtual environment should be enough for portability. And building docker images slows down the iteration loop.


## Notes, Limitations, TODOs:
- Some dependencies need to be updated before our internal packages are serverless-compatible by default (e.g: model-registry 0.2.0)
- This doesn't guarantee stable URLs for published apps. It should help, because the infra will be more solid and change less often. But guaranteeing stable URLs requires some DNS setup. Doable once everything is done in terraform.
- This probably involves rebuilding all the infra for the streamlit apps. Good opportunity to move them to the Data Platform subscription.
- Published streamlit Apps will be accessible to everyone in our Azure tenant and login-gated by default. We're switching to built-in auth and not requiring app code to handle login. Can add access control for streamlit apps in the future if needed.
- Bunch of things that can be simplified with some chef cli commands etc. 
- We have a Test and Prod environment in Azure, but no Dev to speak of. Can look into it but hopefully local dev is good enough.
- Can improve container supports for local dev, devcontainers etc.
- Azure Container Apps defined in `azure.yml` will use a single default dockerfile that sets up `uv` and install dependencies. But we should have an escape hatch and use a project's dockerfile if there is one.
- Can have a key in `azure.yml` to list additional native libs that should be installed in the image. But if we need to do that, we might be overcooking it.
- ACAs have a cold start of 30s to 1min. Keeping an app always-on but idle isn't too expensive. Might want a setting for that.
- Would be nice to have a smoke test in the CI for docker images. Doable for Streamlit / FastAPI but maybe hard to make generic. Might slow down CI too much.
- Should use RBAC in Key Vault so deployed ACAs only have permissions to read the secrets they need.
- The package feed doesn't make internal packages installable in Databricks notebook. Might be nice to have a real PEP 503 feed. Would also fix the issue of the SAS not being revokable.
- Look into something like Renovate to automatically relock projects and pick up patch updates.
- Finish reserving names on PyPi, share ownership, yank (if convenient.)
- Get rid of ODBC driver in data-contracts in favor of lakehouse-federation, or just use dbt models, add missing data to dbt models if needed.
- Check if high-memory serverless is available in DABs now.
