---
title: "Getting rid of credentials with Managed Identities in Azure"
date: 2023-01-29T13:15:47+01:00
draft: false
tags: ["Azure", "Managed Identity", "Security"]
---
---
Posts in this series:
- [Introduction to Managed Identities](/posts/introduction-to-managed-identities)
- [Create a user-assigned Managed Identity](/posts/create-a-user-assgined-managed-identity) 
- Using Managed Identity with Azure SQL (coming soon)
- Machine to Machine authentication (coming soon)
---
# Introduction to Managed Identities
A Managed Identity (MI), formerly known as Managed Service Identity (MSI), is a special Service Principal that is exclusively designed for Azure services. You can assign one or more identities to an Azure service and these Managed Identities can be used to access other Azure services within the Azure environment. This allows you to eliminate the need for credentials, as the Azure service can use its assigned identity to access other services.

> ⚠ **Important** With Managed Identity your are identifying **a client** (Azure service) not a server. Or in other words a Managed Identity is leveraging the [OAuth client credential flow](https://www.oauth.com/oauth2-servers/access-tokens/client-credentials/).

![Managed Identity - Overview](/img/mi-overview.jpg)

# Types of Managed Identities
There are two types of Managed Identities:

## System-assigned Identities
![system-assigned-mi](/img/system-assigned-mi.png)
  - Created when an Azure resource is created
  - Deleted when the corresponding Azure resource is deleted
  - Have the same name as the Azure resource

**Pros**
  - Easy to setup  

**Cons**  
  - Prone to be unstable, as you would create a new Managed Identity when you delete the corresponding Azure resource

## User-assigned Identities
![user-assigned-mi](/img/user-assigned-mi.png)
  - Independent Azure resource with its own life cycle
  - Available in the Azure Portal under Managed Identities
  - Identity can be shared among multiple Azure resources  

**Pros**
 - Very stable, as the life cycle of the identity is not tied to an Azure resourc
 - Can be used to share common authorization among resources, reducing the need for redundant configuration

**Cons**
 - Requires more setup and configuration
  
# Using the Azure Identity Library

[Azure.Identity](https://www.nuget.org/packages/Azure.Identity) is Microsoft's latest library, which is going to replace [Microsoft.Azure.Services.AppAuthentication](https://www.nuget.org/packages/Microsoft.Azure.Services.AppAuthentication), to unify authentication against Azure resources. Azure.Identity is based on [Azure.Core](https://www.nuget.org/packages/Azure.Core), which provides a common abstract base class called `TokenCredential`. A `TokenCredential` consists of two abstract methods.
- `GetTokenAsync`
- `GetToken`

Azure.Identity provides several implementations of TokenCredential.
- `AzureCliCredential`
- `ManagedIdentityCredential`
- `DefaultAzureCredential`
- `ChainedTokenCredential`
- and many more

I typically use a combination of `AzureCliCredential` and `ManagedIdentityCredential` and combine them using `ChainedTokenCredential`.

```csharp
var chainedTokenCredential = new ChainedTokenCredential(new ManagedIdentityCredential(), new AzureCliCredential());
```
`ChainedTokenCredential` is a composite that calls the given `TokenCredential`s in the order they were provided. The first success try to get a token will end the chain. `ManagManagedIdentityCredential` would kickin when your service runs on Azure whereas for local development `AzureCliCredential` would be used. 
Before using `AzureCliCredential` ensure that you provide your Azure user credentials on the command line by typing in `az login`.

Making use of a TokenCredential could then look like this.
```csharp
var blobServiceClient = new BlobServiceClient(new Uri(blobUri), chainedTokenCredential);
```
Next blog post in this series: [Create a user-assigned Managed Identity](/posts/create-a-user-assgined-managed-identity) 