# Base Env and deps
- Switch to serverless was very smooth using a base environment.
- Base environment = pre-built and cached python env, can attach to compute to skip install time and avoid managing deps manually.
- Base env for serverless job was a preview feature. "Serverless workspace base environment support in Jobs." Turned it on in dev.
- Serverless with base env takes about 5 seconds to start.
- Some internal libraries have deps which haven't been updated in a while (`time-machine`) and conflict with the base env, hasn't been a problem, but good to update nonetheless.
- I just used the default Databricks ML base environment, which had all the necessary libs.
- We can make our own base env(s) to remove deps we don't use (CUDA etc.)
- List of installed libs and version in ML base env can be found [here](https://docs.databricks.com/aws/en/release-notes/serverless/environment-version/five#ml-environment) can download the `requirements.txt`.
- There's also a [standard base env](https://docs.databricks.com/aws/en/release-notes/serverless/environment-version/five#installed-libraries) which is lighter and come with less stuff pre-installed.
- Base env is versioned and all libs in it are pinned. Kinda nice to only have to worry about env version instead of individual lib versions.


# Pathing

There are some path shenanigans that come with working with bundles. Mostly because we import internal packages by path.
Having our own pip repository and pip installing internal packages would save us from those. But that's its own decision.

When I run `databricks bundle deploy`, reci-pick ends up in Databricks at:
`Workspace/Users/come.grant@cheffelo.com/.bundle/reci-pick/dev`

`come.grant@cheffelo.com` and `dev` come from my config for the Databricks CLI, my default profile (configured in `~/.databrickscfg`) points to dev and uses a personal access token.
```
[DEFAULT]
host  = https://adb-4291784437205825.5.azuredatabricks.net
token = dapi-mypersonalaccesstoken-2
```

## Bundle structure

This is what the bundle structure looks like on Databricks. I also included some aliases like `${workspace.file_path}` that you can use to dynamically references paths in DAB files like `databricks.yml` or job files.
We do not care about `artifacts/` and `state/` in this case.
```
/Workspace/Users/<you>/.bundle/reci-pick/<target>/  ← ${workspace.root_path}
├── files/                                          ← ${workspace.file_path}
│   ├── packages/
│   │   ├── catalog-connector/                      ← pip-installed by the env spec
│   │   ├── constants/
│   │   └── time-machine/
│   └── projects/
│       └── reci-pick/
│           ├── predict_job        (NOTEBOOK)       ← notebook_task target
│           ├── train_job          (NOTEBOOK)
│           ├── reci_pick/                          ← importable: this dir is on sys.path
│           └── …
├── artifacts/                                      ← ${workspace.artifact_path}
├── resources/                                      ← ${workspace.resource_path}
└── state/                                          ← ${workspace.state_path}
```
Note that The job entrypoints (the train_job.py and predict_job.py notebooks) sit at the project root `...files/projects/reci-pick/`
Databricks puts the root on sys.path so my notebooks can just import from sub-folders like `from reci_pick.helpers import parse_bool` without having to tweak the os path.

Contrary to what you might expect, the bundle files do not start at `reci-pick/`, which is where our bundle definition `reci-pick/databricks.yml` live.
You see that we have `packages` and `projects` folders in the bundle. This is because we use `sync: paths:` to add internal packages to the bundle.
More explanation below.

This is the relevant part of the bundle declaration in `databricks.yml`. Code comments explain how the thing works.
```yaml
bundle:
  name: reci-pick


# The "include" block is for referencing additional config files within the bundle.
# I could define my jobs directly in `databricks.yml`, but splitting them across files is nice for organizing code.
include:
  - "jobs/**.yml"
  - "jobs/**.yaml"

# We need to include the files of the internal libraries in the bundle.
# This changes the folder structure of the bundle to the earliest parent of all the included paths
# so the directory `.bundle/reci-pick/dev/files/` is `sous-chef` instead of `sous-chef/projects/reci-pick`.
# Which is why we see the `projects` and `packages` folders in the bundle.
# This becomes easier if we have our own feed for internal packages
sync:
  paths:
    - .
    - ../../packages/constants
    - ../../packages/time-machine
    - ../../packages/catalog-connector

variables:
  # setup the environment here instead of having the same `environments` block in all our job files
  # This environment block is for serverless compute, the config for a classic cluster looks different.
  reci_pick_environment:
    description: The serverless environment shared by every reci-pick task.
    type: complex
    default:
      environment_key: reci-pick
      spec:
        # ML base env contains all the libraries we need for this project
        base_environment: workspace-base-environments/databricks_ml_v5
        # This is how we tell Databrikcs to install extra packages that aren't in the base env
        # This should be generated from the pyproject.toml (see dependency management section)
        # But nice to show the basic version
        dependencies:
          # internal libs are referenced by path
          - ${workspace.file_path}/packages/constants
          - ${workspace.file_path}/packages/time-machine
          - ${workspace.file_path}/packages/catalog-connector
          # This is how you install external libs that aren't in the base env.
          - pendulum==3.2.0
```


# Serveless compute limitations 
Serverless compute has 16GB memory maximum. Spark workers do auto-scale. But not the driver node, so if we're doing single-node python stuff, 16GB is a hard cap.

Previous code would have crashed with OOM error. But Claude easily optimized it to fit under 16GB mem. And also make the predict job 10x faster. Went from ~1h30-2h to 12min.

Didn't look at the code changes in detail, but they look pretty straightforward to me:
- Not reading columns we don't use.
- Not doing invariants in a loop.
- Using sets / vectorized operations instead of loops.
- Pushing some work to spark

The output is also very similar to previous training runs.

Lessons learned:
- optimize code before scaling up compute
- Pre-Claude code can probably be optimized a lot
- We can probably fit all of our workloads on serverless, by optimizing algorithms, chunking data etc.

If serverless isn't enough, go back to job cluster or persistent cluster. Few notes to improve on that:
- Use single node clusters
```yaml
job_clusters:
    - job_cluster_key: dbt_CLI
        new_cluster:
        cluster_name: ""
        spark_version: 15.4.x-scala2.12
        spark_conf:
            spark.master: local[*, 4]
            spark.databricks.cluster.profile: singleNode
```
- `num_workers: 1` doesn't create a single node cluster, it add a spark worker in addition to the driver node, not worth it if you're mostly doing single node work in python/pandas and not delegating anything to spark. If you're doing `spark.sql(...).toPandas()` directly, you probably don't need a Spark worker.
```yaml
new_cluster:
    num_workers: 1 # very counter-intuitive, but this creates means 2 machines
```
- Removing the extra worker and not pulling a docker image speeds up cluster start time a bit, but still takes >5min, not counting time to install libraries.
- Multiple tasks in a job instead of multiple jobs saves startup time.
- Provisioning persistent compute for the project as a whole means you don't need to recreate the cluster on every job run. Just have to cold-start it once at the beginning of a development session.
- It's possible to create custom base environments for a project to speed up install time when a cluster start. Easy to have a simple chef cli command of bundle script for that.

# Dependency Management

## Which libs do we currently use that aren't in the base env

The ML base environment has a LOT of stuff. Here's a list of all the dependencies of current ML batch jobs which aren't pre-installed in the base env:
```
┌──────────────────────────┬─────────────────────────────────────────────────────────────────────────────┬─────────────────────────────────────────────────────────────────────┐
│        dependency        │                            imported by job code                             │                declared but not imported in job code                │
├──────────────────────────┼─────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┤
│ pydantic-settings        │ attribute-scoring, customer-churn, data-catalog, menu-optimiser,            │ data-model, dishes-forecasting, fog, food, ml-example-project,      │
│                          │ orders-forecasting, preselector, recipe-annotator                           │ recipe-tagging, review-screener, toffee                             │
├──────────────────────────┼─────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┤
│ pandera                  │ attribute-scoring, dishes-forecasting, menu-optimiser, orders-forecasting,  │ —                                                                   │
│                          │ recipe-annotator, recipe-tagging                                            │                                                                     │
├──────────────────────────┼─────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┤
│ pydantic-argparse        │ fog, food, toffee                                                           │ —                                                                   │
├──────────────────────────┼─────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┤
│ pycaret                  │ customer-churn, orders-forecasting                                          │ dishes-forecasting                                                  │
├──────────────────────────┼─────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┤
│ bayesian-optimization    │ recipe-tagging                                                              │ —                                                                   │
├──────────────────────────┼─────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┤
│ catboost                 │ attribute-scoring                                                           │ —                                                                   │
├──────────────────────────┼─────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┤
│ databricks-feature-store │ ml-example-project                                                          │ —                                                                   │
├──────────────────────────┼─────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┤
│ datadog                  │ preselector                                                                 │ recipe-annotator                                                    │
├──────────────────────────┼─────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┤
│ fuzzywuzzy               │ recipe-tagging                                                              │ recipe-annotator                                                    │
├──────────────────────────┼─────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┤
│ openpyxl                 │ toffee                                                                      │ menu-optimiser                                                      │
├──────────────────────────┼─────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┤
│ polars                   │ preselector                                                                 │ food                                                                │
├──────────────────────────┼─────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┤
│ polars-lts-cpu           │ preselector                                                                 │ —                                                                   │
├──────────────────────────┼─────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┤
│ pulp                     │ menu-optimiser                                                              │ —                                                                   │
├──────────────────────────┼─────────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────┤
│ pydantic-ai-harness      │ preselector                                                                 │ —                                                                   │
└──────────────────────────┴─────────────────────────────────────────────────────────────────────────────┴─────────────────────────────────────────────────────────────────────┘
```

`pycaret` is the only problematic one, it's used in customer-churn, orders-forecasting and dishes-forecasting. There are no stable version compatible with `numpy 2.X`. But there's a pre-release version that is compatible. Hopefully it's stable if and when we decide to migrate those, otherwise we can just create an custom env.
Could also move out of `pycaret`, it's a wrapper around deps already installed in the base env.
If we don't want to replace of update it, just create another base env.

## Preventing deps conflicts

The ML projects should be generated from a template and always have a python version compatible with whichever version of the serverless env we're aligning on.
Use `poetry add` to add libraries to your project. Do not pin the libraries you're adding.

Instead of `poetry install`, run `pip install --constraint requirements-ml-5.txt -e` (we could alias this) (the requirements.txt file of the base env we're using would be part of the project template.) 

`pip install --constraint requirements-ml-5.txt -e` will install requirements from `pyproject.toml` while respecting the constraints specified by the base env. If your `pyproject.toml` demands MLflow or any packages pre-installed in the base env, the version specified by the base env's `requirements.txt` will be installed.

This saves you from accidentally upgrading/downgrading packages pre-installed in the serverless cluster. Or writing your code in a version of a package that's incompatible with the base env.

It also fails loudly and clearly instead of having to wait for a databricks job to start before getting a weird error.

```
ERROR: Cannot install demo==0.1.0 because these package versions have conflicting dependencies.
The conflict is caused by:
    demo 0.1.0 depends on xgboost<3.0.0 and >=2.1.1
    The user requested (constraint) xgboost==3.1.1
To fix this you could try to:
1. loosen the range of package versions you've specified
2. remove package versions to allow pip to attempt to solve the dependency conflict
ERROR: ResolutionImpossible
```

I'm pretty sure that aligning on a specific version of a base env and installing while passing a constraint file solve 99% of our dependency management issue. It's also nice to go from 100 library versions to 1 environment version. Plus, the Databricks people have already solved for compatible versions of 400 commonly used ML libraries.

## Generating env specs from poetry

I think it's nice to keep poetry as the main way we manage deps, it's annoying if we have to separately maintain configs for the local and remote env. We should probably generate the specs for the environment automatically from the pyproject.toml file.
I didn't write it, but it sounds like a simple script that would fit nicely as a chef cli command.

Could have the bundle look something like this:
```yaml
bundle:
  name: reci-pick

include:
  - "jobs/**.yml"
  - "${bundle.name}_env.yml"
```

A command like `chef env sync` would automatically creates `${bundle.name}_env.yml` from `pyproject.toml`.


## What if my deps aren't compatible with the base env?

First instinct should be to try and use something else because having a different env for every project is kind of a hassle.
But if really needed, it's easy to specify and create a custom environment:
```yaml
# projects/customer-churn/serverless_env.yaml
environment_version: '2'
dependencies:
  - pycaret==3.3.2
  - pydantic-settings>=2.1.0
```
Dependencies accepts anything valid in a requirements file: pins, --index-url, -r /Workspace/…/requirements.txt, wheels on Volumes, git+https://...

Reference the environment in your job:
```yaml
# Syntax for serverless compute
environments:
  - environment_key: customer-churn
    spec:
      base_environment: workspace-base-environments/customer-churn-pycaret
      dependencies:
        - ${workspace.file_path}/packages/constants
```

A simple API call lets you turn it into a registered environment, so it gets cached and you get to skip install time.
```bash
databricks environments create-workspace-base-environment "Customer Churn (pycaret)" \
  --workspace-base-environment-id customer-churn-pycaret \
  --base-environment-type CPU \
  --filepath /Workspace/Shared/base-envs/customer_churn_env.yaml
```
This can easily be turned into a CLI or bundle command.

## OS-level deps

If your project needs OS-level dependencies which aren't installed on serverless compute, you can use classic compute with an init script running `apt get`.
I would be very surprised if that was ever necessary. The ML base environments is very complete.
Our only OS-level deps that isn't in the base env is `msodbcsql18` which the preselector uses to access the replica directly.
This dependency is not needed. We're already using lakehouse federation to connect to the replica through databricks. So the replica DBs are accessible through Unity Catalog just like any Databricks table (`pim_live`, `operations_live`, `cms_live`.) We can easily add connections to any Azure SQL DBs. This also simplify auth and managing connection & access to the replica.

# Local Dev
- In production code, use `catalog-connector` to get data. It handles getting the ambient spark session on remote or creating one with `databricks-connect` if on local. Generally good to use internal packages which work on both local and remote.
```python
def get_spark_session() -> SparkSession:
    from catalog_connector import connection
    return connection.spark()
```

## Without Docker
- The [Databricks VSCode extension](https://docs.databricks.com/aws/en/dev-tools/vscode-ext/#convert-project) is nice for exploratory work and local dev. You do need to declare the host in your bundle to use it though, which is not something we do by default.
```yaml
targets:
  dev:
    default: true
    mode: "${var.mode}"
    workspace:
      host: https://adb-4291784437205825.5.azuredatabricks.net
```
- `databricks-connect` lets you read from Databricks and use some `dbtutils`. Nice for exploratory work if you want to download data locally.
- `databricks sync ...` ship code to Databricks without redeploying the whole bundle 


## With Docker (devcontainer)

Databricks publishes Docker image for the Databricks Runtime. So it's easy to setup a dev container for local dev in Docker.
Building the image the first time took me 12 minutes because it's pulling a 2GB image for the Databricks runtime then installing all 400-ish in the ML Base env (optional.) But you only need to do it once, after which it's cached and starts in seconds.
You'd need to download a new image if we move to another version of the Databricks serverless environment but that's a once-a-year type of thing.

We'd need multiple image currently, because a lot of our projects use different python version. But I'd rather align everything to a specific version of the Databricks serverless env rather than individually manage the runtime/python/libs versions of every project.

The setup of the dev container can very easily be automatted with cookie-cutter and the chef CLI. Using the base image provided by Databricks is an improvement over the current setup when it comes to environment parity. 

To properly replicate the remote environment, we need to emulate an `amd64` architecture, which will be slower on Apple silicon. This limitation is already present in the current docker setup.

Note that the dev container won't give you features which are exlusive to Databricks notebooks, no ambient spark, `dbutils.widgets`, `display()`, etc. But that's also true of the current setup. If you prefer Databricks notebooks over python files, the IDE extension is probably a better option. If we want to fully standardize on dev containers, the cookie cutter template can ship with examples for creating jobs etc without the notebook-exclusive features.

`reci-pick/.devcontainer/devcontainer.json`
```json
{
  "name": "reci-pick (serverless env v5)",
  "build": { "dockerfile": "Dockerfile" },

  // Mount project + repo root for internal packages
  "workspaceMount": "source=${localWorkspaceFolder}/../..,target=/workspaces/sous-chef,type=bind",
  "workspaceFolder": "/workspaces/sous-chef/projects/reci-pick",
  // Databricks connection settings
  "remoteEnv": {
    "PATH": "/databricks/python3/bin:${containerEnv:PATH}",
    "DATABRICKS_HOST": "${localEnv:DATABRICKS_HOST}",
    "DATABRICKS_TOKEN": "${localEnv:DATABRICKS_TOKEN}"
  },
  // Mount .databrickscfg for auth, etc
  "mounts": [
    "source=${localEnv:HOME}/.databrickscfg,target=/root/.databrickscfg,type=bind,readonly"
  ],

  "customizations": {
    "vscode": {
      "extensions": ["databricks.databricks", "ms-python.python", "charliermarsh.ruff"],
      "settings": {
        "python.defaultInterpreterPath": "/databricks/python3/bin/python",
        "python.testing.pytestEnabled": true,
        // Match the serverless environment version
        "databricks.connect.serverlessDbconnectVersion": "18.0"
      }
    }
  },

  // Install requirements from pyproject.toml while respecting the constraints of the ML Base env or the Standard base env
  // Could pre-install everything in whichever Base env we pick so everything is cached and we save some install time.
   "postCreateCommand": "/databricks/python3/bin/pip install --no-cache-dir --constraint /opt/serverless-env/requirements-ml-5.txt -e"
}

```

`reci-pick/.devcontainer/Dockerfile`
```
FROM --platform=linux/amd64 databricksruntime/environment:v5-standard

COPY requirements-ml-5.txt /opt/serverless-env/requirements-ml-5.txt
RUN /databricks/python3/bin/pip install --no-cache-dir \
      --extra-index-url https://download.pytorch.org/whl/cpu \
      # optional: Install packages in the ML base environment
      --requirement /opt/serverless-env/requirements-ml-5.txt
```

`reci-pick/.devcontainer/internal-packages.txt`
```
-e ../../packages/constants
-e ../../packages/time-machine
-e ../../packages/catalog-connector
```

`reci-pick/.devcontainer/requirements-ml-5.txt`
(downloaded from Databricks docs, but should be committed to the repo)
```
absl-py==2.3.1
accelerate==1.11.0
aiohappyeyeballs==2.4.4
aiohttp==3.11.10
...
```