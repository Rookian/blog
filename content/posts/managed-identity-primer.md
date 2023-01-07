---
title: "Getting rid of credentials with Managed Identities in Azure"
date: 2023-01-07T11:07:43+02:00
draft: true
tags: ["Azure", "Managed Identity", "Security"]
---

Posts in this series:
- [Managed Identity primer](/posts/managed-identity-primer)
- [Create a user-assigned Managed Identity](/posts/create-a-user-assgined-managed-identity)
- [Using Managed Identity with Azure SQL](/posts/using-managed-identity-with-azure-sql)
- [Machine to Machine authentication](/posts/todo)

# Introduction to Managed Identities in Azure?  
A Managed Identity (MI), formerly known as Managed Service Identity (MSI), is a special Service Principal that is exclusively designed for Azure resources. You can assign one or more identities to an Azure resource, and these Managed Identities can be used to access other Azure resources within the Azure environment. This allows you to eliminate the need for credentials, as the Azure resource can use its assigned identity and the appropriate RBAC permissions to access other resources.

> ⚠ **Important** The Identity concept is solely used for identifying an Azure resource from a client perspective.  
> e.g. Azure App Service (client) wants to access Azure SQL database.  
> Managed Identities should not be confused with server-side authentication, which requires the use of an Application/App Registration and Service Principal. More information on this topic can be found in the post on[Machine to Machine authentication](/posts/todo).

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
The Azure Identity Library is a collection of libraries that make it easy to authenticate with Azure Active Directory and manage Managed Identities in your applications. The library is available for a variety of platforms, including .NET, Java, and Python.

Using the Azure Identity Library, you can:
- Authenticate with Azure Active Directory using a variety of authentication flows
- Manage Managed Identities for Azure Resources
- Acquire tokens for accessing Azure resources