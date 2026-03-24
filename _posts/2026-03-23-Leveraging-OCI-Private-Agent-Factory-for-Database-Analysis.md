
Data Analysis Agent: Technical Overview
--

A Data Analysis Agent is an AI-powered autonomous agent engineered to interact directly with enterprise databases. Leveraging Large Language Models (LLMs), it comprehends database schemas, interprets structured data, and generates actionable insights, explanations, and visualizations automatically.

In essence, users can query their databases in natural language and receive answers, charts, and analytic summaries without writing SQL.

Operational Workflow
--
+ Natural Language Understanding:
  The agent interprets user queries and identifies the analytical intent.
+ Schema Awareness & SQL Translation:
  By understanding the underlying database schema, the agent converts questions into optimized, safe SQL queries executed against the target database.
+ Insight Generation:
  Query results are processed using LLM reasoning to generate human-readable explanations, identifying trends, anomalies, and correlations.
+ Automatic Visualization:
  When applicable, the agent generates charts, tables, and dashboards automatically to represent query outputs visually.
+ Semantic Question Refinement:
  The agent performs question expansion and variation analysis to ensure queries are contextually accurate and data-relevant.
+ Enterprise Integration:
  Designed to connect directly to Oracle Database 19c and above, the agent ensures secure, real-time access to enterprise data.
+ Key Capabilities
- LLM-powered explanations: Translates raw query outputs into meaningful insights.
- Automatic chart and table generation: No manual visualization setup required.
- Semantic query expansion: Enhances user questions for comprehensive analysis.
- Direct database connectivity: Fully compatible with Oracle Database 19c+ environments.

In this blog, we will explore how to use the Data Analysis Agent, create a data source, analyze data from the data source objects, and generate meaningful reports.

1. Create Datasource
   
   Log in to OCI AI Database Private Agent Factory and create a new data source by selecting 'Database' as the source type. Provide the required connection details using the connection string of the target database for which reports will be generated.

 ![Apex](/images/dataanalysysagent/DataSource01.png)

2. Verify status is Connected

Ensure the data source is created successfully and its status displays as 'Connected'.

![Apex](/images/dataanalysysagent/success_datasource02.png)

3. Create Data Analysis Agent

  After successfully creating the database data source, the next step is to create a Data Analysis Agent. 

  ![Apex](/images/dataanalysysagent/DataAnalysis_agent03.png)


4. Select tables/views

   Go to ‘Data Analysis Agent’, select the previously created data source, and choose the table or view that will be used for data analysis and report generation.

   ![Apex](/images/dataanalysysagent/Createagent_04.png)

   Note: REPORTSADMIN.ASH_REPORT_AGENT_T is created from an extract of DBA_HIST_ACTIVE_SESS_HISTORY. 

6. Publish Agent

   ![Apex](/images/dataanalysysagent/PublishAgent06.png)

7. Open Agent 

   ![Apex](/images/dataanalysysagent/openagent_08.png)   

8. Generate reports and insights   

   Now let us execute the exploration for the agent to analyze the data and generate insights.

   ![Apex](/images/dataanalysysagent/Exploration_09.png)

   Next, we will request the agent to analyze wait events between 10:00 AM and 11:00 AM.
   ![Apex](/images/dataanalysysagent/Message_10.png)
   ![Apex](/images/dataanalysysagent/Graphoutput_11.png)
   ![Apex](/images/dataanalysysagent/Sql_12.png)

   As shown, the agent generated the SQL query, a graphical visualization, and a summary message explaining the findings from the data.
   


 
