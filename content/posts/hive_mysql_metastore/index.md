---
title: "Installing and configuring the HIVE metastore with a MySQL backend."
date: "2023-09-22"
tags: ['Computing','Clusters','Discussion','HIVE','Spark','SQL', 'Database']
description: "A guide to configuring Hive Metastore to use a MySQL server as the backend RDBMS for metadata storage, while enabling Spark to connect to the Metastore."
author: ["Naveen Kannan"]
weight: 10
draft: false
ShowToc: true
TocOpen: true
cover:
  image: "mysql_logo.svg"
  alt: "<alt text>"
  caption: "<text>"
  relative: false
  hidden: false
  hiddenInList: true
  hiddenInSingle: false
params:
  comments: true
  ShowCodeCopyButtons: true
  ShowReadingTime: true
---

# Hive Metastore

Hive, a powerful data warehousing system built on top of Hadoop, relies on a component known as the metastore to efficiently manage metadata about the data stored within it. This metadata is crucial for organizing, querying, and processing data in Hive. In this blog post, we'll explore the role of the Hive Metastore and the significance of selecting the right relational database management system (RDBMS) for it.

## Metadata Management

The Hive Metastore serves as a central repository for storing and managing metadata related to datasets within Hive. This metadata includes essential information such as:

- Table schemas

- Data types

- Column names

- Physical data file locations

Without the metastore, Hive would struggle to efficiently keep track of these critical metadata elements.

## Schema Definition

In Hive, tables are defined using a schema that specifies column names and their associated data types. This schema is fundamental for querying and processing data correctly. The metastore takes on the responsibility of storing and managing these table schemas, offering users the flexibility to:

- Create new tables

- Modify existing schemas

- Drop tables that are no longer needed

## Metadata Scaling

In large-scale data environments, metadata can become extensive. Fortunately, the Hive Metastore can scale horizontally to handle substantial volumes of metadata efficiently. Additionally, it can be configured for high availability, ensuring continuous access to metadata even in the face of hardware failures or other issues.

# Metastore and the Relational Database Management System (RDBMS)

By default, Hive uses **Apache Derby** as the RDBMS for its metastore. Derby is a lightweight, open-source, Java-based RDBMS, and it serves as the default choice for small to medium-scale Hive installations, testing, and development purposes. Notably, Derby comes bundled with Hive, simplifying the initial setup

{{< figure
    src="apache_derby_logo.png"
    caption="The Apache Derby Logo."
    align=center
>}}

While Derby is suitable for certain use cases, in this blog post, we will use MySQL as the RDBMS for Hive's metastore for various reasons, such as:

- Scalability: MySQL excels in handling larger and more complex workloads, making it ideal for scaling Hive in data-rich environments.

- Performance: MySQL often delivers superior performance for complex queries and large datasets, making it well-suited for high-performance scenarios.

- Backup and Recovery: Advanced backup and recovery options in MySQL enhance data integrity and reliability.

- Enterprise Support: Organizations familiar with MySQL can leverage existing expertise for Hive's metadata management.

- Integration: MySQL's compatibility with various tools simplifies data integration and management processes.

- Community and Documentation: A large user community and extensive documentation resources make MySQL a dependable choice for Hive's metadata management.

# Prerequisites

The following prerequisites will be needed prior to installing MySQL and altering Hive's configuration.

- A properly configured Hadoop, YARN, Spark, and HDFS cluster.

- An installation of Hive.

# Setting up MySQL

My head node runs on RHEL 8. This guide will be for RHEL 8 Linux, but the general overview should be adaptable to other Unix-like systems.

## Download the MySQL Repo onto the node

