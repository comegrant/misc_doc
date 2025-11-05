
# Requirements
Several of this cook's objectives require changes to our ingestion setup. I think those can be solved in the same redesign. 

1. **Multi-frequency support:**
We currently lack a systematic way to make different data sources available at different frequency.

2. **CDC Ingestion support:**
We need an ingestion strategy that reads from change data capture. CDC is more efficient and reliable. Efficiency is key to increasing ingestion frequency without too much cost or load on the data sources. Reliability is critical in cases where historical data is deleted from the source and missing data in the ingestion could cause permanent data loss. 

3. **Decoupling of data sources, ingestion strategies and jobs:**
Currently the ingestion logic is tied to each data source. We have different load functions for each server we connect to (AnalyticsDB, CoreDB, AnalyticsDB staging). The ingestion functions are also tied to the jobs by being called directly in the ingestion notebooks. As we add more datasources (CMSQA...), ingestion strategies (CDC ingestion...) and jobs (for different ingestion frequencies), we need to decouple things to keep the codebase DRY and easy to maintain.


# Not in Scope
1. **Stream processing:**
Real-time data processing (streaming) is very different from the batch processing that we are used to. While it may be worth looking into it in the future, I would suggest that we keep it out of scope for now and focus on high-frequency batching. The maximum frequency we can reach is still TBD, I would say that 2 minutes is ambitious, 5 minutes is doable and 10 minutes is fairly easy. But how much work it will take to reach higher frequencies will vary wildly dependending on which dbt model we are talking about.

2. **Ad-hocs:**
We want to keep a solution for ad-hocs reports fetching data straight from the replica. Whether we do it in Metabase or we publish a live connection to the Replica in PowerBI. The process of ingesting, processing and refreshing data at high-frequency requires a significant amount of work and we should only do it when there is significant upside. Meaning cases where we need to do significant processing or combine with other data available only in NDP.


## Proposed changes to ingestion:
Let's start by looking at changes to the ingestion this is the natural first step and the design is more straightforward than the dbt or PowerBI parts IMO.


### Server Config
The goal here is to abstract the connection setting.

1. Create an abstract base class (ABC) defining a "Server" as something that provides a format and options to pass to spark.read()
```python
class Server(ABC):
    format: str

    @abstractmethod
    def get_options(self) -> Dict[str, str]:
        pass
```

2. Create a SQLServer class as a type of Server.
```python
@dataclass
class SQLServer(Server):
    host: str
    port: int
    format: Literal["sqlserver", "postgresql"]
    username: str
    password: str

    def get_options(self):
        return asdict(self)
```

3. Create concrete instances of SQLServer for different datasources.
```python
coredb_server = Server(
    host="bh-replica.database.windows.net",
    format="sqlserver",
    port=1433,
    username=dbutils.secrets.get(scope="auth_common", key="coreDb-replica-username"),
    password=dbutils.secrets.get(scope="auth_common", key="coreDb-replica-password")
)
```

This is a little bit more complicated but it follows SOLID principle. We can now read data from any source that's compatible with spark using whatever connection format and auth method we want without ever having to change the ingestion functions, jobs etc. All we need to do is create an object that has the required options for spark.read().


### Database Readers
1. Create a DatabaseReader class implementing a query() method that runs some SQL on the provided Server & database
```python
@dataclass
class DatabaseReader():
    server: Server
    database: str

    def query(self, query: str) -> DataFrame:
        spark = SparkSession.builder.getOrCreate()
        result = (spark.read
            .format(self.server.format)
            .options(**self.server.get_options())
            .option("database", self.database)
            .option("query", query)
            .load())
        return result
```
There could be an ABC for DatabaseReader as well, but I think it's very unlikely to come up so let's keep in simple.
It's called DatabaseReader but it can also read from csv, json or whatever source due to the Server abstraction.

2. Create a CDCReader class inheriting from DatabaseReader

```python
@dataclass
class CDCReader(DatabaseReader):
    def get_new_data(...
```
This class is tasked with implementing functions required for reading from CDC. I wouldn't put those functions in DatabaseReader because we might want to implement full ingest but not CDC ingest on a particular datasource.

This could also be an abstract base class. And it's more likely that we will need one. Since the syntax for the cdc functions will be different between different databases, we will need different CDCReaders if we want to support CDC read for postgres in addition to Azure SQL for example. But if that ever comes up, turning CDCReader into an ABC and making different implementations for different databases will be an easy refactor that doesn't require any change in the dependencies. So let's keep it simple for now.

