
Using heatwave cluster for query acceleration in MySQL:
----

Introduction:
----

MySQL HeatWave is an in-memory query acceleration technology introduced by Oracle for the MySQL Database Service on Oracle Cloud Infrastructure. It allows users to run MySQL queries on large sets of data in real-time by utilizing massively parallel processing capabilities of modern CPUs and GPUs.

HeatWave works by automatically synchronizing data between the MySQL database and a separate in-memory cluster. The data is then cached in memory, which allows for faster query performance as it eliminates the need for disk I/O operations.

With HeatWave, MySQL users can take advantage of a highly scalable, fault-tolerant, and secure architecture that can handle complex workloads, such as data analytics, machine learning, and business intelligence. HeatWave also provides features like automatic failover, backup and restore, and real-time analytics, making it an ideal solution for businesses that require high-speed data processing capabilities.

![Apex](/images/heatwavecluster/heatwave_architecture_oci.png)

The architecture of MySQL HeatWave involves the following components:
•	MySQL Database Service: This is the MySQL database that stores the data.
•	In-memory cluster: This is a separate cluster that stores data in main memory in a hybrid columnar format. 
•	HeatWave Accelerated Engine: This is a high-performance engine that is responsible for executing queries on the in-memory data.
•	HeatWave MySQL Shell Plugin: This is a plugin that allows users to connect to the HeatWave Accelerated Engine and run queries.
•	Oracle Cloud Infrastructure Networking: This is the network infrastructure that connects the MySQL Database Service and the in-memory cluster.

When a user runs a query using the MySQL Shell, the query is sent to the HeatWave Accelerated Engine, which executes the query on the in-memory data. The results are then returned to the user through the MySQL Shell. The in-memory data is automatically synchronized with the MySQL Database Service, ensuring that the data is always up-to-date.

Overall, the architecture of MySQL HeatWave is designed to provide high-speed query performance and scalability, making it an ideal solution for businesses that require real-time analytics and data processing capabilities.

Now, let us perform a small test by creating MySQL DB in OCI, enabling Heatwave and see the query performance difference.

 
Perquisites: 
---

Login to OCI Tenancy, create your compartment for this POC, create your VCN and appropriate security rules. Also, create a compute instance in public subnet for connecting to DB in Private subnet and generate private and public key.

Set up MySQL DB and add heatwave cluster:
---

1.	Click on databases menu and click on DB Systems under MySQL.

   ![Apex](/images/heatwavecluster/createdbsystem.png)

2.	Click on “Create DB System”

   ![Apex](/images/heatwavecluster/mysql_dbsystem02.png)

3.	Select the compartment, add DB name and select Heatwave from DB options.

    ![Apex](/images/heatwavecluster/compartment_03.png)

4.  Enter username and password for MySQL DB.
  
    ![Apex](/images/heatwavecluster/userdetail04.png)


5.	 Select the VCN and the Private subnet.

    ![Apex](/images/heatwavecluster/vcn05.png)

6.	 Enable automatic backup.

    ![Apex](/images/heatwavecluster/backup06.png)

7.	Ensure we have 3306 and 33060 is open in the private subnet security rules.

    ![Apex](/images/heatwavecluster/port07.png)

8.	Note down the endpoint Private IP, which will be used to connect to MySQL DB from compute instance.

   ![Apex](/images/heatwavecluster/progress08.png)

9.	Enable Heatwave Cluster in MySQL database. Click on “More Actions” and select "Add Heatwave cluster"

   ![Apex](/images/heatwavecluster/cluster09.png)

10.	Under configure heatwave cluster, click on “estimate node” or you can manually enter the number of nodes required.

   ![Apex](/images/heatwavecluster/addnode10.png)


11.	Click on generate estimate, this will recommend the number of nodes required to process the schema data.

   ![Apex](/images/heatwavecluster/estimate11.png)


12.	Select the schema which you need to load and copy the command which can be used to manually load schema data to the heatwave cluster memory.

    ![Apex](/images/heatwavecluster/schema12.png)

13.	Now we have heat wave enabled with one node and 512 GB memory.

     ![Apex](/images/heatwavecluster/infohw.png)

    ![Apex](/images/heatwavecluster/workrequest13.png)

    Under resources click on work requests to see the progress of heatwave cluster, also click on Heatwave to see if state is active or not.

     ![Apex](/images/heatwavecluster/HWclusterinfo14.png)

    Load data to heatwave and see the performance difference:
1.	ssh to your compute instance, using your private key.

