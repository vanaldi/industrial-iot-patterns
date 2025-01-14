# Overall Equipment Effectiveness(OEE) and KPI Calculation Engine

Goal of this sample is to acceleratre deployment of [Industrial IoT Transparency Patterns](https://docs.microsoft.com/en-us/azure/architecture/guide/iiot-patterns/iiot-transparency-patterns). There is no one size fits all solution, as there are many considerations, please review them before moving your workload to production.

## High Level Design

![Overall Equipment Effectiveness(OEE) and KPI Calculation Engine](../images/oee.png)

## Pre-requisites

- You have [Operational Visibility](../2_OperationalVisibility/README.md) sample working.

## Setup SQL Database


- Create a [Single SQL Database](https://learn.microsoft.com/en-us/azure/azure-sql/database/single-database-create-quickstart?view=azuresql&tabs=azure-cli)    
[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fvanaldi%2Findustrial-iot-patterns%2Fmain%2F3_OEECalculationEngine%2Fdeploy%2FsqlDB.json)

- Run the [sqldb/mes-reporting.sql](sqldb/mes-reporting.sql) script to create the MES and OEE Reporting tables, along with some sample data

## Setup Synapse Workspace

- Create a [Synapse Workspace](https://docs.microsoft.com/en-us/azure/synapse-analytics/quickstart-create-workspace) with default settings:  
    1. Create a Data Lake Storage account  
        `az storage account create --name iiotsamplestaccount --resource-group iiotsample --location westus2 --sku Standard_RAGRS --kind StorageV2`  
    1. Create a container  
        `az storage container create --name iiotsamplefs --account-name iiotsamplestaccount --auth-mode login`
    1. Create the Synapse workspace  
        `az synapse workspace create --name iiotsamplesynapsews --resource-group iiotsample --storage-account iiotsamplestaccount --file-system iiotsamplefs --sql-admin-login-user sqladminuser --sql-admin-login-password <ypur password> --location westus2`

- Create 2 [Linked Services](https://docs.microsoft.com/en-us/azure/data-factory/concepts-linked-services?tabs=data-factory) in Synapse Workspace:

    1. Download [synapse/sqlLinkedService.json](./synapse/sqlLinkedService.json) and add your SQL password
    1. Download [synapse/adxLinkedService.json](./synapse/adxLinkedService.json) and add tenantId, servicePrincipalId and servicePrincipalKey related to the Azure Data Explorer created in the prerequisites
    1. Link the [SQL Database](https://docs.microsoft.com/en-us/azure/data-factory/connector-azure-sql-database?tabs=data-factory#create-an-azure-sql-database-linked-service-using-ui) created above:  
    `az synapse linked-service create --workspace-name iiotsamplesynapse --name sqllinkedservice --file @"./sqlLinkedService.json"`
    1. Link [Azure Data Explorer](https://docs.microsoft.com/en-us/azure/data-factory/connector-azure-data-explorer?tabs=data-factory#create-a-linked-service-to-azure-data-explorer-using-ui) created in the prerequisites:  
    `az synapse linked-service create --workspace-name iiotsamplessynapse --name adxlinkedservice --file @"./adxLinkedService.json"`

- Upload new Workspace package [package/dist/manufacturingmetrics-0.1.0-py3-none-any.whl](package/dist/manufacturingmetrics-0.1.0-py3-none-any.whl)  

    <img src="../images/sparkpool-2.png"  height="60%" width="60%">

- Create a new [Apache Spark Pool](https://learn.microsoft.com/en-us/cli/azure/synapse/spark/pool?view=azure-cli-latest#az-synapse-spark-pool-update):  
    `az synapse spark pool create --name devtestspark --workspace-name iiotsamplesynapse  --resource-group iiotsample --spark-version 2.4 --node-count 3 --node-size Small`

- Upload the [package/requirements.txt](package/requirements.txt) file, select the workspace package created above and click `Apply`. Wait until the packages are deployed.

    <img src="../images/sparkpool-3.png"  height="60%" width="60%">

## Calculate OEE using Synapse Notebook

- Open `Develop` tab, click on `+` and Import the [notebook/CalculateOEE.ipynb](notebook/CalculateOEE.ipynb)

- Attach the notebook to the spark cluster created above.

- In first cell, update the values for `sqldbLinkedServiceName` and `kustolinkedServiceName` as created above

- In second cell, update the `oeeDate` to a date which has telemetry data in Data Explorer.

- Run both the cells

- Open SQL Database created above and verify the data in `OEE` table

## Visualize OEE in Power BI

- Open [powerbi/oee.pbix](powerbi/oee.pbix) file and change the `Data Source settings` to connect with the SQL Database created above.

    <img src="../images/oee-pbi-1.png"  height="70%" width="70%">

    <img src="../images/oee-pbi-2.png"  height="70%" width="70%">


## Additional Resources 

- Update and ReBuild package
    - `cd package`
    - `pip install wheel setuptools`
    - `python setup.py bdist_wheel`