3. (Optional) Create an IncrementalReader inheriting from DatabaseReader

```python
@dataclass
class IncrementalReader(DatabaseReader):
    def get_new_data(...
```
We can implement a single strategy for timestamp-based incremental reads just like we do for CDC. But we can also just keep writing the queries directly.


### Ingestion Strategies
Now that we've created abstractions that handle connecting to a data source and running sql queries on it, we can have the same functions performing our different ingest strategies on any datasources by passing them our reader classes.
These functions are responsible for:
- Getting data from the source by using methods provided by the reader class
- Writing to the sink table
- Querying required data from the sink table (e.g: the latest timestamp in the sink table for an incremental read)
- Error handling, logging etc.
- Setting table properties (retention period etc.)

```python
def query_ingest(
    database_reader: DatabaseReader,
    query: str, 
    source_table: str,
    source_schema: str = "dbo",
    mode: Literal["append", "overwrite"] = "overwrite",
    sink_table: Optional[str] = None,
    sink_schema: str = "bronze",
) -> None:
    ...


def full_ingest(database_reader: DatabaseReader, ...


def cdc_ingest(cdc_reader: CDCReader, ...


def incremental_ingest(incremental_reader: IncrementalReader, ...

def selected_columns_ingest(database_reader: DatabaseReader, ...
```


### Job config
I think the most scalable way is to switch to a .yml config for listing which tables to ingest. I think there are several major benefits to this:
1. It's easier to organize. Having one notebook per server per database per ingestion strategy per ingestion frequency will get very messy.
2. It separates the code and the configuration. If our full_ingest() function is called in 25 notebooks, it will be very painful to change.
3. It very similar to the way that we work in dbt and should hopefully be easy for people to work with.

I imagine the folder structure could be something like ingestion_strategy/server/database.yml
But anything is possible, it doesn't really matter from a technical PoV.
```
full_ingest/
├── analytics_db/
│   ├── cms_qa/
│   ├── core_db/
│   │   ├── cms
│   │   └── finance
│   ├── postgres/
├── cdc_ingest/
├── incremental_ingest/
├── query_ingest/
└── selected_column_ingest/
```

The .yml files contain info about the server & database to connect to as well as any arguments, required or optional, that we want to pass to the relevant ingestion function.
e.g: full_ingest/core_db/cms.yml

```yaml
server: coredb
database: CMS
schemas:
 - name: dbo
   tables:
    - name: address_live # required
      frequency: daily # Could be required or optional argument that defaults to "daily"
      sink_table: cms__addresses # Optional argument for manually picking the name of the sink table

    - name: consents
      frequency: hourly

    - name: consents
      frequency: hourly
```
Can possibly have validation or linting on the yaml configs.


### One Notebook
A single notebook is responsible for:
- Parsing the .yml configs
- Filtering the .yml configs based on parameters
- Picking the appropriate ingestion function to use
- Getting the server & database info from the config
- Creating the DatabaseReader, CDCReader etc. based on the ingestion strategy, server and database
- Looping over tables in the config and calling the ingestion function passing the reader class and any arguments defined in the config

This notebook would have parameters for `server`, `strategy`, `database`, `frequency` & `tables` that it uses to filter the config (defaults to running everything).
These parameters are provided by the different jobs calling this notebook and can be overwritten with widgets for manual runs.


### Several Jobs
Several jobs call the ingest notebook. We need (at leat) one job per frequency. Each of those jobs will have its own CRON schedule and pass a "frequency" parameter that the notebook uses to filter the .yml configs.
It's possible to have different jobs for different servers or types of ingestion. But I think this is simpler.

```yaml
 timeout_seconds: 3600
 edit_mode: EDITABLE

 schedule:
  quartz_cron_expression: 0 * * * * # every hour
  timezone_id: Europe/Oslo

 tasks:
  - task_key: ingest_hourly
    notebook_task:
     notebook_path: ../ingest.py
     base_parameters:
      - frequency: "hourly"
```


### Notes
I really like this setup. It mostly follows SOLID principles (even if we skipped some ABCs for the sake of simplicity) and I think it will scale well. Each component is nicely decoupled which will avoid complex refactors. It's easy to add new datasources or ingestion strategies without changing other components. The separation of code & config makes it easy to ingest more data without increasing the complexity of the code base. And calling the same parameterized notebook from different jobs makes it easy to add different schedules or split the ingestion job into different parts if we ever want to do so.

