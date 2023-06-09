# Welcome to lakeFS Cloud Sample Repository

## What is lakeFS?
lakeFS is an open source data version control system for data lakes.
It enables zero copy Dev / Test isolated environments, continuous quality validation, atomic rollback on bad data, reproducibility, and more.

## Introduction
Welcome to lakeFS sample-repo!
We've included step-by-step instructions, [pre-loaded data](#data-sets-examples) sets and [hooks](https://docs.lakefs.io/hooks/overview.html) to get familiar with lakeFS [versioning model](https://docs.lakefs.io/understand/model.html) and its capabilities.

We'll start by going over [lakeFS basic capabilities](#getting-started), such as creating a branch, uploading an object and committing that object.

We also included instructions on how to use [lakeFS Hooks](#diving-into-hooks), which demonstrates how to govern the data you merge into your main branch, for instance, making sure no PII is presented on the main branch, and that every commit to main includes certain metadata attributes.

## Getting Started

> **_NOTE:_** The hooks example below can be done by using the CLI or the UI, if you'd like to use the CLI, make sure to have [lakectl](https://docs.lakefs.io/reference/commands.html#configuring-credentials-and-api-endpoint) and [spark s3a](https://docs.lakefs.io/integrations/spark.html#access-lakefs-using-the-s3a-gateway) configured correctly.

We'll start by covering lakeFS basics.

Let's start by creating a branch:
```sh
# CLI
$ lakectl branch create lakefs://sample-repo/my-branch -s lakefs://sample-repo/main

# UI
Within the "sample-repo" repository -> click "Branches" -> click "Create Branch" -> fill in "my-branch" for Branch Name -> click "Create".
```

Great! you've created your first branch, you should now see it in the list of branches!

Now let's try uploading an object to the `my-branch` branch. Choose or create a local file in any format to test the upload and use its path in place of `/path/to/some/file` below: 

```sh
# CLI
$ lakectl fs upload lakefs://sample-repo/my-branch/file -s /path/to/some/file

# UI
Within the "sample-repo" repository -> click "Objects" -> pick "my-branch" from the branch drop down -> Click "Upload Object" -> Click "Choose file" and pick a file to upload -> click "Upload".
```

Now that we've uploaded the file, first, you'll see it in the stage area (uncommitted):
```sh
# CLI
lakectl diff lakefs://sample-repo/my-branch

# UI
Within the "sample-repo" repository -> click "Objects" -> pick "my-branch" from the branch drop down -> click "Uncommitted changes".
```

Let's commit the file:
```sh
# CLI
lakectl commit lakefs://sample-repo/my-branch

# UI
Still within the "my-branch" Uncommitted Changes -> click "Commit Changes" -> click once again "Commit Changes".
```

Let's explore some data 
> **_NOTE:_** for this example we'll demonstrate how to query parquet files using `DuckDB` from within the UI.

* Within the "sample-repo" repository, pick the "main" branch from the drop down
* Click the "world-cities-database-population" directory, and the "raw" directory within that
* Click the "part-00000-tid-1091049596617008918-5f8b8e42-730c-4cc2-ba06-3e5f4a4acff6-22194-1-c000.snappy.parquet" parquet file.

Now you should see the parquet file with a standard SQL query displaying the parquet file as table, with its columns.

Let's try to get some insights from this parquet, let's try to find out how many people live in the biggest city in each country. Replace the SQL query with the one below and click "Execute":

```sql
SELECT 
  country_name_en, max(population) AS biggest_city_pop
FROM
  read_parquet(lakefs_object('sample-repo', 'main', 'world-cities-database-population/raw/part-00000-tid-1091049596617008918-5f8b8e42-730c-4cc2-ba06-3e5f4a4acff6-22194-1-c000.snappy.parquet')) 
GROUP BY
 country_name_en
ORDER BY
   biggest_city_pop DESC
```

That was cool, wasn't it?

## Diving Into Hooks

Let's start by trying our first hook, which will ensure that certain metadata is present for any commit to the `main` branch. 

Upload a file (you can use the same one as above):

```sh
# CLI
$ lakectl fs upload lakefs://sample-repo/main/test -s /path/to/some/file

# UI
Within the "sample-repo" repository -> click "Upload object" -> click "Choose file" -> pick a file from your filesystem -> click "Upload".
```

Now that we've uploaded the file, first, you'll see it in the stage area (uncommitted):
```sh
# CLI
lakectl diff lakefs://sample-repo/main

# UI
Within the "sample-repo" repository -> click "Uncommitted changes".
```

Great! now let's try to commit that file:
```sh
# CLI
lakectl commit lakefs://sample-repo/main --message "Test Commit"

# UI
Within the "sample-repo" repository -> click "Uncommitted changes" -> click "Commit Changes" -> click "Commit Changes".
```

Ouch! we were caught in the act of trying to commit to `main` without the required attributes `owner` and `environment`! 

```sh
Branch: lakefs://sample-repo/main
pre-commit hook aborted, run id '5kepqvj1nilti6cut9hg': 1 error occurred:
	* hook run id '0000_0000' failed on action 'pre commit metadata field check' hook 'check_commit_metadata': runtime error: [string "lua"]:7: missing mandatory metadata field: owner


412 Precondition Failed
```

Let's retry that action, this time, with the required attributes:

```sh
# CLI
lakectl commit lakefs://sample-repo/main --message "Test Commit" --meta owner="John Doe",environment="production"

# UI
Within the "sample-repo" repository -> click "Uncommitted changes" -> click "Commit Changes" -> click "+ Add Metadata field" -> insert key: "owner", value: "John Doe" -> click "+ Add Metadata field" -> insert key: "environment", value: "production" -> click "Commit Changes".
```

Congrats! you've just made the first commit using the `pre-commit metadata validator` hook!

Let's jump on to a more advanced example.

Now, we'll try to sneak some private information into the main branch!

We've created a sneaky-branch for you to explore the second hook, we'll try merging a branch that contains a dangerous file (a file with the `email` column within a parquet file)

Let's try to merge that branch into main:
```sh
# CLI
lakectl merge lakefs://sample-repo/emails lakefs://sample-repo/main

# UI
Within the "sample-repo" repository -> click "Compare" tab -> pick the "main" as base branch -> Pick "emails" as compared to branch -> Click "Merge".
```

You should see the following output:
```sh
update branch main: pre-merge hook aborted, run id '5kepi1b1nilh6brjhmmg': 1 error occurred: * hook run id '0000_0000' failed on action 'pre merge PII check on main' hook 'check_blocked_pii_columns': runtime error: [string "lua"]:37: Column is not allowed: 'email': type: BYTE_ARRAY in path: tables/customers/dangerous.parquet : Error: update branch main: pre-merge hook aborted, run id '5kepi1b1nilh6brjhmmg': 1 error occurred: * hook run id '0000_0000' failed on action 'pre merge PII check on main' hook 'check_blocked_pii_columns': runtime error: [string "lua"]:37: Column is not allowed: 'email': type: BYTE_ARRAY in path: tables/customers/dangerous.parquet at tz.merge
```

Phew! we dodged a bullet here, no PII is present on our main branch.

That's all for our hooks demonstration, if you're interested in understanding more about hooks, [read our docs](https://docs.lakefs.io/hooks/).

## Sample Data

For your convenience, we've created a first repository with some sample data:

* [world-cities-database-population](https://www.kaggle.com/datasets/arslanali4343/world-cities-database-population-oct2022) - which contains information on the different cities and population (Licensed: Database Contents License (DbCL) v1.0)

* [nyc-tlc-trip-data](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page) - which contains information on New York City Yellow and Green taxi trip records.

We've also included a couple of hooks to help you get started:
* [pre-commit metdata-validation hook](./_lakefs_actions/pre-commit-metadata-validation.yaml) - which will verify on each commit to `stage` and `main` branches, that the following metadata attributes are present: `owner` (free text) and `environment` (must be one of "production", "staging" or "development").
* [pre-merge format-validation hook](./_lakefs_actions/pre-merge-format-validation.yaml) - which will verify on each merge to the `main` branch, that the following PII (Personal Identifiable Information) columns are **missing** within the `tables/customers/` and `tables/orders/` locations.

## Data Sets Examples

As mentioned above, we've included a couple of datasets for you to experience lakeFS with, here are some examples to get you started:

```sh
trips_df = spark.read.parquet("s3a://sample-repo/main/nyc-tlc-trip-data/yellow_tripdata_2022-11.parquet")

trips_df.printSchema()

trips_df.registerTempTable("yellow_trips")

query = """
SELECT 
    VendorID,
    tpep_pickup_datetime,
    tpep_dropoff_datetime,
    passenger_count,
    trip_distance,
    RatecodeID,
    payment_type,
    extra,
    mta_tax,
    tip_amount,
    tolls_amount,
    improvement_surcharge,
    total_amount,
    airport_fee
FROM 
    yellow_trips
"""

# create a new DataFrame based on the query
combo_df = spark.sql(query)

# Register the new DataFrame so that we can do EDA
combo_df.registerTempTable("combo")

# Let's start by exploring the data
combo_df.select("total_amount").describe().toPandas()

combo_df.select("passenger_count").describe().toPandas()
```

## lakeFS Cheatsheet

> **_NOTE:_** All lakectl commands described below, can be performed using our WebUI 

```sh
# Create Branch
lakectl branch create lakefs://my-repo/feature --source lakefs://my-repo/main

# Reading data via Spark into DataFrame
data = spark.read.parquet("s3a://my-repo/feature/sample_data/release=v1.12/type=relation/20220411_183014_00011_baakr_1aee3559-eec4-4d3c-8895-4b36d965a431").limit(20)

# View the DataFrame
data.show()

# Data Partitioning based on the version column and write it to the `feature` branch
data.write.partitionBy("version").parquet("s3a://my-repo/feature/sample_data/by_version")

# List files in the `feature` branch
lakectl fs ls lakefs://my-repo/feature/sample_data/

# Running diff between two branches
lakectl diff --two-way lakefs://my-repo/feature lakefs://my-repo/main
```

For more more information and subcommands, go to [our docs](https://docs.lakefs.io/).
