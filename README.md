## ETL Data pipeline and Data Warehouse design

Data Pipeline Image

![](/images/new_pipeline.png)

## Assumptions:
1. Assuming the data will arrive timestamped to our file server.
2. Assuming we're using Azure File share(File server), Azure Virtual Compute(Virtual Machine), Azure Blob Storage(Object Storage and Azure Posgresql Managed DB(Relational database).

#### Data Warehouse Design
Data warehouse ER diagram Image

![](/images/wh_design.png)
**NB:** the relationship cardinalities are not accurate.

This is the important database for analytical purposes in order to understand the business, thus we must proven design. To design the data warehouse we'll use Kimball's technique of dimensional modelling i.e. fact and dimension tables, and loading the warehouse tables with the lowest level of detail(granularity) data. The fact table will contain a measure of a business process in this case a transaction, and a dimension table would contain desciptive values for table attributes e.g. `merchant_name`(we'll use snakecase as our naming convention in the warehouse). The database objects we'll create on our warehouse are tables, views and `public` and `staging` schema.


1. POS Devices csv file contains slow moving dimensional data i.e. `store` and `merchant_name` we'll replace that with a foreign/surrogate key.
2. We'll create a dimension table called `dim_merchant_name` on `public` schema to contain those fields.
3. Our data warehouse will have ttwo schemas `public` and `staging`, `public` will store permanent or target tables, and `staging` will store temp tables that wil be dropped off during the ETL process. These will be used to perfom the Load in the ETL process.
4. We'll create empty staging tables e.g. `staging.staging_pos_transactions`, `staging.staging_pos_devices`, `staging.staging_account_balances` and `staging.staging_merchant_name`, these will be placeholders. These will be used to load .csv files from object storage to staging schema before load to target table.
5. We'll create fact tables i.e. `public.fact_pos_transactions`, `public.fact_pos_devices` and `public.fact_account_balances`.
6. Creating unique constraints for facts and dimensions each tables.
7. Create views for requested metrics, i.e. average transaction value, average transaction frequency the tables will also give analysts flexibility for them to write their own queries.

#### DDL for above required db objects
Staging
```
create table staging.staging_merchant_name (
    id int primary key,
    store_name varchar,
    merchant_name varchar
);
```
Public
```
create table public.dim_merchant_name (
    id int primary key,
    store_name varchar,
    merchant_name varchar
);
```
Staging
```
create table staging.staging_pos_transactions (
    transaction_id int primary key,
    transaction_date timestamp,
    credit_card_number int,
    transaction_amount float,
    pos_identifier uuid_type
);
```
Public
```
create table public.fact_pos_transactions (
    transaction_id int primary key,
    transaction_date timestamp,
    credit_card_number int,
    transaction_amount float,
    pos_identifier uuid_type
);
```
Staging
```
create table staging.staging_pos_devices (
    pos_identifier uuid_type not null,
    credit_card_number int --this field is missing from the task description shoud be here else we can't link credit cards to new machines
);
```
Public
```
create table public.fact_pos_devices (
    pos_identifier uuid_type not null,
    credit_card_number --this field is missing from the task description shoud be here else we can't link credit cards to new machines
);
```

From `public.fact_account_balances` we can create a new table on credit card id level of granularity i.e `fact_issued_credit_card` which would contain columns like `credit_card_id`, `activation_date`, `deactivation_date` and `credit_facility_id` and `source_credit_card_id`.
With the `public.fact_credit_facility` table containing fields like `credit_limit`, `eligible_credit_limit`, `is_active` e.t.c.
.
.
.
.
.
#### SQL Views
- For dashboarding or reporting, we can point data sources to views. e.g.
pseudocode example
```
create view public.credit_card_utilisation_view as 
select some key(s), agg(*) metrics
from public.fact_account_balances
left join public.fact_pos_devices 
on join keys 
group by some key(s)
```


### ETL Data Pipeline 
1. Ingest .csv files from file server(azure file share) to Blob storage `raw_data` (Staging area).
2. Read .csv file from Blob store to VM memory and run data validations i.e. check if all required fields are there, no duplicate records, if the data is of correct type and whether should it transformed e.t.c.
3. If the data conforms, run transformations e.g. type casting, replace missing values e.t.c and load to `staging_data` (Staging area).
4. Load csv file from Blob store `staging_data` containeer to Data Warehouse temp staging area e.g `staging.staging_pos_devices`.
5. Perform row insert to target fact transactions table `public.fact_pos_devices`.
6. Log ETL job and persist logs to flat file(ideal would be using a time series DB, and monitor logs using tools like Grafana),these would be useful for debugging purposes, and monitoring job runtime over certain periods.
7. Call Slack client to alert on status of the ETL job(This would be our primary monitoring tool, over time we can use tools like Airflow).

### Source Systems 
- This is the source of our data, in this case Azure File server with timestamped data dumps.
- We'll be fetching data here to our staging area with Azure runbooks or Azure function(lambda function)
- https://charbelnemnom.com/sync-between-azure-file-share-and-azure-blob-container/

### Staging Layer 
- If we have access to OLTP we'll run the first component of the ETL job to Extract previous day transactions data, and dump to staging area.
- Our staging area will have two containers/buckets `raw_data` and `staging_data`
- This would be useful downstream to reduce our dependency on the source data.
- Another form of staging we'll perform is in the warehouse, this will be a temporal staging area to have more control on the ETL process.
- Our ETL process is more like: 

    Extract->Staging->Validate->Transform->Staging->Load

### Tranformation/Integration Layer
- This is the component of the ETL process where we integrate data from various sources(e.g. perform dataframe joins with data that could be coming from different source systems) or perform some tranformations e.g. encoding types, replacing nulls, performing calculations we couldn't do on source data e.t.c.
- We use python pandas mostly at this stage, if the data is too huge in the future we could look at frameworks like Apache Spark.
- The code will be executed by an Ubuntu virtual machine hosted in Azure, scheduled with crontab.

### Access Layer
- This will be used by Analysts for retriving reporting data.
- The Analysts can create views based off existing facts and dim tables or a DE will be reponsible if the code is more complex.
