# Introduction
This is an end to end azure data engineering project. Services that I have used includes Azure Data factory, Azure Databricks, Azure Key Vault, Azure datalakegen2, Azure Synapse Analytics. 
I followed the steps from this youtube tutorial https://www.youtube.com/playlist?list=PLrG_BXEk3kXx6KE4nBmhf6QwSHMbznP2W to build this project

# Key Takeaways
- Familiarized myself with interfaces in Azure Data Factory, Azure Data Bricks and Azure Synapse Analytics.
- There are a lot of security procedure to follow, such as assigning role assignment to user and creating secrets in Azure Key Vaults.
- Gain great understanding of Azure Data Lake Gen 2 Storage, Azure Data Factory and Azure Synapse Analytics.
- Learned to use T-SQL and pyspark.

# Project Flow


![image](https://github.com/tekyifeng/portfolio-project-2/assets/105114292/8a9b670b-3b60-455e-8d8d-5dd970df57ed)


A database will be created in Mircosoft SQL Server, then it'll be connected to Azure Data factory, using azure data factory the data is ingested into Azure Data Lake Gen 2. After that, Databricks will be used to transform the data in Azure Lake Gen 2, then the data will be loaded into Azure Synapse Analytics and finally power bi is used to connect to the SQL Server in Synapse analytics for Data visualization and Analysis.

# Microsoft SQL Server (MSSQL Server)

A sample database called AdventureWorksLT2017 is being used in this project. Here's the link to the databases:  https://learn.microsoft.com/en-us/sql/samples/adventureworks-install-configure?view=sql-server-ver16&tabs=ssms
After downloading the file from the link, I loaded up the database in Microsoft SQL Server management studio. Then, I created a login credential to the database with the following SQL Query:

CREATE LOGIN feng WITH PASSWORD - "xxxxxxxxxx"
create user feng for login feng

After Credentials have been created, I then create 2 secrets from Azure Key Vault to accomandate the credentials I have created.

# Storage Account

A storage account with azure lake gen 2 storage capability is created. Then, 3 containers bronze, silver and gold is created.

![image](https://github.com/tekyifeng/portfolio-project-2/assets/105114292/05c1da58-3d92-4aaf-8131-d2ee6a792959)

# Azure Data Factory

In Azure Data factory, I have created the following services for data ingestion
1. Dataset - Dataset for Table from Database, Dataset for Data from Tables and Dataset to output all data
2. Linked Services -  Linked services between On premise SQL server (MSSQL Server) and data factory, Linked Service between Data factory and Azure Data Lake Gen 2 and Linked service for azure databricks

The ingestion pipeline I have created looks like

![image](https://github.com/tekyifeng/portfolio-project-2/assets/105114292/c254b8ad-5bcf-4da5-bba2-ff275cc51dae)

In the Lookup function, the following query is applied to read all the tables from the Database:

    SELECT s.name AS SchemaName, t.name  AS TableName FROM sys.tables AS t

Then using for each funtion and the following query in copy data function:

    @{concat('SELECT * FROM ' , item().SchemaName, '.' , item().TableName)} 

each table is copied into a dataset and output as parquet file in Azure Lake Gen 2 Storage for data transformation
The data that we have ingested to Azure Lake Gen 2 Storage are all in the bronze folder.

# Azure Databricks

Now we have all the data in Lake Gen 2 storage, Azure Databricks is used to transform the data. The data transformation method is following the best practices recommneded by Mircosoft, which is bronze, silver and gold layer

1. Bronze Layer - the unprocessed file (raw file)
2. Silver Layer - the validated data
3. Gold Layer - the enriched data

The following shows the pyspark code that I used to transform the data

# MOUNTING

    configs = {
      "fs.azure.account.auth.type": "CustomAccessToken",
      "fs.azure.account.custom.token.provider.class": spark.conf.get("spark.databricks.passthrough.adls.gen2.tokenProviderClassName")
    }
    
    #Optionally, you can add <directory-name> to the source URI of your mount point.
    dbutils.fs.mount(
      source = "abfss://bronze@portfolioproject2fengsa.dfs.core.windows.net/",
      mount_point = "/mnt/bronze",
      extra_configs = configs)


# BRONZE TO SILVER
#The following code changes the date type for all date columns in all Tables

        from pyspark.sql.functions import from_utc_timestamp, date_format
        from pyspark.sql.types import TimestampType
        
        for table in table_name:
            input_path = f"/mnt/bronze/SalesLT/{table}/{table}.parquet"
            df = spark.read.format('parquet').load(input_path)
            column = df.columns
        
            for col in column:
                if "Date" in col or "date" in col:
                    df = df.withColumn(col , date_format(from_utc_timestamp(df[col].cast(TimestampType()),"UTC"), "yyyy-MM-dd"))
        
            output_path = f"/mnt/silver/SalesLT/{table}/"
            df.write.format('delta').mode("overwrite").save(output_path)

# SILVER TO GOLD
#The following code changes the column name from the format ColumnName to Column_Name

        from pyspark.sql.functions import from_utc_timestamp, date_format
        from pyspark.sql.types import TimestampType
        
        table_name = []
        
        for i in dbutils.fs.ls('mnt/silver/SalesLT'):
            table_name.append(i.name.split('/')[0])
        
        #Loop through table
        for table in table_name:
            input_path = f"/mnt/silver/SalesLT/{table}/"
            df = spark.read.format('delta').load(input_path)
            column_name = df.columns
        
        #Loop through each column name to rename to Column_Name format
                for name in column_name:
                    counter_i = 0
                    column_name_list = []
                    for letter in name:
                        try:
                            if (name[counter_i + 1].isupper() or name[counter_i + 1].isdigit()) and name[counter_i].isupper() == False:
                                letter = letter + "_"
                                counter_i += 1
                                column_name_list.append(letter)
                            else:
                                counter_i += 1
                                column_name_list.append(letter)
                        except:
                            column_name_list.append(letter)
                    new_column_name = "".join(column_name_list)
            
                    df = df.withColumnRenamed(name , new_column_name)
                
                output_path = f"/mnt/gold/SalesLT/{table}/"
                df.write.format('delta').mode('overwrite').save(output_path)

In overall, the data pipeline in Azure Data Factory looks like

![image](https://github.com/tekyifeng/portfolio-project-2/assets/105114292/8e9818fd-32a7-45de-970a-a3e362b40f5b)

# Azure Synapse Analytics

In Synapse Analytics, I craeted a database called db_gold_container and then create tables using the following script

    USE db_gold_container
    GO
    
    CREATE OR ALTER PROC CreteSQLServerlessView_gold @ViewName NVARCHAR(100)
    AS
    BEGIN
    
        DECLARE @statement VARCHAR(MAX)
    
        SET @statement = N'CREATE OR ALTER VIEW ' + @ViewName + ' AS
        SELECT *
        FROM
            OPENROWSET(
                BULK ''https://portfolioproject2fengsa.dfs.core.windows.net/gold/SalesLT/' + @ViewName + '/'',
                FORMAT = ''DELTA''
            ) AS [result]
        '
    
        EXEC (@statement)
        
    END

After creating the tables, the following pipeline is created to create views in the SQL Database

![image](https://github.com/tekyifeng/portfolio-project-2/assets/105114292/ba4cb039-13f9-4e14-b90d-425885a265df)

First an integration dataset is created to read all the table names from the gold conatiner that I have created, then the get metadata function is used to read all the table names. Using for each and storage procedure function, each table view is created in the SQL database.

Then the data ready for reporting and visualization

# Data Visualization

To connect to Azure Synapse Analytics in Power BI, in the get data option, select Azure Synapse Analytics (SQL DW). Then, proceed to Azure Synapse Analytics and get the Serverless SQL endpoint link to use it to the connect to Power BI for visualization.
Please refer to the attachment for the dashboard creatd. When creating this dashboard, I used features like
1. Drill Through
2. Bookmark
3. Create Date table for Date Dimension using DAX
4. Use DAX function to create formulas



