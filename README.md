# Jamstack Todo App with Azure Static Web Apps, Data API builder and Azure SQL Database

A sample Todo app built with Vue.js that uses Azure Static Web Apps, Data API builder and Azure SQL Database.

The Todo application allows to

- Create either public or private todos 
- Update or delete todos
- Mark a todo as completed
- Drag and drop to reorder todos

it uses the following features

- Backend REST API via Data API builder 
- Authentication via Easy Auth configured to use GitHub
- Authorization via Data API builder policies
- Data API builder hosted in Azure via Static Web Apps "Database Connections"

## Local development

*Fork the repository* and clone it to your local machine.

Make sure you have the .NET Core 6, Node 16 and [Static Web Apps CLI](https://azure.github.io/static-web-apps-cli/docs/intro) installed.

### Deploy the database

Deploy the database project to an SQL Server or Azure SQL Emulator instance using VS Code Database Projects extension or manually using the [`sqlpackage`](https://learn.microsoft.com/sql/tools/sqlpackage/sqlpackage-download) tool.

If you don't have a SQL Server or Azure SQL Emulator installed locally, you can use the updated `sqlcmd` tool download the latest version of SQL Server and run it withint a container. (Note: make sure you have [Docker](https://www.docker.com/products/docker-desktop/) installed on your machine first)

Install the `sqlcmd` tool, using the documentation available [here](https://learn.microsoft.com/en-us/sql/tools/sqlcmd/go-sqlcmd-utility). 

Then download the latest version of SQL Server and deploy the .dacpac:

```shell
sqlcmd create mssql --database TodoDB --use .\database\TodoDB\bin\Release\TodoDB.dacpac --context-name todo-demo
```

once download is finished, make sure the container is running:

```shell
sqlcmd start todo-demo
```

### Run the app locally

Once database has been deployed, build the Vue fronted via

```shell
swa build
```

If you haven't create an `.env` file yet, create one in the root folder, and the `AZURE_SQL_APP_USER` environment variable to set it to the connection string pointing to the local SQL Server instance or Azure SQL Emulator:

```shell
sqlcmd config connection-strings --type ADO.Net > secret & SET /p AZURE_SQL_APP_USER=<secret & del secret
```

Then start the app locally using the following command:

```shell
swa start --data-api-location ./swa-db-connections
```

## Deploy to Azure

Make sure you have the AZ CLI installed.

### Setup the Static Web App resource

Create a Resource Group if you don't have one already:

```shell
az group create -n <resource-group> -l <location>
```

then create a new Static Web App using the following command:

```shell
az staticwebapp create -n <name> -g <resource-group>
```

once it has been created, get the Static Web App deployment token:

```shell
az staticwebapp secrets list --name <name> --query "properties.apiKey" -o tsv
```

Take the token and add it to the repository secrets as `AZURE_STATIC_WEB_APPS_API_TOKEN`.

### Get the connection string to Azure SQL DB

Create a new Azure SQL Server if you don't have one already: 

```shell
az sql server create -n <server-name> -g <resource-group> -l <location> --admin-user <admin-user> --admin-password <admin-password>
```

and set yourself as the AD admin. To do that get your user object id:

```shell
az ad signed-in-user show --query objectId -o tsv
```

and get the display name:

```shell
az ad signed-in-user show --query displayName -o tsv
```

The create the AD admin in Azure SQL server:

```shell
az sql server ad-admin create --display-name <display-name> --object-id <object-id> --server <server-name> -g <resource-group>
```

Make sure that Azure Services can connect to the created Azure SQL server:

```shell
az sql server firewall-rule create -n AllowAllWindowsAzureIps -g <resource-group> --server <server-name> --start-ip-address 0.0.0.0 --end-ip-address 0.0.0.0
```

Make sure to read more details about how to configure the firewall here: [Connections from inside Azure](https://learn.microsoft.com/azure/azure-sql/database/firewall-configure?view=azuresql#connections-from-inside-azure)

Get the connection string (don't worry if the TodoDB database doesn't exist yet, it will be created later automatically):

```shell 
az sql db show-connection-string -s <server-name> -n TodoDB -c ado.net
```

Replace the `<username>` and `<password>` in the connection string with those for a user that can perform DDL (create/alter/drop) operations on the database, and then create a new secret in the repository called `AZURE_SQL_CONNECTION_STRING` with the value of the connection string.

### Push the repository to GitHub

Push the repo to GitHub to kick off the deployment.

### Configure the Static Web App Database Connection

Once the deployment has completed, navigate to the Static Web App resource in the Azure Portal and click on the [Database connection](https://learn.microsoft.com/azure/static-web-apps/database-azure-sql?tabs=bash&pivots=static-web-apps-rest) item under *Settings*. Click on *Link existing database* to connect to the Azure SQL server and the TodoDB that was created by the deployment.

You can use the sample application user that is created during the [database deployment phase](./database/TodoDB/Script.PostDeployment.sql):

- User: `todo_dab_user`
- Password: `rANd0m_PAzzw0rd!`

## Done!

Use your browser to navigate to the Static Web App URL and start using the app.
