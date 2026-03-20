## Introduction

Oracle Corporation AI Database Private Agent Factory (Agent Factory) is a no-code platform that enables users to rapidly build, test, and deploy intelligent AI agents without writing any code. It empowers organizations to securely connect to enterprise data sources and create smart assistants using pre-built agents, customizable workflows, and reusable templates.

With built-in capabilities such as the Knowledge Agent and Data Analysis Agent, along with support for multiple large language model (LLM) providers and enterprise-grade security, Agent Factory simplifies the development of scalable and governed AI solutions—often within minutes.

While Agent Factory can be provisioned on local environments such as Linux or macOS, this blog focuses on deploying it through the Oracle Cloud Infrastructure Marketplace for a streamlined and fully managed setup experience.

In this blog, I will walk you through the step-by-step process of installing OCI AI Private Agent Factory from the Marketplace and demonstrate its usage by testing a sample template from the template gallery.

Setting Up Agent Factory Resources from the OCI Marketplace
-----

1. Select the AI Database Private Agent Factory from OCI Marketplace and click on "Launch Stack"

 ![Apex](/images/aiagentfactory/marketplace01.png)

2. Proceed with the default version: 

![Apex](/images/aiagentfactory/launchstack02.png)

3. Select the compartment where you want to host the Agent Factory VM.

![Apex](/images/aiagentfactory/config03.png)

4. Choose the region, VCN, and subnet for your Agent Factory VM. (For this POC, we are using a public subnet—not recommended for production.)

![Apex](/images/aiagentfactory/launch04.png)

5. Choose the appropriate VM shape and size for your Agent Factory instance, and upload your SSH public key for secure access.

![Apex](/images/aiagentfactory/agentvm05.png)

6. Launch the deployment job and check the Terraform logs to confirm that all resources were created successfully.

![Apex](/images/aiagentfactory/successful06.png)

Check the Terraform logs to confirm the Agent Factory VM, app catalog subscription, and agreements were created, and note the private (172.*.*.*) and public (40.*.*.*) IPs along with the Agent Factory URL.

```sh
2026/03/19 13:56:25[TERRAFORM_CONSOLE] [INFO] Apply complete! Resources: 3 added, 0 changed, 0 destroyed.
2026/03/19 13:56:25[TERRAFORM_CONSOLE] [INFO] 
2026/03/19 13:56:25[TERRAFORM_CONSOLE] [INFO] Outputs:
2026/03/19 13:56:25[TERRAFORM_CONSOLE] [INFO] 
2026/03/19 13:56:25[TERRAFORM_CONSOLE] [INFO] Agent_Factory_URL = "https://40.*.*.28:8080/studio/installation"
2026/03/19 13:56:25[TERRAFORM_CONSOLE] [INFO] agent_factory_server_all_private_ips = {
2026/03/19 13:56:25[TERRAFORM_CONSOLE] [INFO]   "AgentFactoryVM" = "172.*.*.*"
2026/03/19 13:56:25[TERRAFORM_CONSOLE] [INFO] }
2026/03/19 13:56:25[TERRAFORM_CONSOLE] [INFO] agent_factory_server_compute_linux_instances = {
2026/03/19 13:56:25[TERRAFORM_CONSOLE] [INFO]   "AgentFactoryVM" = {
2026/03/19 13:56:25[TERRAFORM_CONSOLE] [INFO]     "id" = "ocid1.instance.oc1.dummyociddumyociddummyociddummyocid"
2026/03/19 13:56:25[TERRAFORM_CONSOLE] [INFO]     "ip" = "40.*.*.*"
2026/03/19 13:56:25[TERRAFORM_CONSOLE] [INFO]   }
2026/03/19 13:56:25[TERRAFORM_CONSOLE] [INFO] }
2026/03/19 13:56:25[TERRAFORM_CONSOLE] [INFO] agent_factory_server_public_ip = {
2026/03/19 13:56:25[TERRAFORM_CONSOLE] [INFO]   "AgentFactoryVM" = "40.*.*.*"
2026/03/19 13:56:25[TERRAFORM_CONSOLE] [INFO] }
2026/03/19 13:56:25[TERRAFORM_CONSOLE] [INFO] 
```

Copy the Agent_Factory_URL from the Terraform output, as it will be used to connect to the 26AI database and complete the installation.

Configuring the Agent Factory Application
----

1. Launch the Agent factory URL

![Apex](/images/aiagentfactory/browser07.png)

Click on proceed to website

2. Set up Access

![Apex](/images/aiagentfactory/loginpage08.png)

Save the entered credentials securely, as they will be required to log in to the Agent Factory dashboard.

3. Database Preparation for Backend

As observed from the deployment stack, no DBCS or database resources are provisioned as part of this setup. If required, ensure that an Oracle Database 26ai instance is created on OCI or any other supported host before proceeding.

During initial testing, a 19c DBCS instance was used; however, it resulted in errors related to constraints and an older Python version on the host. Therefore, it is strongly recommended to use Oracle Database 26ai for compatibility and a successful setup.

