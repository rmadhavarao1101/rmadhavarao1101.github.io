
## GoldenGate Microservices on OCI

Introduction:
---

The GoldenGate Microservices architecture is a modern data integration solution designed for the microservices era. It provides a flexible and scalable platform for moving and processing data in real-time between different systems and data stores. The architecture is comprised of several components, including:

•	Sender Server: Sends change data from source systems to the GoldenGate Microservices architecture.
•	Receiver Server: Receives change data from the source systems and updates the target systems.
•	Service Manager: Manages the lifecycle of the microservices used to move and process data.
•	Admin Server: Provides a centralized point of control for the administration and management of the GoldenGate Microservices environment.
•	Distribution Server: Provides a centralized point for distributing and processing data.
•	Performance Metrics Server: Provides real-time performance metrics and statistics for the microservices and data integration processes.

 ![Apex](/images/GGMicroservices/architecture_01.png)


In the GoldenGate Microservices architecture, data is moved between systems using a combination of microservices and APIs, providing a flexible and scalable solution for real-time data integration. The architecture supports both batch and real-time data processing and can be used to integrate data between a wide range of systems and data stores, including relational databases, NoSQL databases, cloud-based data stores, and more.

In this blog we will discuss about setting up Goldengate microservices from OCI Marketplace. With GG Microservices, you can deploy in an off-box architecture, which means you can run and manage your Oracle GoldenGate deployment from a single location.

Deployment:

Login to your OCI tenancy
From Menu select “Marketplace”, under Marketplace select “all applications”, search for oracle goldengate and select “oracle goldengate for oracle”

 ![Apex](/images/GGMicroservices/marketplace02.png)

Figure 1: OCI Marketplace

 ![Apex](/images/GGMicroservices/gg_mplace_03.png)

Figure 2: Search for Oracle GG in Marketplace search bar

Select 21.8 Microservice edition and the target compartment where you want to deploy.

 ![Apex](/images/GGMicroservices/launchstack04.png)

 Figure 3: selecting stack and launching the same

Click on Launch Stack:

If you do not have object storage under your compartment, please go ahead and create object storage in your compartment and add your object storage name here.

 ![Apex](/images/GGMicroservices/stackinfo05.png)

 Figure 4: Stack Information

Add your VCN and subnet details:

 ![Apex](/images/GGMicroservices/config_variables06.png)

 Figure 5: Network and Configuration variables

For testing purpose you can proceed to check the public IP button which will enable access to your compute instance from the public network.

![Apex](/images/GGMicroservices/settings07.png)

Figure 6: Instance Settings

![Apex](/images/GGMicroservices/runstack08.png)

Figure 7: Run stack

![Apex](/images/GGMicroservices/jobdetail09.png)

Figure 8: Job details

![Apex](/images/GGMicroservices/successfuljob10.png)

Figure 9: Successful creation

![Apex](/images/GGMicroservices/instancedetail11.png)

Figure 10: GG Instance detail

Now the GG Microservices instance details can be found under “Compute”-> ”Instances”, and this can be accessed thru your jump server from public subnet.

Login to GG instance and make sure you have noted the default credentials for GG Microservice instance, which is present under /home/”<user>”/ogg-credentails.json.


```sh

-bash-4.2$ pwd
/home/opc
-bash-4.2$ ls
ogg-credentials.json
-bash-4.2$ cat ogg-credentials.json
{"username": "oggadmin", "credential": "**************"}
-bash-4.2$

```

Now we will login to GG Microservice portal using the URL below and enter the username and password from Json file.

http://<IP-address GG instance>:9000

![Apex](/images/GGMicroservices/servicestatus12.png)

Figure 11:

As we can see all the services are up and running.

Overall, Oracle GoldenGate Microservices provides a flexible, scalable, and cost-effective solution for real-time data integration, making it a valuable tool for organizations looking to integrate data across their systems and data stores.

In our next blog we will see how to add source, targets, replication, and extract scenarios.


P.S.

Service Manager:
--

A Service Manager acts as a watchdog for other services available with Microservices Architecture.

It manages the lifecycle of the various microservices used to move and process data. It provides a centralized point of control for deploying, starting, stopping, and monitoring microservices. The service manager also provides a unified interface for managing the configuration and settings of the various microservices, making it easier to manage large-scale, complex data integration solutions. By using the service manager, administrators can automate the deployment and management of microservices, reducing the risk of human error and increasing the overall reliability of the data integration process.

Admin Server:
--
The Admin Server provides a web-based user interface for performing administrative tasks, such as managing microservices, monitoring their performance, and configuring security settings. It also provides a REST API for programmatic access to the administration and management features, allowing developers to automate administrative tasks and integrate the GoldenGate Microservices environment with other systems and tools. The Admin Server helps simplify the management of the GoldenGate Microservices environment, reducing the risk of human error and increasing the overall reliability of the data integration process.

Distribution Server:
--
The Distribution Server is a component in the Oracle GoldenGate Microservices architecture that provides a centralized point for distributing and processing data. The Distribution Server receives change data from the GoldenGate Microservices Receiver Server, performs any necessary transformations, and distributes the data to the target systems. It can also aggregate data from multiple sources and distribute the aggregated data to multiple target systems. The Distribution Server helps improve the scalability and reliability of the data integration process by providing a centralized point for processing and distributing data. Additionally, it can help reduce the load on the target systems by performing data transformations and data aggregation before the data is stored.

Receiver Server:
--
Receiver Server is a component in the Oracle GoldenGate Microservices architecture that receives change data from source systems and updates the target systems. The receiver server works in conjunction with the GoldenGate Microservices sender server to ensure that changes are propagated in real-time from source to target systems. The receiver server is responsible for processing change data, transforming it as required, and storing it in the target system.

Performance Metrics Server:
--
The Performance Metrics Server is a component in the Oracle GoldenGate Microservices architecture that provides real-time performance metrics and statistics for the microservices and data integration processes. The Performance Metrics Server collects data from the various microservices and provides a centralized repository for performance metrics, such as data transfer rate, processing time, and error rates. The metrics data can be viewed through the Admin Server web-based user interface or accessed programmatically through the REST API. The Performance Metrics Server helps administrators and developers monitor the performance of the data integration process and identify any bottlenecks or issues that need to be addressed. It also helps improve the overall reliability and efficiency of the data integration process by providing visibility into the performance of the microservices and data integration process.

Admin Client:
--
The GoldenGate Admin Client is a command-line tool used for administering and managing the Oracle GoldenGate Microservices environment. The Admin Client provides a simple, easy-to-use interface for performing administrative tasks, such as starting and stopping microservices, monitoring their performance, and configuring security settings. It can be used to interact with the Admin Server, allowing administrators to manage the GoldenGate Microservices environment from the command line. The Admin Client can be run on the same machine as the Admin Server or on a remote machine, making it possible to manage the GoldenGate Microservices environment from anywhere with network access. The Admin Client provides a convenient way for administrators to manage the GoldenGate Microservices environment, making it an essential tool for organizations using GoldenGate Microservices for data integration.


