# Usage and Cost in Tinybird

## Introduction
As described in the [Tinybird documentation](https://www.tinybird.co/docs/billing/plans-and-pricing.html), pricing is driven by the amount of compressed data stored and the amount of data processed. Processed data includes the amount of data written to Data Sources and Materialized Views, and the amount of data read to generate API responses and serve Query API requests.   

This data project contains a ```tb-usage-cost``` API endpoint that utilizes the [Service Data Sources](https://www.tinybird.co/docs/monitoring/service-datasources.html) to estimate the total usage and cost of all Data Sources, Materialized Views, and Pipes in your Tinybird Workspace. With this project, you can quickly create an endpoint that returns billing estimates and measures the metrics that drive those estimates. 

The endpoint returns the total processed and stored data:
- Data written to Data Sources and Materialized Views from `tinybird.datasources_ops_log`
- Data read to generate API Endpoint responses and serve Query API requests from `tinybird.pipe_stats`
- Data stored in your Data Sources and Materialized Views from `tinybird.datasources_storage`

The API Endpoint accepts three query parameters to customize your request:
- `start_date` and `end_date` to customize the date range. By default, the API Endpoint returns data for the current month by setting the `start_time` to first day of the month, and `end_date` to today. These are useful to also look at periods of interest, such as when you are adding new Data Sources and API Endpoint features. 
- `resources` to filter on one or more Data Source or Pipe. Defaults to *all* if not defined. Multiple items should be comma-delimted. 

To calculate cost, the API Endpoint uses the [Professional](https://www.tinybird.co/docs/billing/plans-and-pricing.html#professional) pricing as listed on the [website](https://www.tinybird.co/pricing). For enterprise customers, Tinybird offers volume-based discounts. 

## Getting started

First, it is assumed that you already have a Tinybird account and know where to find an authentication token. If that is not the case, you can get started [HERE](https://www.tinybird.co/docs). 

Second, it is also assumed that you have created a Tinybird Data Source, and likely have a curated Pipe processing data. And probably, you have published an API Endpoint and are in a place where you need to monitor requests, data processing, and storage space.

For the quickest deployment of the billing API Endpoint, you should have the Tinybird CLI installed and ready to go. See the [CLI Quick Start](https://www.tinybird.co/docs/quick-start-cli.html) documentation for more details. If you prefer to not use the CLI, the API Endpoint can be created with the Tinybird user-interface by manually creating a Pipe and five Nodes. See below for instructions for both methods. 

## Setting up your billing endpoint
There are two ways to create the "tb_usage_cost" Pipe and API Endpoint: using the CLI or the UI. Using the CLI is the recommended method. It is quicker, and includes detailed descriptions of how the Pipe and Nodes work.

With the UI, it is a manual process of creating a Pipe, and copying and pasting the Nodes that define it. This process will take about 10 minutes. If you are brand new to Tinybird, manually building the Pipe is a way to explore how Pipes and API Endpoints work.   

### Creating the billing endpoint with the CLI

Here are the steps:
* The `tb_usage_cost.pipe` file (in the /endpoints project folder) contains the Pipe definition. Either clone this repository or copy the contents of this file to your local environment. 
* Navigate to the location of this file. 
* Start up the CLI and authorize with a Tinybird Token associated with the Workspace you want to update (```tb auth```).  
* Use ```tb push``` to load the ```tb_usage_cost``` Pipe into your Workspace. 


### Creating the billing endpoint with the UI
This method takes about ten minutes and is a great way to better understand how Pipes and Nodes work. 

### Nodes


#### cost_variables

```sql
SELECT
    0.07 AS price_per_processed_gb,
    0.34 AS price_per_stored_gb
```

#### processed_ingest
```sql
%
        SELECT
            datasource_name AS resource,
            'datasource' AS resource_type,
            sum(read_bytes) / pow(10, 9) AS gb_read,
            sum(written_bytes) / pow(10, 9) AS gb_written
        FROM tinybird.datasources_ops_log
        WHERE
            event_type NOT IN ('populateview', 'populateview-queued')
            {% if defined(start_date) and defined(end_date) %}
                AND toDate(timestamp) BETWEEN {{Date(start_date, description="Start date in format YYYY-MM-DD. Defaults to yesterday if not defined.")}}
                AND {{Date(end_date, description="End date in format YYYY-MM-DD. Defaults to today if not defined.")}}
            {% else %}
                AND toDate(timestamp) BETWEEN addDays(today(),-1) AND today()
            {% end %}
            {% if defined(resources) %}
                AND resource IN {{Array(resources, 'String', description="Comma-separated resources. Defaults to all if not defined.")}}
            {% end %}
        GROUP BY resource, resource_type
```

#### processed_APIs

```sql
%
SELECT
            pipe_name AS resource,
            if(resource == 'query_api', 'query_api', 'pipe') AS resource_type,
            sum(read_bytes_sum) / pow(10, 9) AS gb_read
        FROM tinybird.pipe_stats
        WHERE
            1 = 1
            {% if defined(start_date) and defined(end_date) %}
                AND date BETWEEN {{ Date(start_date) }} AND {{ Date(end_date) }}
            {% else %} AND date BETWEEN addDays(today(), -1) AND today()
            {% end %}
            {% if defined(resources) %}
                AND resource IN {{ Array(resources, "String") }}
            {% end %}
        GROUP BY resource, resource_type
```

#### storage

```sql
%
    SELECT
        toStartOfMonth(day) AS month,
        resource,
        resource_type,
        argMax(gb_stored, day) AS gb_stored
    FROM
        (
            SELECT
                toDate(timestamp) AS day,
                datasource_name AS resource,
                'datasource' AS resource_type,
                (max(bytes) + max(bytes_quarantine)) / pow(10, 9) AS gb_stored
            FROM tinybird.datasources_storage
            WHERE
                1 = 1
                {% if defined(start_date) and defined(end_date) %}
                    AND toDate(timestamp) BETWEEN {{ Date(start_date) }}
                    AND {{ Date(end_date) }}
                {% else %}
                    AND toDate(timestamp) BETWEEN addDays(today(), -1) AND today()
                {% end %}
                {% if defined(resources) %}
                    AND resource IN {{ Array(resources, "String") }}
                {% end %}
            GROUP BY day, resource, resource_type
        )
    GROUP BY month, resource, resource_type
    ORDER BY resource, month desc
```
#### endpoint

```sql
WITH
    (SELECT price_per_processed_gb FROM cost_variables) AS price_per_processed_gb,
    (SELECT price_per_stored_gb FROM cost_variables) AS price_per_stored_gb
SELECT
    resource,
    resource_type,
    round((gb_read + gb_written) * price_per_processed_gb, 4) AS cost_processed,
    round(gb_stored * price_per_stored_gb, 4) AS cost_stored,
    round(cost_processed + cost_stored, 4) AS total_cost,
    sum(gb_read) AS gb_read,
    sum(gb_written) AS gb_written,
    sum(gb_stored) AS gb_stored
FROM
    (
        SELECT resource, resource_type, gb_read, gb_written, 0 AS gb_stored
        FROM processed_ingest
        UNION ALL
        SELECT resource, resource_type, gb_read, 0 AS gb_written, 0 AS gb_stored
        FROM processed_APIs
        UNION ALL
        SELECT
            resource,
            resource_type,
            0 AS gb_read,
            0 AS gb_written,
            sum(gb_stored) AS gb_stored
        FROM storage
        GROUP BY resource, resource_type
    )
GROUP BY resource, resource_type
ORDER BY total_cost desc
```






## Working with the Tinybird CLI

To start working with data projects as if they were software projects, first install the Tinybird CLI in a virtual environment.
Check the [CLI documentation](https://docs.tinybird.co/cli.html) for other installation options and troubleshooting.

```bash
python3 -mvenv .e
. .e/bin/activate
pip install tinybird-cli
tb auth --interactive
```

Choose your region: __1__ for _us-east_, __2__ for _eu_

Go to your workspace, copy a token with admin rights and paste it. A new `.tinyb` file will be created.


## Project Description

```bash
├── endpoints
│   └── tb_usage_cost.pipe
```

In the `/endpoints` folder, there is 1 API endpoint:
- `tb_usage_cost` returns the usage and cost for a given date range and resource(s).

Push the data project to your workspace:

```bash
tb push
```
