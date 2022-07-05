---
title: "Create a User Assgined Managed Identity"
date: 2022-07-05T11:15:47+02:00
draft: false
tags: ["Azure", "Azure SDK", "Managed Identity", "User-assgined", "AzureAD"]
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