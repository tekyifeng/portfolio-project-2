This is an end to end azure data engineering project. Services that I have used includes Azure Data factory, Azure Databricks, Azure Key Vault, Azure datalakegen2, Azure Synapse Analytics. I followed the steps from this youtube tutorial https://www.youtube.com/playlist?list=PLrG_BXEk3kXx6KE4nBmhf6QwSHMbznP2W to build this project

Project Flow
This is the project flow

![image](https://github.com/tekyifeng/portfolio-project-2/assets/105114292/8a9b670b-3b60-455e-8d8d-5dd970df57ed)


A database will be created in Mircosoft SQL Server, then it'll be connected to Azure Data factory, using azure data factory the data is ingested into Azure Data Lake Gen 2. After that, Databricks will be used to transform the data in Azure Lake Gen 2, then the data will be loaded into Azure Synapse Analytics and finally power bi is used to connect to the SQL Server in Synapse analytics for Data visualization and Analysis.

Microsoft SQL Server

A sample database called AdventureWorksLT2017 is being used in this project. Here's the link to the databases:  https://learn.microsoft.com/en-us/sql/samples/adventureworks-install-configure?view=sql-server-ver16&tabs=ssms
After downloading the file from the link, I loaded up the database in Microsoft SQL Server management studio. Then, I created a login credential to the database with the following SQL Query:

CREATE LOGIN feng WITH PASSWORD - "xxxxxxxxxx"
create user feng for login feng

After Credentials have been created, I then create 2 secrets from Azure Key Vault to accomandate the credentials I have created.

Azure Data Factory