Some areas of improvements that aren't discussed here:
- We could have watermarks tables for storing last ingested LSNs and timestamps for each models. But getting the maximum value for a column in databricks should be a really fast metadata operation. So just querying the max value from a table should be fine.
- How do we do backfills?
- How do we do the initial load of a table before turning on cdc ingest?
- Would be good to tag queries generated by the ingestion functions so we can find them easily in the query history. But I'm not sure how to do that. The query history table has a "query_tag" column but no results on the internet on how to set query tags. Might be that the feature is still in development.
- Maybe have validation / linting of the yaml configs in the CI/CD.
- This doesn't cover partitioning, Z-ordering or any sort of optimization to the bronze table, but it might be worth looking into if we want to improve performance.



## Proposed changes to dbt:

### Tag-based scheduling
The setup is similar to how we handle frequency in the ingestion. We add a frequency tag in the models.yml.
The name of the tags should be kept consistent between dbt & the ingestion.
Note that we do not need to tag every models, untagged models will be run daily, as that's our default speed.

```yaml
models:
  - name: dim_companies
    +tags: "hourly"
    description: ""
    latest_version: 1
    config:
      alias: dim_companies
      contract:
        enforced: true
```

We can then have one dbt job per frequency that runs dbt selecting the appropriate tag.

```yaml
- task_key: dbt-run-hourly
    dbt_task:
    project_directory: ../transform/
    warehouse_id: ${var.warehouse_id}
    catalog: ${bundle.target}
    commands:
        - "dbt deps"
        - "dbt seed"
        - "dbt snapshot"
        - "dbt run --exclude drafts.*" --select tag:hourly
```


### Multi-frequency orchestration
The orchestration of the different frequency job is more complicated with dbt than with the ingestion, because this time, our different frequency jobs might have dependencies on each other. There are many different ways to handle orchestration of jobs with different frequencies. You can have sensors, one job triggering the next one, event-based orchestration or just run jobs at an offset from one another. 

My favorite way to handle this is to sidestep the problem altogether by having **every job run dbt models with the same or higher frequency**. For example, if we have an hourly and a daily job, the hourly job would run hourly models every hour and the daily job would then run the daily & hourly models. This means that the hourly job would actually run 25 times per day.

This not the most efficient solution in terms of compute. But it makes life way easier. The waste caused by unnecessarily running models should be compensated by the fact that any model that we run on a high frequency should be either a strictly idempotent incremental model (meaning the extra run doesn't process any data) or fast enough that the wasted compute is negligible. 


### Making dbt run faster
If we want to run a dbt model on a high frequency, performance becomes much more important. Different strategies are possible to achieve low-frequency in dbt. We can use incremental models, microbatch incremental models, materialized views, views etc.
I think our go-to options should be incremental models, as they are the most straightforward and flexible. Although microbatch incremental models could be worth looking into for time series data (mostly fact tables and their upstream dependencies).

The biggest advantage of microbatch incremental models is that you can run several batches in parrallel. So if we wanted to run a fact table every 5 minutes, we could process 5 1-minute batches concurrently. But I have never used it in the past so I can't say whether it will work for us. 

In any case, we will need to update our documentation and conventions to go more in-depth about how we work with incremental models.

The hard part about making a dbt model run at a high frequency will be figuring out which of its upstream models need to run at the same frequency and updating those as well. This can be difficult when it comes to fact tables which are usually at the very end of the lineage with a lot of upstream models.

Two ways we could solve this if it ever comes up:
1. Process the data twice at different frequency. If we need a report built on top of a fact table refreshed every 5 minutes but getting the entire fact table to run that fast is difficult, we can always create a second high-frequency model that is a subset of that fact table with less join to dimensions etc.
2. Delta Views. This involves two components: A view that contains the transformation logic and an historical table. The historical table runs on a daily basis as an incremental model that uses the view as a source. You can then serve the end-user a view that unions of the historical table and the view filtered on data for the current day. I do not love this option, but it is an option.



## Proposed changes to PowerBI:

Very much the same thinking as for dbt & the ingestion. We need incremental refresh policies on tables which are large or run frequently. And we need several jobs refreshing different Power BI tables at different frequencies. We already have code for adding incremental refresh policies to Power BI tables and refreshing selected tables. So this all should be manageable.

If this is too complicated or too slow. We can always publish new, smaller, direct query semantic models for the data that needs to be refreshed often. The downside of this approach is that more semantic models means more dependencies and we might have to update several models for one dbt change.