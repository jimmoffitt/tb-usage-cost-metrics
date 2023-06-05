# Usage and Cost in Tinybird

{intro to pricing}

This data project contains an API endpoint that utilizes the [Service Data Sources](https://www.tinybird.co/docs/monitoring/service-datasources.html) to estimate the total usage and cost of all data sources and pipes in your Tinybird workspace.

The endpoint returns the total processed and stored data:
- Data processed on ingestion from `tinybird.datasources_ops_log`
- Data processed on API requests from `tinybird.pipe_stats`
- Data stored from `tinybird.datasources_storage`

The API accepts 3 parameters to control the result:
- `start_date` and `end_date` to filter the date range (default to yesterday and today, respectively, if either is not defined)
- `resources` to filter on one or more data source or pipe (defaults to all if not defined)

To calculate cost, the endpoint uses the PRO pricing as listed on the [website](https://www.tinybird.co/pricing). For enterprise customers, Tinybird offers volume-based discounts. 

## Prerequisites
Here we assume you already have a Tinybird account and know where to find an authentication token. If that is not the case, you can get started [HERE)(https://www.tinybird.co/docs).

We also assume that you have created a Tinybird Data Source, and likely have a curated pipe processing data. And hopefully, you have published an API Endpoint and are in a place where you need to monitor requests, data processing, and storage space.  

## Setting up your billing endpoint
There are two ways to create the "tinybird_billing" Pipe and API Endpoint: using the user interface (UI) or the command-line interface (CLI) tool. Using the CLI is the quickest method. Another advantage of using the CLI is that you will automatically end up with a Pipe and Nodes with descriptions that describe how they work.

Creating the Pipe with the UI is a manual process of copying and pasting three Nodes (see below), and likely about 10 minutes to set up. If you are brand new to Tinybird, manually building the Pipe is a great way to explore how Pipes and API Endpoints work.   

### Creating the billing endpoint with the CLI




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
