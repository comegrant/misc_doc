# Incremental Ingest of Basket Deviation:

## Problem:
1. We are not keeping the full history in bronze. So history must be kept in the replica. 
2. Keeping the history in the replica leads to a large table that slows down the ingestion job. 
3. History is kept in the incremental silver layer model, but we need to be able to full refresh the silver model, so we need all the history in the bronze layer.


## Proposed solution
1. Change the ingestion job to not overwrite history. Fetch records based on `updated_at` so we capture updated rows.
2. Deduplicate in silver with merge incremental strategy + custom deduplication logic for full-refresh.
3. Keep some history in the source DB to avoid data loss in case the ingestion job fails. How much data to keep is TBD.

This is my favorite option, as it allows full replayability of changes compared to, for example, writing the merge logic directly in the ingest folder. It also keeps the merging logic in the silver table in dbt and minimizes the amount of logic in the ingestion job. 


## Limitations
This solution brings the possibility of permanent data loss if the ingestion jobs fails repeatedly for long enough that the history is deleted from the source database before it's been ingested in DataBricks. This is more of less likely to happen depending on how much history is kept in the source.

The best solution would be to read from the CDC on prod. This would minimize the risk of data loss and should be more compute-intensive overall. 