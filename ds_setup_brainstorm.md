**TODO** Elaborate
General principles:
- safe defaults
- always have an escape hatch
- collaborate on finding the golden path
- ownership doesn't mean "not my problem" or that others can't contribute. More of a rule of general principle. We're at a size where red tape or strictly defined modes of interaction would cause more harm than good.


# Definition of components and division of ownership

## Static Infrastructure

Covers resources which are static, generic and long-lived. Necessary plumbing that Data Scientists can ignore.
This is unambiguously owned by Data Platform with Data Scientists as a key stakeholder.

Includes:
- Networking
- IAM
- Secret Management
- Resource Groups
- Storage Accounts
- Container Registry
- Container App Environment
- Package feed
- Logging & Alerting infra


## Dynamic Infrastructure

Resources requiring per-project configuration. Or resources that may change based on activity by Data Scientists (for example, creating a new Container App for a project.)
Co-owned by Data Science and Data Platform. Databricks resources are configured through Databricks Asset Bundles and Azure resources are configured through a similar interface owned and provided by Data Platform.

Includes:
- Container Apps (name, entry points, available compute)
- App Privileges (access to Key Vaults secrets, Databricks Permissions)
- What to log
- When to trigger alerts 
- Custom Databricks compute (job compute, classic compute)
- API endpoints
- Service bus topics
- Redis / Lakebase instances
- Other compute (serverless functions, container instances)

The use of an interface (likely a yaml config read and applied by Terraform) to provision Azure resources aims to simplify the management of infrastructure for Data Scientists while making sure that infra is accounted for, version-controlled and automatically-managed.  
Said interface should be extended by the Platform team when a Data Science project requires new, not yet supported, infrastructure. This  guarantees that new infrastructure is reliable, well-configured and replicable. At the cost of some dependency from Data Science towards Data Platform.  

The co-ownership implies that both Data Platform and Data Science should monitor and intervene on failures of dynamic infrastructure. Data Platform is responsible for providing an intuitive, thorough and well-documented interface but may not entirely prevent issues stemming from configuration or app code (for example, an out-of-memory error.) 

Data Scientists are welcome to contribute to the Terraform project managing Data Science infra.


## ML Pipeline

The end-to-end workflow of developping, maintaining and serving ML models.

Includes:
- Reading data from various sources
- Processing data
- Serving data asynchronously (writing to persistent storage, message brokers etc.)
- Serving data synchronously (API endpoints, real-time inference, writing to cache)
- Training, storing, promoting and deprecating ML models (in Databricks MLflow)
- Monitoring (logging, alerting, drift detection)

Data Scientists own implementation of the above for any specific project. Generic components required for performing any of these tasks fall under static or dynamic infra. Data Platform and Data Science collaborate on finding a "golden path" for different purposes. Data Scientists may independently manage components isolated from rest of the Data Infrastructure (MLflow?) if desired. 


## App Code & Conventions

All the logic in Data Science projects is owned by Data Scientists. This includes code processing, reading or serving data, orchestration of batch jobs, when to throw an alert, how to evaluate model performance etc.  
The elicitation and enforcement of Data Science conventions is owned by Data Scientists. Data Platform is a stakeholder in conventions that may impact the configuration or maintainability of related infrastructure.  
Application code in the Data Science project template is owned by Data Science.


## Continuous Integration

Data Platform is responsible for the setup of generic CI. Such as publishing preview apps on PRs, testing that docker images correctly build and start.
Data Scientists are responsible for the testing and QA of app code. Data Platform will provide the generic github workflow running the test suite / linting etc. Generic workflows may be overruled by Data Scientists.


## Continuous Deployment

Data Platform is responsible for the deploying changes in app code and infra configuration made by Data Scientists. This involves deploying DABs, creating Azure resources configured by Data Scientists, building and publishing docker images, updating images used by container apps etc.  
Data Science infra is managed by an independent Terraform deployment to minimize dependency on Data Platform. 


## Common Packages and Tooling

**TODO**
Why / when do we actually need custom packages?
Makes sense to have some quality of life commands in chef for the dev X. But feels like a good idea to avoid unnecessary wrappers. Might be good to have a critical look at which internal packages are actually needed.


# Current issues

## Lack of version control

Part of the Data Science infrastructure is manually configured in the Azure web console or through the az cli. Logging and alerting in Datadog is exclusively manual. 

