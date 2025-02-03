# README: Treasure Data Workflow for Sampling Production Data to Development

## Overview
This Treasure Data (TD) workflow queries data from the production TD environment, samples it, and then pushes the sampled data to the development environment. The workflow ensures efficient data handling while maintaining data recency and sufficent data for relationships between tables.

## Workflow Process
1. **Create Crossref Samples**: Applies sampling logic on crossref table to get sample of `honors_number`, `guest_service_id`, `stay_id` or `memberid`.
2. **Extract Data**: Query data from the production TD database (`stage`).
2. **Obfuscate Data**: Hash PII data based on configs of `sample_data.config_pii_columns` table.
3. **Filter Data**: Apply a Filtering logic based on `sample_percentage` and `recency_days` in `base.yml`.
4. **Sample Data**: joins data with crossref samples to get the final sample of data.
5. **Push Data to Dev**: Creates table an inserts sampled data in `stage` database of dev environment.

## Configuration Files
### `config/base.yml`
Defines global settings for the workflow:
```yaml
td:
  database: stage  # Production database
prod_sample_database: sample_data  # Target database in dev to push sampled data
sample_percentage: 10  # Percentage of data to sample on all tables except full_load ones.
recency_days: '60'  # Time window for data selection
```

### `config/table_config.yml`
Lists tables to be processed along with their primary keys and sampling reference tables:

1. **database:** source database containing table which needs to be loaded to dev.
2. **table:** name of the table which needs to be sampled and loaded to dev.
3. **primary_column:** name of the column in the **table** which will be used by workflow to join with mentioned corssref table for sampling.
4. **crossref_sample_table:** name of the crossref table which should be used to sample the mentioned **table**. refer below crossref tables in the order of priority:
   
    a.  **guest_service_id_sample:** check if the source table have either `guest_service_id` or `honors_number` column then use `guest_service_id_sample` and note that use `honors_number` only when `guest_service_id` is unavailable and in that case also mention `crossref_column: honors_number`.

    b.  **stay_id_sample:** use when source table contians `stay_id` column and none of the above id columns are available.

    c.  **memberid_sample:** use this for thirdnode tables when `memberid` column is available.

    d.  **full_load: true** use when none of the above id columns are available and table needs to be fully loaded. 
    
#### Example: 


```yaml
tables:
  - database: stage
    table: d_customer_email_marketablity
    primary_column: honors_number
    crossref_sample_table: guest_service_id_sample
    crossref_column: honors_number

  - database: stage
    table: commission
    primary_column: stay_id
    crossref_sample_table: stay_id_sample

  - database: stage
    table: d_customer_email_base
    primary_column: guest_service_id
    crossref_sample_table: guest_service_id_sample

  - database: stage
    table: checkout_folio
    primary_column: stay_id
    crossref_sample_table: stay_id_sample

  - database: stage
    table: thirdnode_segmentmember_base
    primary_column: memberid
    crossref_sample_table: memberid_sample
  
  - database: stage
    table: conf_email_delivery
    full_load: true
```

## Execution Steps
1. **Configure table_config.yml**: table_config.yml needs to be configured for each new table that needs to be sampled and loaded to dev environment.
2. **Configure base.yml**: set gobal peremeters like target database sampling size and data recency in in base.yml config file.
3. **Configure config_pii_columns table**: config_pii_columns table must be configured for each column with pii data which needs to be obfusticated before loading data to dev environment. 
4. **Run Workflow**: Verify that the data has been successfully sampled and pushed to `sample_data`.

## Notes
- Ensure that the sample percentage is set appropriately to avoid excessive data extraction.
- Adjust `recency_days` as needed to filter data based on business requirements.
- Add mappings in `table_config.yml` for adding new tables.

## Troubleshooting
- **Job Failure**: Check workflow logs in TD for errors.
- **Data Mismatch**: Verify that primary keys and sampling logic are correctly applied.
- **Performance Issues**: Use small sample if execution times are long.
