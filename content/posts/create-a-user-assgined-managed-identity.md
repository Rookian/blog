---
title: "Create a User Assgined Managed Identity"
date: 2023-01-29T13:15:47+01:00
draft: false
tags: ["Azure", "Azure SDK", "Managed Identity", "User-assgined", "AzureAD"]
---
---
Previous blog post in the series: [Introduction to Managed Identities](/posts/introduction-to-managed-identities)

Posts in this series:
- [Introduction to Managed Identities](/posts/introduction-to-managed-identities)
- [Create a user-assigned Managed Identity](/posts/create-a-user-assgined-managed-identity) 
- Using Managed Identity with Azure SQL (coming soon)
- Machine to Machine authentication (coming soon)
---
With the new [Azure SDK Management Libraries](https://azure.github.io/azure-sdk/releases/latest/mgmt/dotnet.html) I have had the requirement to create a user-assgined Identity. Unfortunately there is no easy way in the new SDK to do so. So I created my own little helper method.

Nuget Packages
- [Azure.ResourceManager.Resources](https://www.nuget.org/packages/Azure.ResourceManager.Resources)
- [Azure.Identity](https://www.nuget.org/packages/Azure.Identity)  
  
Code
```csharp
async Task<UserAssignedIdentity> CreateOrUpdateUserAssignedIdentity(string identityName, string resourceGroup)
{
    var armClient = new Azure.ResourceManager.ArmClient(new AzureCliCredential());
    var sub = await armClient.GetDefaultSubscriptionAsync();
    var rg = (await sub.GetResourceGroups().GetAsync(resourceGroup)).Value;
    var umi = new GenericResourceData(AzureLocation.WestEurope);
    var umiId = rg.Id.AppendProviderResource("Microsoft.ManagedIdentity", "userAssignedIdentities",
        identityName);
    var res = await armClient.GetGenericResources().CreateOrUpdateAsync(Azure.WaitUntil.Completed, umiId, umi);
    var userAssignedIdentity =
        res.Value.Data.Properties.ToObjectFromJson<UserAssignedIdentity>(new JsonSerializerOptions()
            { PropertyNameCaseInsensitive = true });
    return userAssignedIdentity;
}
record UserAssignedIdentity(string PrincipalId, string ClientId, string TenantId);
```