---
title: "Using Managed Identity With Azure Sql"
date: 2021-08-07T21:48:51+02:00
draft: true
tags: ["Managed Identity", "Azure", "SQL", "Security", "Pulumi"]
---

## Objective
Typically you would access SQL Server with a SQL user and password using a connection string like this:
```
Server=tcp:myserver.database.windows.net,1433;Initial Catalog=myDb;Persist Security Info=False;User ID=myloginforsql;Password=passWord123@123;MultipleActiveResultSets=False;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;
```
We all have been struggling with storing passwords securely. So let's get rid of them entirely.
Azure provides a concept of accessing Azure resources without credentials. This concept is called Managed Identity and is based on Azure AD service principal.

Our objective is to access an Azure SQL database from an Azure App Service without any credentials.

![AccessSQLServer](/img/AccessSQLServer.jpg)

## Solution
You need to implement the following steps:
- Create a Managed Identity for your App Service
- Grant Managed Identity access to Azure SQL
- Access Azure SQL from App Service  

**1. Create a Managed Identity for your App Service**

The first step is quite easy and can be done with Pulumi
```csharp
var plan = new AppServicePlan("pulitestserviceplan", new AppServicePlanArgs
{
   Location = config.Location,
   ResourceGroupName = resourceGroup.Name,
   Kind = "Linux",
   Reserved = true,
   Sku = new SkuDescriptionArgs { Size = "B1", Tier = "Standard", Name = "B1" }
});

new WebApp("myApp", new WebAppArgs
{
    Location = config.Location,
    ResourceGroupName = resourceGroup.Name,
    // Create a System-Assigned Managed Identity
    Identity = new ManagedServiceIdentityArgs { Type = ManagedServiceIdentityType.SystemAssigned },
    ServerFarmId = plan.Id
});
```

A System-Assigned Managed Identity is an identity that is created by Azure based on the given Azure resource. It will be named exactly the same as the resource. It's lifetime is also bound to the correspondign Azure resource. You can find more about this top [here](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/overview#managed-identity-types).

**2. Grant Managed Identity access to Azure SQL**
Before we can grant access to a SQL Server, we ofc first need to create one with a database:
```csharp
var sqlServer = new Server("myServer", new ServerArgs
{
    AdministratorLogin = "myloginforsql",
    AdministratorLoginPassword = "passWord123@123", // TODO encrypt this password
    ResourceGroupName = resourceGroup.Name,
});

new Database("myDb", new DatabaseArgs
{
    ResourceGroupName = resourceGroup.Name,
    ServerName = sqlServer.Name,
    Sku = new SkuArgs { Family = "Gen5", Name = "Basic", Tier = "Basic" }
});
```

> You might wonder, why we need to define a user name and password. Well, `AdministratorLogin` and `AdministratorLoginPassword` are required in order create an Azure SQL Server. But we won't use those credentials.

Now we can register the Managed Identity from the App Service to the Azure Sql Server.

```sql
-- myApp is the name of the Managed Identity/App Service
CREATE USER [myApp] FROM EXTERNAL PROVIDER
```

Additional we can set some roles like `db_datareader`.

```sql
ALTER ROLE db_datareader ADD MEMBER [myApp] 
```

![Provisioningprocess](/img/Provisioningprocess.jpg)
