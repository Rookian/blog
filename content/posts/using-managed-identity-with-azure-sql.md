---
title: "Using Managed Identity With Azure Sql"
date: 2021-08-07T21:48:51+02:00
draft: true
tags: ["Managed Identity", "Azure", "SQL", "Security", "Pulumi"]
---

## Objective
Typically you would access SQL Server with a SQL user and password. We all have been struggling with storing passwords securely. So let's get rid of them entirely.
Azure provides a concept of accessing Azure resources without credentials. This concept is called Managed Identity and is based on Azure AD service principal.

Our objective is too access an Azure SQL database from an Azure App Service.

![AccessSQLServer](/img/AccessSQLServer.jpg)

## Solution
You need to implement the following steps:
- Create a Managed Identity for your App Service
- Grant Managed Identity access to Azure SQL


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

**2. Grant Managed Identity access to Azure SQL**
Before we can grant access to a SQL Server, we need to create one with a database:
```csharp
var sqlServer = new Server("myServer", new ServerArgs
{
    AdministratorLogin = "myloginforsql",
    AdministratorLoginPassword = "passWord123@123",
    ResourceGroupName = resourceGroup.Name,
});
new Database("myDb", new DatabaseArgs
{
    ResourceGroupName = resourceGroup.Name,
    ServerName = sqlServer.Name,
    Sku = new Pulumi.AzureNative.Sql.Inputs.SkuArgs { Family = "Gen5", Name = "Basic", Tier = "Basic" }
});
```

![Provisioningprocess](/img/Provisioningprocess.jpg)