```sh

Rajesh.Madhavarao@Eclipsyss-MacBook-Pro .ssh % ssh -i ./oci-mysql-key opc@140.238.144.160                                                                                       
Activate the web console with: systemctl enable --now cockpit.socket

Last login: Fri Apr 28 09:05:25 2023 from 24.215.68.152
[opc@mysql-heatwave ~]$
[opc@mysql-heatwave ~]$

```

Note: follow below steps to download test schema dump to compute instance and from there we will upload to MySQL DB

```sh

[opc@mysql-heatwave ~]$ wget https://downloads.mysql.com/docs/airport-db.tar.gz
--2023-05-03 14:31:15--  https://downloads.mysql.com/docs/airport-db.tar.gz
Resolving downloads.mysql.com (downloads.mysql.com)... 23.66.195.54, 2600:140a:0:69b::2e31, 2600:140a:0:682::2e31
Connecting to downloads.mysql.com (downloads.mysql.com)|23.66.195.54|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 655687673 (625M) [application/x-gzip]
Saving to: ‘airport-db.tar.gz’

airport-db.tar.gz                                 100%[===========================================================================================================>] 625.31M  32.1MB/s    in 20s     

2023-05-03 14:31:36 (30.6 MB/s) - ‘airport-db.tar.gz’ saved [655687673/655687673]


[opc@mysql-heatwave ~]$ tar -xf airport-db.tar.gz

```

2.	Use MySQL Shell to connect to MySQL DB and load the dump file.

```sh
[opc@mysql-heatwave ~]$ mysqlsh root@10.0.1.124
MySQL Shell 8.0.33
[opc@mysql-heatwave ~]$ mysqlsh root@10.0.1.124
MySQL Shell 8.0.33

Copyright (c) 2016, 2023, Oracle and/or its affiliates.
Oracle is a registered trademark of Oracle Corporation and/or its affiliates.
Other names may be trademarks of their respective owners.

Type '\help' or '\?' for help; '\quit' to exit.
Creating a session to 'root@10.0.1.124'
Fetching schema names for auto-completion... Press ^C to stop.
Your MySQL connection id is 22 (X protocol)
Server version: 8.0.33-u2-cloud MySQL Enterprise - Cloud
No default schema selected; type \use <schema> to set one.
MySQL  10.0.1.124:33060+ ssl  JS >


MySQL  10.0.1.124:33060+ ssl  JS > util.loadDump("airport-db", {threads: 16, loadIndexes: "false", ignoreVersion: true,resetProgress: true})
Loading DDL and Data from 'airport-db' using 16 threads.
Opening dump...
NOTE: Dump format has version 1.0.2 and was created by an older version of MySQL Shell. If you experience problems loading it, please recreate the dump using the current version of MySQL Shell and try again.
Target is MySQL 8.0.33-u2-cloud (MySQL Database Service). Dump was produced from MySQL 8.0.26-cloud
Scanning metadata - done       
Checking for pre-existing objects...
Executing common preamble SQL
Executing DDL - done         
Executing view DDL - done       
Starting data load
8 thds loading | 100% (2.03 GB / 2.03 GB), 7.03 MB/s, 14 / 14 tables done   
Executing common postamble SQL                                           
39 chunks (59.50M rows, 2.03 GB) for 14 tables in 1 schemas were loaded in 2 min 17 sec (avg throughput 15.70 MB/s)
0 warnings were reported during the load.                                
```

Note: MySQL Shell can also be downloaded by visiting this website: https://dev.mysql.com/downloads/shell/

3.	Switch to SQL and verify the loaded schema and tables.

```sql
MySQL  10.0.1.124:33060+ ssl  JS > \sql
Switching to SQL mode... Commands end with ;
Fetching global names for auto-completion... Press ^C to stop.
MySQL  10.0.1.124:33060+ ssl  SQL > 
MySQL  10.0.1.124:33060+ ssl  SQL > show schemas;
+--------------------+
| Database           |
+--------------------+
| airportdb          |
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| world              |
+--------------------+
6 rows in set (0.0010 sec)
 MySQL  10.0.1.124:33060+ ssl  SQL > use airportdb;
Default schema set to `airportdb`.
Fetching global names, object names from `airportdb` for auto-completion... Press ^C to stop.
 MySQL  10.0.1.124:33060+ ssl  airportdb  SQL > show tables;
+---------------------+
| Tables_in_airportdb |
+---------------------+
| airline             |
| airplane            |
| airplane_type       |
| airport             |
| airport_geo         |
| airport_reachable   |
| booking             |
| employee            |
| flight              |
| flight_log          |
| flightschedule      |
| passenger           |
| passengerdetails    |
| weatherdata         |
+---------------------+
14 rows in set (0.0012 sec)
```