Manually configured infrastructure is not on Github. We lack a thorough, as-code, description of what's out there, which makes onboarding and debugging difficult. Changes can not be easily reverted. Changes to infrastructure do not go through the PR process. It is difficult to see a history of changes when investigating an issue.
Manually managed infra also leads to a lot of hidden dependencies. When making code changes, it is very easy to forget to make the corresponding manual changes in Datadog or Azure.


## Multiple configurations

Part of the infrastructure is configured manually and can not be automatically updated when other infra changes require doing so. For example, the App Registration used by fog-editor to handle login hard-codes the redirect URL. Changes to the Container App Environment which require a re-creation change the URL and locks people out of fog-editor until the App Registration is manually updated.

Other parts of the infra are managed as-code with Python and not integrated with Terraform. For example, secrets and environment variables in container apps are fetched from the key vault by chef deploy. A scheduled token rotation in Terraform requires manually triggering the deployment of every project using said token. Configuration changes (such as a secret rotation) may not reach running replicas and cause them to crash.

The language barrier between infrastructure managed in Python and Terraform generally makes things more fragile and harder to automate. Chef deploy hard-codes the name of resources provisionned managed by Terraform and vice-versa. Multiple, loosely integrated configurations make it difficult to estimate the blast radius and assert the safety of changes done in Terraform, Python or by hand. Making Data Science infrastructure more fragile and harder to change.


## Coupling of deployment and infrastructure

Chef deploy handles both deploying code changes and provisioning a lot of the infrastructure surrounding container apps (networking, container app environment.) A code change that is completely safe can cause an infrastructure failure if it happens to run into a transient error or drift.

The same chef deploy script runs for both QA and prod and manage networking resources shared by both environments. So changes to chef deploy ran in QA can break production. It is unlikely for a project's QA deploy to break Prod infrastructure, but that is due to the specific resources shared by QA and Prod and their lifecycle rather than proper environment separation.


## Lack of state management

Unlike Terraform, chef deploy is completely stateless. The entire infrastructure is deployed on every run, even when nothing has changed, which increases the likelyhood of encountering transient errors. Container job changes always use a destroy + create strategy which can cause outages if the create step fail. 

Lack of state also means that deleting or renaming an app in code does not destroy the associated infrastructure. Orphaned resources stay live in Azure and cost money until manually deleted (which is not a safe process due to the lack of version control.)

Most importantly, no state means that chef deploy has no planning step. It is impossible to know which changes will be made to the infrastructure before actually deploying them. Some errors that would have raised in a `terraform plan` are encountered during the deploy and may take down QA or Prod. Chef deploy also does not give us visibility on whether we are overwriting manual changes. 


## Only in-place changes are supported

With the exception of Container Instances, which are always deployed using a destroy + create strategy, resources managed by chef deploy are only ever updated or created but never destroyed. So any configuration changes that can not be done in-place require manually destroying the existing infrastructure and rerunning the deploy to recreate it. This means downtime and the opportunity for a lot of things to go wrong. A lot of changes that would be trivial in terraform are prohibitively risky and difficult using chef deploy.



## Outdated & broken dependencies

Many projects have dependencies which are several years old with multiple critical vulnerabilities (although unlikely to be exploitable in practice.) Some projects have a broken dependency set which can not be resolved (for example, analytics-api depends on preselector which declares another version of fastAPI.)

This would be a blocker for migrating existing projects to Databricks serverless compute.

Automating patch updates would be a good idea.


## Lack of testing

Many projects have little to no tests. Changes to DS projects generally require manual QA. No drift detection...
Lots of opportunities to improve on the MLOps front.


## Other issues
- Networking resources (container app environments, subnets, private endpoints) are duplicated per container app. Not dramatic, but wasteful.
- Some resources (preselector api) are accessible on the open internet with no auth or have suboptimal network security.
- Widespread use of the bundle SP PATs isn't ideal from a security PoV. Apps generally run with unnecessarily wide permissions.
- Usage of admin keys for storage account access.


# Ideas, fixes, etc


## Migrate to Terraform

Most infrastructure problems listed above are fixed simply by moving that infra to Terraform. The rest of them are much easier to fix once the infra has been moved to Terraform. 



**TODO**
move stuff to terraform, but what and in which order.