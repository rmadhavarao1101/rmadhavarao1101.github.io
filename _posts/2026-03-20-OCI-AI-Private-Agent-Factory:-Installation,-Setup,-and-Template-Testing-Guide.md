##Introduction

Oracle Corporation AI Database Private Agent Factory (Agent Factory) is a no-code platform that enables users to rapidly build, test, and deploy intelligent AI agents without writing any code. It empowers organizations to securely connect to enterprise data sources and create smart assistants using pre-built agents, customizable workflows, and reusable templates.

With built-in capabilities such as the Knowledge Agent and Data Analysis Agent, along with support for multiple large language model (LLM) providers and enterprise-grade security, Agent Factory simplifies the development of scalable and governed AI solutions—often within minutes.

While Agent Factory can be provisioned on local environments such as Linux or macOS, this blog focuses on deploying it through the Oracle Cloud Infrastructure Marketplace for a streamlined and fully managed setup experience.

In this blog, I will walk you through the step-by-step process of installing OCI AI Private Agent Factory from the Marketplace and demonstrate its usage by testing a sample template from the template gallery.

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