4.	Load data to Heatwave cluster:

   ```sql
MySQL  10.0.1.124:33060+ ssl  airportdb  SQL > CALL sys.heatwave_load(JSON_ARRAY("airportdb"),NULL);


+-------------------------------------------------------------------------------+
| LOAD SUMMARY                                                                  |
+-------------------------------------------------------------------------------+
|                                                                               |
| SCHEMA                          TABLES       TABLES      COLUMNS         LOAD |
| NAME                            LOADED       FAILED       LOADED     DURATION |
| ------                          ------       ------      -------     -------- |
| `airportdb`                         14            0          105      29.35 s |
|                                                                               |
+-------------------------------------------------------------------------------+
6 rows in set (29.5453 sec)
```

5.	Let us query few tables and see the performance difference:

   ```sql

MySQL  10.0.1.124:33060+ ssl  airportdb  SQL > explain select count(flight_id) from booking where price>300\G;
*************************** 1. row ***************************
           id: 1
  select_type: NONE
        table: NULL
   partitions: NULL
         type: NULL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: NULL
     filtered: NULL
        Extra: Using secondary engine RAPID. Use EXPLAIN FORMAT=TREE to show the plan.
1 row in set, 1 warning (0.0119 sec)
Note (code 1003): /* select#1 */ select count(`airportdb`.`booking`.`flight_id`) AS `count(flight_id)` from `airportdb`.`booking` where (`airportdb`.`booking`.`price` > 300.00)
ERROR: 1065: Query was empty
MySQL  10.0.1.124:33060+ ssl  airportdb  SQL > select count(flight_id) from booking where price>300;
+------------------+
| count(flight_id) |
+------------------+
|         21823795 |
+------------------+
1 row in set (0.0307 sec)

```
As we can see from the explain plan that the query is using secondary Heatwave engine since we have Heatwave enabled and the time taken is 0.0307 seconds.

Now let us test the same query disabling the Heatwave engine.

```sql

MySQL  10.0.1.124:33060+ ssl  airportdb  SQL > SET SESSION use_secondary_engine=off;
Query OK, 0 rows affected (0.0005 sec)

MySQL  10.0.1.124:33060+ ssl  airportdb  SQL > explain select count(flight_id) from booking where price>300\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: booking
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 52411514
     filtered: 33.32999801635742
        Extra: Using where
1 row in set, 1 warning (0.0022 sec)
Note (code 1003): /* select#1 */ select count(`airportdb`.`booking`.`flight_id`) AS `count(flight_id)` from `airportdb`.`booking` where (`airportdb`.`booking`.`price` > 300.00)
ERROR: 1065: Query was empty



MySQL  10.0.1.124:33060+ ssl  airportdb  SQL > select count(flight_id) from booking where price>300;
+------------------+
| count(flight_id) |
+------------------+
|         21823795 |
+------------------+
1 row in set (11.3129 sec)

```

Here we have disabled the heatwave engine, we can see that the explain plan is using InnoDB engine and the query is taking 11.3129 seconds, which is significantly time consuming than the query performance with the Heatwave engine.

Here are some of the ways that MySQL HeatWave helps improve query performance:

+	In-Memory Processing: MySQL HeatWave stores data in memory, which means that queries can be processed much faster than with traditional disk-based storage. This reduces the time it takes to read data from the database,  and speeds up query processing.
+	Distributed Computing: MySQL HeatWave uses a distributed architecture that allows queries to be processed in parallel across multiple nodes. This helps to reduce query response times and improves overall query performance.
+	Columnar Storage: MySQL HeatWave uses a columnar storage format that is optimized for analytic queries. This allows for faster data retrieval and processing, as only the relevant columns are accessed during a query.
+	Automatic Data Management: MySQL HeatWave automatically manages data placement and data movement across nodes to optimize query performance. This ensures that data is always available in the right place at the right time for queries to be processed efficiently.

Overall, MySQL HeatWave helps to improve query performance by leveraging in-memory processing, distributed computing, columnar storage, and automatic data management. This enables faster query processing times and more efficient use of resources, leading to improved application performance and user satisfaction.


Reference links:

[https://dev.mysql.com/doc/heatwave/en/mys-hw-architecture.html](https://dev.mysql.com/doc/heatwave/en/mys-hw-architecture.html)
[https://apexapps.oracle.com/pls/apex/r/dbpm/livelabs/run-workshop?p210_wid=878&p210_wec=&session=107375094360287](https://apexapps.oracle.com/pls/apex/r/dbpm/livelabs/run-workshop?p210_wid=878&p210_wec=&session=107375094360287)





