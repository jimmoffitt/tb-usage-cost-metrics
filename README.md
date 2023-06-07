# Usage and Cost in Tinybird

## Introduction
As described in the [Tinybird documentation](https://www.tinybird.co/docs/billing/plans-and-pricing.html), pricing is driven by the amount of compressed data stored and the amount of data processed. Processed data includes the amount of data written to Data Sources and Materialized Views, and the amount of data read to generate API responses and serve Query API requests.   

This data project contains a ```tb-usage-cost``` API Endpoint that utilizes the [Service Data Sources](https://www.tinybird.co/docs/monitoring/service-datasources.html) to estimate the total usage and cost of all Data Sources, Materialized Views, and Pipes in your Tinybird Workspace. With this project, you can quickly create an endpoint that returns billing estimates and measures the metrics that drive those estimates. 

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

### Setting up your billing endpoint
There are two ways to create the ```tb_usage_cost``` Pipe and API Endpoint: using the CLI or the UI. Using the CLI is the recommended method. It is quicker, and includes detailed descriptions of how the Pipe and Nodes work.

With the UI, it is a manual process of creating a Pipe, and copying and pasting the Nodes that define it. This process will take about 10 minutes. If you are brand new to Tinybird, manually building the Pipe is a way to explore how Pipes and API Endpoints work.   

## Creating the billing endpoint with the CLI

Creating the endpoint with the CLI involves creating a Pipe with multiple Nodes, then publishing the API Endpoint with the UI. 

* **Add the ```tb_usage_cost``` Pipe in your Workspace**
  * The `tb_usage_cost.pipe` file (in the /endpoints project folder) contains the Pipe definition. Either clone this repository or copy the contents of this file to your local environment. 
  * Navigate to the location of this file. 
  * Start up the CLI and authorize with a Tinybird Token associated with the Workspace you want to update (```tb auth```).  
  * Use ```tb push``` to load the ```tb_usage_cost``` Pipe into your Workspace. 
  * Navigate to your Workspace and check for the ```tb_usage_cost``` Pipe and take a tour of the Nodes.  

* **Publishing the ```tb_usage_cost``` API Endpoint**
  * Select the ```tb_usage_cost``` Pipe and click on the "Create API Endpoint" button and select the ```endpoint``` Node. 
  * That's it. The endpoint is now available at https://api.tinybird.co/v0/pipes/tb_usage_cost.json.
  
## Creating the billing endpoint with the UI
This method takes around ten minutes and is a way to better understand how Pipes and Nodes work. The first step is creating a new Pipe, setting up a sequence of five Nodes, and publishing the API Endpoint. 

* **Create the ```tb_usage_cost``` Pipe in your Workspace**
  * Select the Pipes "+" icon to create a new Pipe.
  * Rename the Pipe to ```tb_usage_cost```, or name it to what you prefer.

* **Create Nodes**
  * Copy the Node content from below and paste it into the Node editor box. After updating the Node contents, press the ```Run``` button and see the results. 
  * Repeat for this process for each Node, in the order below.    

* **Publishing the ```tb_usage_cost``` API Endpoint**
  * Select the ```tb_usage_cost``` Pipe and click on the "Create API Endpoint" button and select the ```endpoint``` Node. 
  * That's it. The endpoint is now available at https://api.tinybird.co/v0/pipes/tb_usage_cost.json.

## Nodes 

The ```tb_usage_cost``` Pipe contains the following Nodes, defined in the following order. All of these are defined in the ```tb_usage_cost.pipe``` file.

### cost_variables

This Node sets two contants for the Tinybird prices for stored and processed data per gigabyte (GB). Details are available here: [tinybird.co/pricing](https://www.tinybird.co/pricing) and [tinybird.co/docs/billing/plans-and-pricing](https://www.tinybird.co/docs/billing/plans-and-pricing.html).

```sql
SELECT
    0.07 AS price_per_processed_gb,
    0.34 AS price_per_stored_gb
```

### processed_ingest

Retrieves from the ```tinybird.datasources_ops_log``` Service Data Source the amount of processed data associated with writing/ingesting data to Data Sources. This includes when you create, append, delete or replace data in a Data Source. This also includes the data written to Materialized Views *after*  they are created. There is no charge when you first create and populate a Materialized View, only incremental updates are billed.

By default this Node provides totals for the month-to-date.

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

### processed_APIs
Retrieves from tht ```tinybird.pipe_stats``` Service Data Source the daily amount of processed data associated with reading data from Data Sources. You read data when you run queries against your Data Sources to generate responses to API Endpoint requests. You also read data when you make requests to the Query API.

A reminder that manually executing a query inside the Tinybird UI is free. This means that you can develop, experiment and iterate inside the Tinybird UI without incurring additional cost.

By default this Node provides totals for the month-to-date.

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

### storage

Retrieves from the ``` tinybird.datasources_storage``` Service Data Source the maximum amount of storage from the previos day. Data storage refers to the disk storage of all the data you keep in Tinybird. This includes all of your Data Sources and Materialized Views, and well as any quarantined data. Data storage pricing is based on the amount of storage used after compression. 

For billing, a measurement of this storage is made at the end of every month. This Node queries for the maximum amount of stored data over the previous day by default. The maximum storage over a different period can be queried using the `start_date` and `end_date` parameters.
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
### endpoint

This Node compiles results from the previous Nodes and calculates the estimated stored and processed data costs for each Data Sources, Materialize Views, and Pipes, as well as any Query API usages. To estimate total cost for the selected data range (defaults to month-to-date), you need to add all the ```total_cost``` values.

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