The yum repo for MySQL needs to be downloaded to the system. The appropriate repo for your system can be found [here.](https://dev.mysql.com/downloads/repo/yum/)

## Install the repo using `yum`

I downloaded the repo to `/opt/mysql80-community-release-el8-8.noarch.rpm` in my file directory.

Install the contents of the repo using

```bash
yum localinstall /opt/mysql80-community-release-el8-8.noarch.rpm -y
```
## Disabling the local `mysql` module

Disable the local `mysql` module to prevent conflicts with the installation of the `mysql-community-server` module.

```bash
yum -y module disable mysql
```
## Installing `mysql-community-server`

Install the Community Server version of MySQL using:

```bash
yum -y install mysql-community-server
```
## Installing `mysql-connector-java`

`mysql-connector-java`, or the MySQL JDBC (Java Database Connectivity) driver, is a Java-based library that provides connectivity between Java applications and the MySQL database management system. It allows Java programs to interact with MySQL databases by facilitating the exchange of data and SQL queries.

Install it by running

```bash
yum -y install mysql-connector-java
```
## Initializing the `mysql` service

Initialize MySQL by running:

```bash
service mysqld start

```
This will initialize the MySQL service. However, this installation of MySQL needs to be secured!

# Securing MySQL and initializing the server

## Running `mysql_secure_installation`

MySQL version 8.0 or higher generates a temporary random password in `/var/log/mysqld.log` after installation.

Find your random password by running

```bash
grep 'A temporary password' /var/log/mysqld.log |tail -1 |awk '{split($0,a,": "); print a[2]}'
```
Once you have that, run `mysql_secure_installation`.

This script will guide you through setting up a root password, removing anonymous users, disallowing remote root login, and other security-related configurations.

## Adding new users and setting up the metastore

Initialize MySQL by running

```bash
mysql -u root -p
```
and enter your password.

Run the following:

```
mysql> CREATE DATABASE metastore;
Query OK, 1 row affected (0.02 sec)
mysql> use metastore;
Database changed
mysql> create user 'hive'@'localhost' identified by '<root_password>';
Query OK, 0 rows affected (0.02 sec)
mysql> grant all PRIVILEGES on *.* to 'hive'@'localhost' with grant option;
Query OK, 0 rows affected (0.02 sec)
mysql> create user 'hdfs'@'localhost' identified by '<root_password>';
Query OK, 0 rows affected (0.04 sec)
mysql> grant all PRIVILEGES on *.* to 'hdfs'@'localhost' with grant option;
Query OK, 0 rows affected (0.03 sec)
mysql> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.02 sec)
mysql> exit
Bye
```

Replace `<root_password>` with your actual password. This will create the metastore and also create the users `hive` and `hdfs` and grant them all necessary privileges to work within the metastore and it's data.


### Bonus! Automate this!

If you've been following my posts, you'll know that I've been using Ansible to automate this process. The above command is interactive! How do we get around this? If you're deploying MySQL for production level systems, then this might be an unnecessary expenditure of time.

In this section, I've added a bash script that automates the process of securing MySQL, adding new users and creating the metastore database.

```bash
#!/bin/bash
# Obtaining the  Temporary Password
root_temp_pass=$(grep 'A temporary password' /var/log/mysqld.log |tail -1 |awk '{split($0,a,": "); print a[2]}')
echo "root_temp_pass:"$root_temp_pass

# Creating a mysql_secure_installation.sql script to secure MySQL
cat > mysql_secure_installation.sql << EOF
# Ensuring that the server cannot be accessed without a password
# The password will be the cluster password
# Adding a user 'hdfs' and a user 'hive'
ALTER USER 'root'@'localhost' IDENTIFIED BY 'root_password';
CREATE USER 'hive'@'localhost' IDENTIFIED BY 'root_password';
CREATE USER 'hdfs'@'localhost' IDENTIFIED BY 'root_password';
# Removing the anonymous users
DELETE FROM mysql.user WHERE User='';
# Disallowing remote login for root
DELETE FROM mysql.user WHERE User='root' AND Host NOT IN ('localhost', '127.0.0.1', '::1');
# Removing the demo database
DROP DATABASE IF EXISTS test;
DELETE FROM mysql.db WHERE Db='test' OR Db='test\\_%';
# Make our changes take effect
FLUSH PRIVILEGES;
EOF

# Running the SQL script to secure the installation of MySQL
mysql -uroot -p"$root_temp_pass" --connect-expired-password <mysql_secure_installation.sql

# Creating another script to create the metastore and grant privileges to the hive, hdfs and root users for it
cat > mysql_privilege_grant.sql << EOF
# Granting privileges to the users hive, hdfs and root
CREATE DATABASE metastore;
use metastore;
grant all PRIVILEGES on *.* to "hive"@"localhost" with grant option;
grant all PRIVILEGES on *.* to "hdfs"@"localhost" with grant option;
grant all PRIVILEGES on *.* to "root"@"localhost" with grant option;
FLUSH PRIVILEGES;
EOF

# Running the SQL script to create the metastore and grant priviliges to the root, hive and hdfs users for the metastore
mysql -u root -p"root_password" --connect-expired-password <mysql_privilege_grant.sql
```

Replace `root_password` with your actual password and run this script. **Remember to delete this script and the ones created by this script once it finishes running!**

# Linking MySQL and Hive (and Spark!)

## Deleting a duplicate library

We will need to remove the `log4j-slf4j-impl-2.17.1.jar` file since MySQL has the same library, and the presence of multiple duplicate libraries in the `CLASSPATH` may cause Hive to fail.

Change directories to your Hive installation, and then enter the `lib` directory, and delete the `log4j-slf4j-impl-2.17.1.jar` file.

## Adding the MySQL JDBC library to Hive

Copy the `mysql-connector-java.jar` file located at `/usr/share/java/mysql-connector-java.jar` to the `lib` directory of your Hive installation. This ensures that Hive can utilize the MySQL JDBC library to connect to MySQL databases seamlessly.

For example,

```bash
cp /usr/share/java/mysql-connector-java.jar /opt/apache-hive-3.1.3-bin/lib
```

## Adding the MySQL JDBC library to Spark

Copy the `mysql-connector-java.jar` file located at `/usr/share/java/mysql-connector-java.jar` to the `jars` directory of your Spark installation.

This will enable Spark to also utilize the MySQL JDBC library to connect to MySQL databases seamlessly.

For example,

```bash
cp /usr/share/java/mysql-connector-java.jar /opt/spark-3.3.2-bin-hadoop3/jars
```

## Creating a `hive-site.xml` file

Create a file named `hive-site.xml` in the `conf` directory of your Hive installation with the following content:

```xml
<configuration>
  <property>
    <name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:mysql://localhost/metastore?createDatabaseIfNotExist=true</value>
    <description>URL for establishing a connection to the Hive Metastore database.</description>
  </property>
  <property>
    <name>javax.jdo.option.ConnectionDriverName</name>
    <value>com.mysql.jdbc.Driver</value>
    <description>Fully qualified class name of the JDBC driver used for connecting Hive and MySQL.</description>
  </property>
  <property>
    <name>javax.jdo.option.ConnectionUserName</name>
    <value>hive</value>
    <description>User name used to connect to the MySQL database for Hive Metastore.</description>
  </property>
  <property>
    <name>javax.jdo.option.ConnectionPassword</name>
    <value>root_password</value>
    <description>Password for authenticating and connecting to the MySQL database.</description>
  </property>
  <property>
    <name>datanucleus.autoCreateSchema</name>
    <description>
      When set to 'true,' this option allows automatic creation of the schema (database structure)
      if it doesn't already exist when Hive Metastore initializes.
    </description>
    <value>true</value>
  </property>
  <property>
    <name>datanucleus.fixedDatastore</name>
    <value>true</value>
  </property>
  <property>
    <name>datanucleus.autoCreateTables</name>
    <description>
      When set to 'true,' it indicates that the datastore (MySQL database) structure is fixed
      and should not be altered automatically by Hive Metastore.
    </description>
    <value>True</value>
  </property>
</configuration>

```
## Editing the `spark-env.sh` file

Add the following line to your `spark-env.sh` file located in the `conf` directory of your Spark installation.

```bash
export SPARK_CLASSPATH=/opt/apache-hive-3.1.3-bin/jars/mysql-connector-java.jar
```

## Editing the `spark-defaults.conf` file

Add the following line to your `spark-defaults.conf` file located in the `conf` directory of your Spark installation.

```bash
spark.sql.warehouse.dir=hdfs://localhost:9000/user/hive/warehouse
```

## Restarting the MySQL service

Run the following commands to restart the MySQL service:

```bash
service mysqld stop
service mysqld start
```
## Initializing the Metastore

Run the following commands:

```bash
hdfs dfs -mkdir /tmp
hdfs dfs -mkdir /user
hdfs dfs -mkdir /user/hive
hdfs dfs -mkdir /user/hive/warehouse
hdfs dfs -chmod g+w /tmp
hdfs dfs -chmod g+w /user/hive/warehouse
schematool -dbType mysql -initSchema
```
This will initialize the Hive Metastore.

This should successfully have Hive configured to use the MySQL database as the backend RDBMS for metadata storage!

# Testing

Now we will load a table into Hive and see if it is present where it should be.

Download the data files from MovieLens 100k on the [GroupLens datasets page](http://grouplens.org/datasets/movielens/) (which also has a README.txt file and index of unzipped files):

```bash
wget http://files.grouplens.org/datasets/movielens/ml-100k.zip
```
Unzip the files

```bash
unzip ml-100k.zip
```

Initialize the Hive server.

Then, create a table by running the following:

```sql
CREATE TABLE u_data (
  userid INT,
  movieid INT,
  rating INT,
  unixtime STRING)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY '\t'
STORED AS TEXTFILE;
```
And load `u.data` into the table that was just created:

```sql
LOAD DATA LOCAL INPATH '<path>/ml-100k//u.data'
OVERWRITE INTO TABLE u_data;
```
Replace `<path>` with your path to the `ml-100k` folder.

Count the number of rows in table u_data.

```sql
SELECT COUNT(*) FROM u_data;
```

Enter the `mysql` shell.

```bash
mysql -u hive -p
```
and enter your password.

Run the following in the command line for MySQL:

```sql
use metastore;
```

Followed by

```sql
show tables;
```

This will be your output:

```
+-------------------------------+
| Tables_in_metastore           |
+-------------------------------+
| AUX_TABLE                     |
| BUCKETING_COLS                |
| CDS                           |
| COLUMNS_V2                    |
| COMPACTION_QUEUE              |
| COMPLETED_COMPACTIONS         |
| COMPLETED_TXN_COMPONENTS      |
| CTLGS                         |
| DATABASE_PARAMS               |
| DBS                           |
| DB_PRIVS                      |
| DELEGATION_TOKENS             |
| FUNCS                         |
| FUNC_RU                       |
| GLOBAL_PRIVS                  |
| HIVE_LOCKS                    |
| IDXS                          |
| INDEX_PARAMS                  |
| I_SCHEMA                      |
| KEY_CONSTRAINTS               |
| MASTER_KEYS                   |
| MATERIALIZATION_REBUILD_LOCKS |
| METASTORE_DB_PROPERTIES       |
| MIN_HISTORY_LEVEL             |
| MV_CREATION_METADATA          |
| MV_TABLES_USED                |
| NEXT_COMPACTION_QUEUE_ID      |
| NEXT_LOCK_ID                  |
| NEXT_TXN_ID                   |
| NEXT_WRITE_ID                 |
| NOTIFICATION_LOG              |
| NOTIFICATION_SEQUENCE         |
| NUCLEUS_TABLES                |
| PARTITIONS                    |
| PARTITION_EVENTS              |
| PARTITION_KEYS                |
| PARTITION_KEY_VALS            |
| PARTITION_PARAMS              |
| PART_COL_PRIVS                |
| PART_COL_STATS                |
| PART_PRIVS                    |
| REPL_TXN_MAP                  |
| ROLES                         |
| ROLE_MAP                      |
| RUNTIME_STATS                 |
| SCHEMA_VERSION                |
| SDS                           |
| SD_PARAMS                     |
| SEQUENCE_TABLE                |
| SERDES                        |
| SERDE_PARAMS                  |
| SKEWED_COL_NAMES              |
| SKEWED_COL_VALUE_LOC_MAP      |
| SKEWED_STRING_LIST            |
| SKEWED_STRING_LIST_VALUES     |
| SKEWED_VALUES                 |
| SORT_COLS                     |
| TABLE_PARAMS                  |
| TAB_COL_STATS                 |
| TBLS                          |
| TBL_COL_PRIVS                 |
| TBL_PRIVS                     |
| TXNS                          |
| TXN_COMPONENTS                |
| TXN_TO_WRITE_ID               |
| TYPES                         |
| TYPE_FIELDS                   |
| VERSION                       |
| WM_MAPPING                    |
| WM_POOL                       |
| WM_POOL_TO_TRIGGER            |
| WM_RESOURCEPLAN               |
| WM_TRIGGER                    |
| WRITE_SET                     |
+-------------------------------+
74 rows in set (0.01 sec)
```
The created table should be located in `TBLS`.

Run the following command:

```sql
select * from TBLS;
```

This is what you should see:

```
+--------+-------------+-------+------------------+-------+------------+-----------+-------+----------------------------------------+---------------+--------------------+--------------------+----------------------------------------+
| TBL_ID | CREATE_TIME | DB_ID | LAST_ACCESS_TIME | OWNER | OWNER_TYPE | RETENTION | SD_ID | TBL_NAME                               | TBL_TYPE      | VIEW_EXPANDED_TEXT | VIEW_ORIGINAL_TEXT | IS_REWRITE_ENABLED                     |
+--------+-------------+-------+------------------+-------+------------+-----------+-------+----------------------------------------+---------------+--------------------+--------------------+----------------------------------------+
|      1 |  1695343066 |     1 |                0 | root  | USER       |         0 |     1 | u_data                                 | MANAGED_TABLE | NULL               | NULL               | 0x00
+--------+-------------+-------+------------------+-------+------------+-----------+-------+----------------------------------------+---------------+--------------------+--------------------+----------------------------------------+


```

# Further Reading

-   [Hive Basics](https://cwiki.apache.org/confluence/display/Hive/Tutorial#Tutorial-Concepts)
-   [Apache Derby](https://db.apache.org/derby/)
-   [MySQL](https://www.mysql.com/)
