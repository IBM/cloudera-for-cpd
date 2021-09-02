# Virtualizing Cloudera Data using IBM Cloud Pak for Data

The purpose of this article is to highlight the ability of IBM Cloud Pak for Data to integrate with other market-leading data management platforms, such as Cloudera Data Platform.

Due to the complexity of installing Cloudera Data Platform and Cloud Pak for data, it is assumed the reader will already have access to these environments. If not, you can still follow along and get insights into how integration can be accomplished.

## Topics

1. [Overview of technologies](#1-overview-of-technologies)
1. [Architecture overview](#2-architecture-overview)
1. [Load data on Cloud Pak for Data](#3-load-data-on-cloud-pak-for-data)
1. [Create Db2 connection to access data on Cloud Pak for Data](#4-create-db2-connection-to-access-data-on-cloud-pak-for-data)
1. [Load data on Cloudera Data Platform](#5-load-data-on-cloudera-data-platform)
1. [Use IBM BigSQL to synchronize data from Hive table into Cloud Pak for Data](#6-use-ibm-bigsql-to-synchronize-data-from-hive-table-into-cloud-pak-for-data)
1. [Use Data Virtualization to merge data from Cloudera and Cloud Pak for Data](#7-use-data-virtualization-to-merge-data-from-cloudera-and-cloud-pak-for-data)
1. [Build Cognos Analytics dashboard to visualize merged data](#8-build-cognos-analytics-dashboard-to-visualize-merged-data)

## Video

To view the following steps as a video click on the image below:

[![NXuqBUnZfxg](https://img.youtube.com/vi/NXuqBUnZfxg/0.jpg)](https://www.youtube.com/watch?v=NXuqBUnZfxg)

## 1. Overview of technologies

Let's start with an overview of the technologies that we will be utilizing:

### Cloudera Data Platform

[Cloudera Data Platform](https://docs.cloudera.com/cdp-private-cloud/latest/index.html) (CDP Private Cloud) is built for hybrid cloud, connecting on-premises environments to public clouds. It provides a disaggregation of compute and storage, allowing for independent scaling of compute and storage clusters. Analytics running on containerized compute nodes, scalable object store, and a secure data lake.

* [Cloudera Manager](https://docs.cloudera.com/cloudera-manager/latest/index.html) provides administration of your Cloudera cluster, including operations for installation, upgrading, and host management and monitoring.

* [Apache Hive](https://hive.apache.org/) runs over the Hadoop framework and provides an SQL-like interface for processing and querying the HDFS data.

* [Apache Impala](https://impala.apache.org/) is similar to Hive, but with low-latency and high concurrency it provides a better option for interactive computing.

* [Apache Knox](https://knox.apache.org/) is an application gateway for interacting with the REST APIs and UIs of Hadoop deployments. Knox presents consumers with one endpoint for access to all the required services across multiple Hadoop clusters.

### IBM Cloud Pak for Data

[IBM Cloud Pak for Data](https://www.ibm.com/support/producthub/icpdata/docs/content/SSQNUZ_latest/cpd/overview/overview.html) is a unified, pre-integrated data and AI platform that runs natively on the [Red Hat OpenShift Container platform](https://www.openshift.com/products/container-platform). Services are delivered with an open and extensible cloud native platform for collecting, organizing, and analyzing data. It’s a single interface to perform end-to-end analytics with built-in governance. It also supports and governs the end-to-end AI workflow.

* [IBM Db2](https://www.ibm.com/support/producthub/icpdata/docs/content/SSQNUZ_latest/svc-welcome/db2oltp.html) is a Relational Database Management System (RDBMS). Along with providing the Db2 relational database, it includes a family of tools that allows you to manage both structured and unstructured data across on-premises and multi-cloud environments.

* [IBM Big SQL](https://www.ibm.com/support/producthub/icpdata/docs/content/SSQNUZ_latest/svc-welcome/bigsql.html) provides a single database connection to query data across Hadoop and other relational/NoSQL databases. Data can reside on local systems or on the cloud.

* [Data Virtualization](https://www.ibm.com/support/producthub/icpdata/docs/content/SSQNUZ_latest/svc-welcome/dv.html) can query data across many systems without having to copy and replicate data. Is accurate because you’re querying the latest data at its source.

* [IBM Cognos Analytics Dashboard](https://www.ibm.com/support/producthub/icpdata/docs/content/SSQNUZ_latest/svc-welcome/ca.html): self-service analytics, infused with AI and machine learning, enable you to create stunning visualizations and share your findings through dashboards and reports.

## 2. Architecture overview

### Versions

> We have tested CDP versions 7.1.6 and 7.1.7 and Cloud Pak for Data 3.5.2 and 4.0.1

### CDP configuration

We provisioned an 8 VMs running CentOS 7.9 on IBM Cloud to host our CDP instance.

We also provisioned two additional virtual servers -- one for a [bastion node](https://en.wikipedia.org/wiki/Bastion_host) so that we could easily SSH into all the other nodes, and the other for an Active Directory Server to manage authentication for our cluster.

![vm-architecture](images/vm-architecture.png)

Here are the nodes listed in our IBM Cloud devices list:

![ibm-cloud-vms](images/ibm-cloud-vms.png)

* `cid-bastion` is our bastion
* `cid-adc` is our Active Directory server
* `cid-vm-01` to `cid-vm-03` are our master nodes
* `cid-vm-04` to `cid-vm-06` are our worker nodes
* `cid-vm-07` and `cid-vm-08` are our edge nodes

#### Windows 2019 Server

We set up Windows 2019 Server [`cid-adc`]. We took advantage of it's Active Directory service for user management, and it's Domain Services to manage DNS. We created a private domain `cdplab.local` so as much of our traffic would stay on private networks.

#### Cloudera Manager

We manage our CDP using Cloudera Manager running on `cid-vm-01`. Here you see our `cpdlab` cluster and list of hosts:

![cloudera-manager](images/cloudera-manager.png)

Here is the list of services available on the cluster:

![cdp-service-list](images/cdp-service-list.png)

### IBM Cloud Pak for Data configuration

Cloud Pak for Data comes pre-packaged with a host of tools and services that can be instantiated. We provisioned Data Virtualization, Db2, Big SQL, and Cognos Analytics.

![cpd-instances](images/cpd-instances.png)

## 3. Load data on Cloud Pak for Data

In Cloud Pak for Data we will add data to the Db2 instance. The data [Sales.csv](data/Sales.csv) is meant to represent transactions and contains information such as profit, revenue, and sales price.

The Db2 service has been provisioned in our Cloud Pak for Data instance. To view it, from the main Cloud Pak for Data menu, select `Services`, and then click on `Instances`. Click on the Db2 instance name to bring up the details panel (the `⋮` icon). Click on the `Open database` button in the top right corner.

![cpd-db2-instance](images/cpd-db2-instance.png)

From the `Summary` panel, click `Load Data`.

![cpd-db2-load-option](images/cpd-db2-load-option.png)

From the `File selection` window, upload the `Sales.csv` CSV file from our [data directory](data), then click `Next`.

![cpd-db2-load-page-1](images/cpd-db2-load-page-1.png)

To create a new `sales` table, first create a new `GREAT_OUTDOORS` schema, and then click the `New table +` button. Enter `SALES` as the new table name and then click `Create`. 

![cpd-db2-load-page-2](images/cpd-db2-load-page-2.png)

Once our table is created, click `Next`.

![cpd-db2-load-page-3](images/cpd-db2-load-page-3.png)

You can then view all of the data rows that will be uploaded into our new `SALES` table.

![cpd-db2-load-page-4](images/cpd-db2-load-page-4.png)

Note that you can toggle the `Header in first row` button to view the column header names.

Click `Next` to start the loading of the data. If successful, there should be no error messages.

To view the data, click again on `Summary` panel, and click `Explore` > `Tables`.

## 4. Create a JDBC connection to Db2

After creating the data, we need to create a new connection to access it. Note that this task requires you to log in as `admin` on your CPD console.

From the main menu, select `Data` and then `Platform connections`.

![cpd-data-platform-connections](images/cpd-data-platform-connections.png)

From the `Platform connections` panel, click `New connection`.

![cpd-new-connection](images/cpd-new-connection.png)

From the `New connection` panel, in the IBM list of connection options, select `Db2`.

![cpd-connection-types](images/cpd-connection-types.png)

To fill in the data for the new connection, take the values from the following 2 panels:

1. **Panel 1**: The Db2 instance panel, which you can navigate to by selecting `Services` and then `Instances` from the main menu on the CPD Home page, and then clicking on the `Db2` instance name.

    ![cpd-db2-instance-details](images/cpd-db2-instance-details.png)

2. **Panel 2**: The Db2 instance connection information panel. From the previous details page, click on `Open database`, then select `Connection Information` from the `Summary` drop-down menu.

    ![cpd-db2-connection-info](images/cpd-db2-connection-info.png)

Transfer the data from these panels into the new connection panel and click `Test` to ensure it works.

![cpd-db2-connection-test](images/cpd-db2-connection-test.png)

* Enter unique name for the connection `Name`.
* `Database` name can be found in panel 1.
* `Hostname` can be found in panel 2.
* `Port` can be parsed from the JDBC Connection URL (SSL) found in panel 1.
* `Username` and `Password` are your Cloud Pak for Data credentials.
* `SSL certificate` can be downloaded from panel 1.

Click `Create` to create the connection.

## 5. Load data on Cloudera Data Platform

On the Cloudera side, we will use HDFS to populate our Hive database. In step uses the data set [Products.csv](data/Products.csv) that contains information about products (brand, description, price, ID, etc.).

The first step is to add the data to a users HDFS directory. Download the data where appropriate and run the command below. Replacing the last value with a directory you have access to.

```bash
hdfs dfs -put Products.csv /user/<your-user-name>
```

Next we'll connect to Hive using the `beeline` CLI tool. To do this download the Hive-on-Tez configuration file by selecting the `Download Client Configuration` option listed under the `Actions` drop-down menu for our Cloudera `Hive on Tez` service.

![cloudera-hive-client-config](images/cloudera-hive-client-config.png)

Unzip the downloaded file and open the `beeline-site.xml` file to find the URL associated with the property `beeline-hs2-jdbc-url-hive_on_tez`. Copy this value and then exit the file.

Log into one of the nodes in your CPD cluster and start the `beeline` CLI by typing:
  
```bash
beeline
```

>**NOTE**: The authentication process will depend on how you configured your CPD Private instance.
  
Connect to the Hive service by typing the command below, replacing the last value with the value copied from the previous step:
  
```bash
!connect <jdbc-url>
```

Use the following SQL command to create a new database and switch context to it:

```SQL
CREATE DATABASE great_outdoors;
USE great_outdoors;
```

Use the following SQL command to create a new table for the products:

```SQL
CREATE TABLE products (
  `Product_number` INT,
  `Product_line` STRING,
  `Product_type` STRING,
  `Product` STRING,
  `Introduction_date` STRING,
  `Product_brand` STRING,
  `Product_color` STRING,
  `Product_size` STRING,
  `Product_description` STRING,
  `Unit_cost` STRING,
  `Unit_price` STRING)
row format delimited fields terminated by ','
STORED AS textfile TBLPROPERTIES('transactional'='false');
```

Use the following SQL command to load the data from HDFS to the table:

```SQL
LOAD DATA INPATH '/<user-dir>/Products.csv' OVERWRITE INTO TABLE products;
```

If the data is loaded correctly you can query the data by running the SQL command below:

```SQL
SELECT * from products;
```

## 6. Use IBM BigSQL to synchronize data from Hive table into Cloud Pak for Data

At this point we'll use BigSQL to synchronize data from Hive to Db2 so make sure you have the service installed on your CPD cluster.

![cpd-big-sql-instance](images/cpd-big-sql-instance.png)

Open the `Big SQL` service by clicking on `Open` in the action (`⋮`) menu.

From the `Summary` dropdown menu click on the `Run SQL` options.

Click on the `Create new +` button, enter the following SQL command, and click on the `Run all` button.

```SQL
CALL SYSHADOOP.HCAT_SYNC_OBJECTS('GREAT_OUTDOORS', 'PRODUCTS', 't');
```

From the dropdown menu click `Explore` > `Hadoop tables`. Verify that the product data shows up when going to the `GREAT_OUTDOORS` schema and viewing the `PRODUCTS` table.

### Create a JDBC connection to BigSQL

From the main menu, select `Data` and then `Platform connections`.

![cpd-data-platform-connections](images/cpd-data-platform-connections.png)

From the `Platform connections` panel, click `New connection`.

![cpd-new-connection](images/cpd-new-connection.png)

From the `New connection` panel, in the IBM list of connection options, select `Db2 BigSQL`.

Below is an example of a BigSQL connect that you can build your own connection from.

![cpd-big-sql-connection-details](images/cpd-big-sql-connection-details.png)

> **NOTE** To determine host name, follow the procedures outlined in the [Setting up a connection to Db2 Big SQL](https://www.ibm.com/docs/en/cloud-paks/cp-data/3.5.0?topic=sql-setting-up-connection-db2-big) instructions. We used the second suggested method - "Passthrough secure route method". The result was a route that we could use for the `Hostname or IP Address` field.

For `Username` and `Password` use your Cloudera Data Platform credentials.

For SSL certificate, you must run an `OpenSSL` command to the hostname just generated.

```bash
openssl s_client -showcerts -servername <hostname without port> -connect <hostname:port> 
```

This will return a chain certificate, so you will need to copy both and paste into the SSL certificate field. Note, be sure to remove any extraneous lines outside of the `BEGIN` and `END` certificate delimiters.

Click the `Test` button to ensure the connection works.

<!-- TODO: add screenshot -->

## 7. Use Data Virtualization to merge data from Cloudera Data Platform and Cloud Pak for Data

From the Cloud Pak for Data home page menu, open up the `Data` menu and click on `Data Virtualization`.

![cpd-dv-start](images/cpd-dv-start.png)

### Add data sources

The default landing page for `Data Virtualization` is `Data Sources`. Click on the `Add data source` button.

![cpd-dv-data-sources-start](images/cpd-dv-data-sources-start.png)

Choose to option to `Select existing connection` and choose the `cdp-pvc-base` connection that represents the connection we made to BigSQL (which reads from our Hive databases on Cloudera). Click `Add`.

![cpd-dv-add-connection](images/cpd-dv-add-connection.png)

Repeat the step for the `db2-on-cpd` connection representing the connection we made to Db2.

### Virtualize tables

From the main drop-down menu, click on `Virtualize`, which is listed under the `Virtualization` option.

![cpd-dv-virtualize-option](images/cpd-dv-virtualize-option.png)

From the `Table` list, select each of the tables we have created, and add them to the `Cart`.

* The `SALES` table associated with the `db2-on-cpd` connection (data from Cloud Pak for Data):

    ![cpd-dv-add-sales-table](images/cpd-dv-add-sales-table.png)

* The `PRODUCTS` table associated with the `cpd-pvc-base` connection (data from Cloudera Data Platform):

    ![cpd-dv-add-products-table](images/cpd-dv-add-products-table.png)

Click the `Add to cart` button.

Once available, click the `View cart` button.

![cpd-dv-full-cart](images/cpd-dv-full-cart.png)

Resolve any conflicts, assign to your CPD project, and then click `Virtualize`.

Once complete, click the `View my virtualized data` button on the modal dialog.

> **NOTE**: You can also navigate to your data by clicking on the `My virtualized data` option from the main Data Virtualization menu.

![cpd-dv-view-my-data-option](images/cpd-dv-view-my-data-option.png)

You should now see your new virtualized data tables listed.

![cpd-dv-tables-for-join](images/cpd-dv-tables-for-join.png)

### Join tables

Select the Sales and Products tables, then click the `Join` button in the table header.

We won't need all of the columns for our join, so start by de-selecting all of the column names.

The columns we are interested in are:

Products table: `PRODUCT_NUMBER`, `PRODUCT`, `PRODUCT_TYPE`
Sales table: `Product number`, `REVENUE`, `Gross profit`

![cpd-dv-select-columns](images/cpd-dv-select-columns.png)

Connect the primary keys together by clicking in the `Product number` row in the `Sales` table, and drag and drop the cursor onto the `PRODUCT_NUMBER` row in the `PRODUCTS` table.

![cpd-dv-connect-keys](images/cpd-dv-connect-keys.png)

If successful, the table in the right hand margin should reflect the joining of the keys.

Click the `Open in SQL editor` button at the top of the right hand margin.

![cpd-dv-go-to-sql-editor](images/cpd-dv-go-to-sql-editor.png)

In the editor window, you will see the SQL statement that was generated from the actions from the previous step. We will need to modify this slightly so that we get total `SUM` values for revenue and gross profit. The `GROUP BY` statements allow us to generate the sums. Note that we are also changing the view name.

```SQL
CREATE VIEW HIGHEST_REVENUE_PRODUCTS
AS
SELECT
    "HIGHREV"."PRODUCTS"."PRODUCT_NUMBER" AS "HIGHREV_PRODUCTS_PRODUCT_NUMBER",
    "HIGHREV"."PRODUCTS"."PRODUCT" AS "HIGHREV_PRODUCTS_PRODUCT",
    "HIGHREV"."PRODUCTS"."PRODUCT_TYPE" AS "HIGHREV_PRODUCTS_PRODUCT_TYPE",
    SUM("HIGHREV"."SALES"."REVENUE") AS "HIGHREV_SALES_REVENUE",
    SUM("HIGHREV"."SALES"."Gross profit") AS "HIGHREV_SALES_Gross profit"
FROM
    "HIGHREV"."PRODUCTS",
    "HIGHREV"."SALES"
WHERE
    "HIGHREV"."PRODUCTS"."PRODUCT_NUMBER" = "HIGHREV"."SALES"."Product number"
GROUP BY 
    "HIGHREV"."PRODUCTS"."PRODUCT_NUMBER",
    "HIGHREV"."PRODUCTS"."PRODUCT",
    "HIGHREV"."PRODUCTS"."PRODUCT_TYPE",
    "HIGHREV"."SALES"."REVENUE",
    "HIGHREV"."SALES"."Gross profit"
```

Click the `Run all` button to execute the SQL.

![cpd-dv-sql-run-all](images/cpd-dv-sql-run-all.png)

### New view

To see our new view, navigate back to the `My virtualized data` panel. You should now see the new view.

![cpd-dv-new-view](images/cpd-dv-new-view.png)

Use the action menu for the view to select the `Preview` option.

<!-- TODO: screen shots of new view. I got CPD errors -->

## 8. Build Cognos Analytics dashboard to visualize merged data

<<TODO: Cognos section is WIP>>

To use Cognos Analytics, we will need to connect to `Big SQL` to access the Hive data on our Cloudera cluster. Grab the port number for our `Big SQL` instance from the Red Hat OpenShift console:

<<TODO: add screen shot. And how do we connect to OpenShift?>>
PORT: 32052

### Launch Cognos Analytics instance

From the CPD `Instances` panel, locate the Cognos Analytics service, then click on `Open` from the action menu.

![cpd-cognos-open](images/cpd-cognos-open.png)

This will bring up the Cognos Analytics service home page.

![cpd-cognos-home](images/cpd-cognos-home.png)

### Add Big SQL connection

To navigate to the `Big SQL` connection panel, start by clicking on the `Manage` option from the main menu. From the `Data server connections` panel, clicking the `+` button to create a new connection.

From the `Type` list, select `IBM Big SQL`.

![cpd-cognos-config-connection](images/cpd-cognos-config-connection.png)

Update the fields to match your `Big SQL` service connection, and click `Test` to try it out.

Note that for our setup, we have SSL enabled and need to pass in our `user ID and password`.

![cpd-cognos-connection-params](images/cpd-cognos-connection-params.png)
jdbc:db2://bigsql-1623808667998103-jdbc-tls.cp4d.svc.cluster.local:32052/bigsql

If successful, change the name of the connection and click `Save`.

Set up URL to connect to data virtualization service on CPD

You will need to enter uname/pwd to complete

Load schema and metadata

### Create data module

Use data source just added

Select table that we want: HIGHEST_REVENUE_PRODUCTS

Save data module

### Build dashboard

Drag SALES_REVENUE
Add PRODUCT_TYPE

Change scatter plot to tree map

<!--
## Associated Videos

<<TODO: REMOVE before release>>

1. Set up Active Directory Server on Windows 2019 Server
    - needed for Cloudera Data Platform
    - services include Active Directory Domain Services and DNS server
    - uses private DNS domain (cdplab.local)
    - add DNS entries for each VM in CDP cluster (cid-vm-01.cpdlab.local to cid-vm-08)
2. Use Ansible playbooks to create Cloudera Private Cloud on VMs created in IBM Cloud
3. Setup edge nodes (07 & 08) so we can install Hadoop Execution Engine (HEE)
4. Configure Cloudera so we can install HEE on edge nodes
    - need to configure HIVE because HEE install calls the HIVE server
    - use RPM to yum install HEE on edge nodes
    - use CPD Platform configuration tab to integrate with edge nodes
5. Configure CPD Platform connections via HEE
    - for HDFS (requires webHDFS url, SSL cert for HEE). Named `hdfs-hee`.
    - for Hive (requires Hive JDBC41 driver, Hadoop Integration Service url, SSL cers for HEE and Hive server). Named `hive-hee`.
    - for Impala (requires Impala JDBC41 driver, link to Impala daemon running on Cloudera). Named `impala-hee`.
6. IBM Db2 on CPD
    - upload data (product data in csv)
    - create a Db2 connection (port 31293). Requires SSL cert.
    - connection is named `db2-on-cpd`
7. Hive
    - connect to Hive using command line
    - use Hive on Tez service found on Cloudera
    - data will be on Cloudera
    - uses `beeline` CLI to create database, create table and load data from csv
8. Use BigSQL to synchronize data from Hive table into CPD
    - BigSQL in installed on CPD - named `Db2-Big-SQL-2`
    - To use virtualization, need to create a JDBC connection (port 443)
    - connection named `bigsql`
9. Use CPD Data Virtualization
    - "add existing connection" to point to connectors `bigsql` and `db2-on-cpd`
    - virtualize tables:
      - PRODUCTS (bigsql) (rename to PRODUCTS_HIVE to avoid conflict)
      - PRODUCTS (db2-on-cpd)
    - Join the 2 tables by mapping PRODUCT name
10. Cognos integration

-->

## License

This code pattern is licensed under the Apache Software License, Version 2.  Separate third party code objects invoked within this code pattern are licensed by their respective providers pursuant to their own separate licenses. Contributions are subject to the [Developer Certificate of Origin, Version 1.1 (DCO)](https://developercertificate.org/) and the [Apache Software License, Version 2](https://www.apache.org/licenses/LICENSE-2.0.txt).

[Apache Software License (ASL) FAQ](https://www.apache.org/foundation/license-faq.html#WhatDoesItMEAN)
