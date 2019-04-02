# Lab 2: Analysis of data in Amazon S3 using Amazon Redshift Spectrum

* [Deploying Amazon Redshift Cluster](#deploying-amazon-redshift-cluster)
* [Running AWS Glue Crawlers](#running-aws-glue-crawlers---csv--parquet-crawler)
* [Create Redshift Spectrum Scehma and reference external table form AWS Glue Data Catalog Database](#create-redshift-spectrum-scehma-and-reference-external-table-form-aws-glue-data-catalog-database)
* [Querying data from Amazon S3 using Amazon Redshift Spectrum](#querying-data-from-amazon-s3-using-amazon-redshift-spectrum)
* [Querying partitioned data using Amazon Redshift Spectrum](#querying-partitioned-data-using-amazon-redshift-spectrum)


## Architectural Diagram
![architecture-overview-lab2.png](images/architecture-overview-lab4.png)

## Deploying Amazon Redshift Cluster

In this section you will use a CloudFormation template to create an Amazon RedShift cluster. The template will also install [pgweb](https://github.com/sosedoff/pgweb), SQL Client for PostgreSQL, in an  Amazon EC2 instance to connect and run your queries on the launched Amazon Redshift cluster.

Alternatively, you can connect to the Amazon Redshift cluster using standard SQL Clients such as SQL Workbench/J. For more information refer http://docs.aws.amazon.com/redshift/latest/mgmt/connecting-using-workbench.html.

1. Login in to your AWS console and open the [Amazon CloudFormation Dashboard](https://ap-southeast-1.console.aws.amazon.com/cloudformation/home?region=ap-southeast-1])
2. Click **Create Stack**
3. Select **Specify an Amazon S3 template URL**
4. Copy paste the following S3 template URL

>[https://s3-ap-southeast-1.amazonaws.com/aws-slides-pdf/msd/redshiftspectrumglue-lab2.template.json](https://s3-ap-southeast-1.amazonaws.com/aws-slides-pdf/msd/redshiftspectrumglue-lab2.template.json)

6. Click **Next**

>**Note:**
>Click on the link [redshiftspectrumglue-lab2.template.json](../Lab2/redshiftspectrumglue-lab2.template.json) to view the Amazon CloudFormation template file

![Stack Name](images/stack-name.png)

8. Type a name *(e.g. RedshiftSpectrumLab)* for the **Stack Name**

![Cluster Config](images/cluster-config.png)

9. Enter the following **Parameters** for **Redshift Cluster Configuration**

    1. Choose *multi-node* for **ClusterType**
    2. Type *2* for the **NumberOfNodes**
    3. For **NodeType** select *ds2.xlarge*

![Database Config](images/database-config.png)

10.  Enter the following **Parameters** for **Redshift Database Configuration**.
    i. Type a name (e.g. dbadmin) for **MasterUserName**.
    ii. Type a password for **MasterUserPassword**.
    iii. Type the a name (e.g. taxidb) for **DatabaseName**.
    iv. Type the IP address of your local machine for **ClientIP**.

![Glue Crawler](images/glue-crawler.png)

11. Enter the following **Parameters** for **Glue Crawler Configuration**
    1. Type the name(e.g. taxi-spectrum-db) for **GlueCatalogDBName**.    
    2. Type the name(e.g. csvCrawler) for **CSVCrawler**.
    3. Type the name(e.g. parquetCrawler) for **ParquetCrawler**.

12. Click **Next**

![Tags](images/tags.png)

13. [Optional] In the **Tags** sub-sections in **Options** type a **Key** name *(e.g. Name)* and **Value** for key.
14. Click **Next**

![Acknowledge](images/acknowledge.png)

15. Check **I acknowledge that AWS CloudFormation might create IAM resources.**
16. Click **Create**

> **Note:** This is may take approximately 15 minutes

17. Ensure that status of the Amazon CloudFromation stack that you just create is **CREATE_COMPLETE**
18. Select your Amazon CloudFormation stack *(RedshiftSpectrumLab)*
19. Click on the **Outputs** tab
20. Review the list of **Key** and thier **Value** which will look like the following.

![Output](images/output.png)

You can also check out the Resources created from the stack. Here are the key resources:
- 1x RedShift Database cluster
- 2x Glue Crawler (1 parquet crawler and 1 csv crawler)
- 1x Glue database
- 1x Linux instance for pgweb (SQL Client)
- IAM roles and security groups

---

## Running AWS Glue Crawlers - CSV & Parquet Crawler
1. Open [AWS Management Console for Glue](https://ap-southeast-1.console.aws.amazon.com/glue/home?region=ap-southeast-1#)
2. Go to AWS Glues Crawlers page by clicking on **Crawlers** in the navigation pane

![Crawlers](images/crawlers.png)

3. Select the AWS Glue Crawler for CSV (e.g. csvCrawler)
4. Click **Run crawler**
5. Select the AWS Glue Crawler for CSV (e.g. csvCrawler)
6. Click **Run crawler**

> Note: This may take approximately 5 min for both the crawlers to parse the data in CSV and Parquet format.

> If you are interested, you can view the raw data of what we are crawling from here: [CSV](https://console.aws.amazon.com/s3/home?region=ap-southeast-1&bucket=us-west-2.serverless-analytics&prefix=NYC-transportation/taxi/) | [Parquet](https://console.aws.amazon.com/s3/home?region=ap-southeast-1&bucket=us-west-2.serverless-analytics&prefix=canonical/NY-Pub/).

![Crawlers Completed](images/crawlers-completed.png)

7. Wait for the **Status** of both the crawlers to be in the *Ready* state

Now that you have run the crawlers lest ensure that new tables *taxi* and *ny_pub* been created.

8. To to the list of databases in the AWS Glue Data Catalog click on **Databases** in the navigation pane.
9. Click on **taxi-spectrum-db**

![taxispectrumdb](images/taxi-spectrum-db.png)

10. Click on **Tables in taxi-spectrum-db**. If you see no tables listed, click on the refresh button.

![taxispectrumdb-tables](images/taxi-spectrum-db-tables.png)

11. Click on **taxi** to review the table definition and schema
12. Navigate back and click on **ny_pub** to review the table definition and schema

>**Note:**
>The good news is that you don’t have to create a new table or definition to read the CSV document we just looked at. With AWS Glue crawlers, you have already inferred the schema and created tables namely taxi and ny_pub.

13. Click on **View partitions** to review the partition metadata

>**Note:**
> The major advantage of Glue Crawlers is that they understand the partitions based on the S3 object prefix and automatically create the table with partitions as part of the crawling.

---

## Create Redshift Spectrum Schema and reference external table from AWS Glue Data Catalog Database

1. Open the [Amazon CloudFormation Dashboard](https://ap-southeast-1.console.aws.amazon.com/cloudformation/home?region=ap-southeast-1])
2. Select your Amazon CloudFormation stack *(RedshiftSpectrumLab)*
3. Click on the **Outputs** tab
4. Navigate to the **pgWeb** URL
5. In the pgWeb console ensure that the **SQL Query** tab is selected
6. Copy the following statement to create a database *(e.g. taxispectrum)* in Redshift Spectrum

```sql
  create external schema taxispectrum from data catalog
  database 'taxi-spectrum-db'
  iam_role '<specify the redshift IAM Role arn from the CloudFormation outputs section>'
```
7. Replace the *<specify the redshift IAM Role arn from the CloudFormation output section'>* in the statment with the value of **redshiftIAMRole** from the **Outputs** tab of the Amazon CloudFromation stack *(RedshiftSpectrumLab)* you created as part of the lab.
> Note: The IAM role must be in single quotes

8. Click **Run Query**

> Note: You can create an external table in Amazon Redshift, AWS Glue, Amazon Athena, or an Apache Hive metastore. For more information, see [Getting Started Using AWS Glue](http://docs.aws.amazon.com/glue/latest/dg/getting-started.html) in the AWS Glue Developer Guide, [Getting Started](http://docs.aws.amazon.com/athena/latest/ug/getting-started.html) in the Amazon Athena User Guide, or [Apache Hive](http://docs.aws.amazon.com/emr/latest/ReleaseGuide/emr-hive.html) in the Amazon EMR Developer Guide. If your external table is defined in AWS Glue, Athena, or a Hive metastore, you first create an external schema that references the external database. Then you can reference the external table in your SELECT statement by prefixing the table name with the schema name, without needing to create the table in Amazon Redshift. For more information, see [Creating External Schemas for Amazon Redshift Spectrum](http://docs.aws.amazon.com/redshift/latest/dg/c-spectrum-external-schemas.html).

## Querying data from Amazon S3 using Amazon Redshift Spectrum

Now that you have created the schema, you can run queries on the data set and see the results in PGWeb Console.

1. Copy the following statement into the query pane, and then choose **Run Query**.

```sql
    SELECT * FROM taxispectrum.taxi limit 10
```

Results for the above query look like the following:

![Query Results](images/query-result.png)

2.	Copy the following statement into the query pane, and then choose **Run Query** to get the total number of taxi rides for yellow cabs.

```sql
    SELECT COUNT(1) as TotalCount FROM taxispectrum.taxi
```
Results for the above query look like the following:

![taxispectrum count](images/total-count-taxispectrum.png)

3. Copy the following statement into the query pane, and then choose **Run Query** to query for the number of rides per vendor, along with the average fair amount for yellow taxi rides

```sql
    SELECT
    CASE vendorid
         WHEN '1' THEN 'Creative Mobile Technologies'
         WHEN '2' THEN 'VeriFone Inc'
         ELSE CAST(vendorid as VARCHAR) END AS Vendor,
    COUNT(1) as RideCount,
    avg(total_amount) as AverageAmount
    FROM taxispectrum.taxi
    WHERE total_amount > 0
    GROUP BY (1)
```

Results for the above query look like the following:

![Vendor result](images/vendor-taxispectrum-result.png)

## Querying partitioned data using Amazon Redshift Spectrum

By partitioning your data, you can restrict the amount of data scanned by each query, thus improving performance and reducing cost. Amazon Redshift Spectrum leverages Hive for [partitioning](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DDL#LanguageManualDDL-AlterPartition) data. You can partition your data by any key. A common practice is to partition the data based on time, often leading to a multi-level partitioning scheme. For example, a customer who has data coming in every hour might decide to partition by year, month, date, and hour. Another customer, who has data coming from many different sources but loaded one time per day, may partition by a data source identifier and date.


Now that you have added the partition metadata to the Athena data catalog you can now run your query.

1. Copy the following statement into the query pane, and then choose **Run Query** to get the total number of taxi rides

```sql
    SELECT count(1) as TotalCount from taxispectrum.ny_pub
```
Results for the above query look like the following:

![Total Count.png](images/total-count-nypub.png)

>**Note:**
> This query executes much faster because the data set is partitioned and it in optimal format - Apache Parquet (an open source columnar).

2. Copy the following statement into the query pane, and then choose **Run Query** to get the total number of taxi rides by year

```sql
    SELECT YEAR, count(1) as TotalCount from taxispectrum.ny_pub GROUP BY YEAR
```
Results for the above query look like the following:
![Total Count Year.png](images/total-count-taxispectrum-year.png)

3. Copy the following statement into the query pane, and then choose **Run Query** to get the top 12 months by total number of rides across all the years

```sql
    SELECT YEAR, MONTH, COUNT(1) as TotalCount
    FROM taxispectrum.ny_pub
    GROUP BY (1), (2)
    ORDER BY (3) DESC LIMIT 12
```
Results for the above query look like the following:
![Total Count Limit](images/total-count-taxispectrum-limit.png)

4. Copy the following statement into the query pane, and then choose **Run Query** to get the monthly ride counts per taxi time for the year 2016.

```sql
    SELECT MONTH, TYPE, COUNT(1) as TotalCount
    FROM taxispectrum.ny_pub
    WHERE YEAR = 2016
    GROUP BY (1), (2)
    ORDER BY (1), (2)
```
Results for the above query look like the following:
![Total Count Group](images/total-count-taxispectrum-group.png)

5. Copy the following statement anywhere into the query pane, and then choose **Run Query**.

```sql
    SELECT MONTH, TYPE,
      avg(trip_distance) avgDistance,
      avg(total_amount/trip_distance) avgCostPerMile,
      avg(total_amount) avgCost,
      percentile_cont(0.99)
      within group (order by total_amount)
    FROM taxispectrum.ny_pub
    WHERE YEAR = 2016 AND (TYPE = 'yellow' OR TYPE = 'green')
    AND trip_distance > 0 AND total_amount > 0
    GROUP BY MONTH, TYPE
    ORDER BY MONTH
```

Results for the above query look like the following:

![Total Count Group Avg](images/total-count-taxispectrum-group-avg.png)

## Deleting the Amazon CloudFormation Stack

Now that you have successfully queried the dataset using Amazon Redshift Spectrum, you need to tear down the stack that you deployed using the Amazon CloudFormation template.

1. Open the [Amazon CloudFormation Dashboard](https://ap-southeast-t.console.aws.amazon.com/cloudformation/home?region=ap-southeast-1)
2. Enable the check box next to the name of the stack *(e.g. RedshiftSpectrumLab)* that you deployed at the beginingo fo the Lab.
3. Click on **Actions** drop down button.
4. Select **Delete Stack**'
5. Click **Yes, Delete** on the *Delete Stack* pop dialog
6. Ensure that Amazon CloudFromation stack name *(e.g. RedshiftSpectrumLab)* is no longer showing in the list of stacks.
