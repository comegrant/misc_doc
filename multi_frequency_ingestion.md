
# Ingestion changes

Several of this cook's objectives require changes to our ingestion setup. I think those can be solved in the same redesign. 

1. **Multi-frequency support:**
We currently lack a systematic way to make different data sources available at different frequency.

2. **CDC Ingestion support:**
We need an ingestion strategy that reads from change data capture. CDC is more efficient and reliable. Efficiency is key to increasing ingestion frequency without too much cost or load on the data sources. Reliability is critical in cases where historical data is deleted from the source and missing data in the ingestion could cause permanent data loss. 

3. **Decoupling of data sources, ingestion strategies and jobs:**
Currently the ingestion logic is tied to each data source. We have different load functions for each server we connect to (AnalyticsDB, CoreDB, AnalyticsDB staging). The ingestion functions are also tied to the jobs by being called directly in the ingestion notebooks. As we add more datasources (CMSQA...), ingestion strategies (CDC ingestion...) and jobs (for different ingestion frequencies), we need to decouple things to keep the codebase DRY and easy to maintain.


## Not in Scope

1. **Stream processing:**
Developping ways to process data in real-time would require significant changes to our platform, including deploying new tools etc. 

