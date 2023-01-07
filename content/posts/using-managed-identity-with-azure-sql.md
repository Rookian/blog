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

**2. Grant Managed Identity access to Azure SQL**
Before we can grant access to a SQL Server, we ofc first need to create one with a database:
```csharp
var sqlServer = new Server("myServer", new ServerArgs
{
    AdministratorLogin = "myloginforsql",
    AdministratorLoginPassword = "passWord123@123", // TODO encrypt this password
    ResourceGroupName = resourceGroup.Name,
});

var database = new Database("myDb", new DatabaseArgs
{
    ResourceGroupName = resourceGroup.Name,
    ServerName = sqlServer.Name,
    Sku = new SkuArgs { Family = "Gen5", Name = "Basic", Tier = "Basic" }
});
```

> ⚠ **Important**  `AdministratorLogin` and `AdministratorLoginPassword` are required in order create an Azure SQL Server. But we won't use those credentials.

Now we can register the Managed Identity of the App Service with the following SQL script.

```sql
-- myApp is the name of the Managed Identity/App Service
CREATE USER [myApp] FROM EXTERNAL PROVIDER
```

Additional we can set some roles like `db_datareader`.

```sql
ALTER ROLE db_datareader ADD MEMBER [myApp] 
```

![Provisioningprocess](/img/Provisioningprocess.jpg)

Running the SQL script on your **local development environment** is going to work fine whereas running it on a CI server like Azure Devops using a service principal is going to fail.
To be able to successfully execute `CREATE USER [myApp] FROM EXTERNAL PROVIDER` with a service principal you have to the following:
- Create a Managed Identity for your Azure SQL Server
- Set your deployment service principal as an administrator
- Assign the Azure SQL Server identity to Azure AD [Directory readers role](https://docs.microsoft.com/en-us/azure/active-directory/roles/permissions-reference#directory-readers). 

```csharp
var sqlServer = new Server("myServer", new ServerArgs
{
    AdministratorLogin = "myloginforsql",
    AdministratorLoginPassword = "passWord123@123", // TODO encrypt this password
    ResourceGroupName = resourceGroup.Name,

    // Create System-Assigned Managed Identity
    Identity = new ResourceIdentityArgs { Type = IdentityType.SystemAssigned },

    // Add your deployment service principal as an administrator
    Administrators = new ServerExternalAdministratorArgs
    {
        PrincipalType = PrincipalType.Application,
        Login = "Your deployment service principal name",
        Sid = "Yor deployment service principal ObjectId"
    }
});
```

As the chances are high that your service principal might not be able to assign an Azure AD principal to the Directory readers role, I would suggest to create a separate Azure Ad group by your Azure Ad administrator manually. Let your admistrator assign that group group to Directory readers and make your service principal an owner of that group. With this setup you can self-manage the given group and can add your Azure SQL Server Managed Identity. 

> ⚠ **Important** For local development your personal account should also be an owner of that group.

```csharp
new GroupMember("DirectoryReadersGroup", new GroupMemberArgs
{
    GroupObjectId = "ObjectId of your group",
    MemberObjectId = sqlServer.Identity.Apply(x => x.PrincipalId)
});
```

Now you can create a `SQLConnection` without any credentials using [Azure Active Directory Authentication](https://docs.microsoft.com/en-us/sql/connect/ado-net/sql/azure-active-directory-authentication?view=sql-server-ver16).

```csharp

public class MyStack : Stack
{
  // define some output variables in your stack
  [Output]
  public Output<string> SqlConnectionString { get; set; }

  [Output]
  public Output<string> WebAppManagedIdentity { get; set; }

...
  public MyStack(StackConfig config, ICryptoService cryptoService)
  {
    ...

    // Assign output variables
    WebAppManagedIdentity = webApp.Name;
    SqlConnectionString = Output.Format($"Server=tcp:{sqlServer.Name}.database.windows.net;initial catalog={database.Name};Authentication=Active Directory Default;");
  }
}
```

After you run your stack using 
```csharp
var result = await stack.UpAsync(new UpOptions { OnStandardOutput = Console.WriteLine });
```

you can add your WebApp Managed Identity to your SQL Server.
```csharp
var sqlConnectionString = result.Outputs[nameof(MyStack.SqlConnectionString)].Value.ToString();
var webAppManagedIdentity = result.Outputs[nameof(MyStack.WebAppManagedIdentity)].Value.ToString();

await CreateUser(sqlConnectionString, webAppManagedIdentity);
// optionally add your personal account
await CreateUser(sqlConnectionString, "Your personal e-mail address");

private static async Task CreateUser(string sqlConnectionString, string principal)
{
    var sqlConnection = new SqlConnection(sqlConnectionString);
    await sqlConnection.OpenAsync();
    var sqlCommand = sqlConnection.CreateCommand();
    var stringBuilder = new StringBuilder();
    stringBuilder.AppendLine($"IF DATABASE_PRINCIPAL_ID('{principal}') IS NULL");
    stringBuilder.AppendLine("BEGIN");
    stringBuilder.AppendLine($"\tCREATE USER [{principal}] FROM EXTERNAL PROVIDER");
    stringBuilder.AppendLine("End");
    sqlCommand.CommandText = stringBuilder.ToString();
    await sqlCommand.ExecuteNonQueryAsync();
}
```