What Happens Inside the Database?

When the Agent Factory connects to the database, it performs the following actions:

+ Schema Creation: Creates required schemas, tables, and metadata structures to store agent configurations.
+ Metadata Storage: Stores agent definitions, workflows, templates, and execution details.
+ Vector Storage: Enables storage of embeddings for AI use cases such as semantic search and retrieval.
+ User & Access Setup: Configures necessary database users, roles, and privileges.
+ Integration Setup: Prepares the database for interaction with AI services and external data sources.
+ Prerequisite: Create Dedicated Database User

Execute the following SQL statements to create a dedicated user for Agent Factory:

```sql
CREATE USER mypriv_agent01 IDENTIFIED BY <DB_PASSWORD> DEFAULT TABLESPACE USERS QUOTA unlimited ON USERS;
GRANT CONNECT, RESOURCE, CREATE TABLE, CREATE SYNONYM, CREATE DATABASE LINK, CREATE ANY INDEX, INSERT ANY TABLE, CREATE SEQUENCE, CREATE TRIGGER, CREATE USER, DROP USER TO mypriv_agent01;
GRANT CREATE SESSION TO mypriv_agent01 WITH ADMIN OPTION;
GRANT READ, WRITE ON DIRECTORY DATA_PUMP_DIR TO mypriv_agent01;
GRANT SELECT ON SYS.V_$PARAMETER TO mypriv_agent01;
```

This setup ensures that the database acts as the backend for managing agents, storing context, and enabling AI-driven operations.
   
4. Provide the 26AI database connection string details to connect the Agent Factory application to your database.

![Apex](/images/aiagentfactory/connstring08.png)

Once the database connection is successfully validated, proceed to the next step in the setup.

5. Click Install after a successful setup, then proceed with the LLM configuration.

![Apex](/images/aiagentfactory/successfulinstall09.png)



LLMConfig
--
Purpose: Used when your agent needs to connect to an external LLM (Large Language Model) service.
-
Typical Use Cases:
-
+ Connecting to OCI’s AI services (like Oracle Generative AI).
+ Connecting to other external LLM providers (OpenAI, Anthropic, etc.) via API.

What it contains:
-  
+ Model ID (the specific LLM model you want to use)
+ Endpoint (the API endpoint for the LLM service)
+ API keys, credentials, or authentication info
+ Optional configurations like temperature, max tokens, etc.

When to use:
-
Whenever your agent needs dynamic natural language generation or external model inference.

6. Please provide all the requested information, including Configuration Name, Mode (With Fingerprint), Model ID, Endpoint, Compartment OCID, User OCID, Tenancy OCID, Fingerprint, Region, and upload the corresponding Private Key.
   
![Apex](/images/aiagentfactory/llmconfig10.png)

Once the test connection is successful, please save the configuration.

Note: Please ensure that the LLM Model ID you provide in the prompt is available in your region; otherwise, you may encounter the error: 'The LLM name is not in the configured LLMs for the user' when using the agent prompt.

This page provides a list of regions where OCI Generative AI models are available 

[https://docs.oracle.com/en-us/iaas/Content/generative-ai/model-endpoint-regions.htm](https://docs.oracle.com/en-us/iaas/Content/generative-ai/model-endpoint-regions.htm)

--

EmbeddedConfig
--
Purpose: Used when your agent relies on predefined rules, templates, or small models embedded directly in the agent.
-
Typical Use Cases:
-
+ Rule-based responses
+ Simple data queries from a local source
+ Agents that don’t need heavy LLM computation

What it contains:
-
+ Static prompts or rules
+ Embedded scripts or logic
+ Configuration to access local data sources if needed

When to use:
-
Lightweight agents for faster response and on-premises data access without calling an external LLM.

7. Please provide all the requested information, including Configuration Name, Mode (With Fingerprint), Model ID, Endpoint, Compartment OCID, User OCID, Tenancy OCID, Fingerprint, Region, and upload the corresponding Private Key.

![Apex](/images/aiagentfactory/embeddingconfig11.png)

Once the test connection is successful, please save the configuration.

Key Difference:
-
LLMConfig: calls an external model (more flexible, heavy-duty, needs API/key).

EmbeddedConfig: uses internal logic or small embedded models (faster, no external call, less flexible).


Running a test using a template from the Template Gallery.
--

1. This is what the OCI Private Agent Factory interface looks like:

![Apex](/images/aiagentfactory/privatefactory_dashboard12.png)

2. Click on "template gallery" and import "meeting summary item extractor."

![Apex](/images/aiagentfactory/tempelategalarry13.png)

3. Now, select the LLM to use (from the LLM configuration you set up earlier), give the workflow a name, save it, and click on 'Playground' to test.

![Apex](/images/aiagentfactory/Workfloe14.png)

4. Here’s the view of the Playground

![Apex](/images/aiagentfactory/Playground15.png)

5. Now, to test, I will paste a dummy transcript to check the summary.

![Apex](/images/aiagentfactory/Playground15.png)







