
Using heatwave cluster for query acceleration in MySQL:


Introduction:

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
Login to OCI Tenancy, create your compartment for this POC, create your VCN and appropriate security rules. Also, create a compute instance in public subnet for connecting to DB in Private subnet and generate private and public key.

Set up MySQL DB and add heatwave cluster:

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

    ![Apex](/images/heatwavecluster/workrequest13.png)

