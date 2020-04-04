---
title: "How to fix HTTP Error 500.32 - ANCM Failed to Load dll"
subtitle: ""
date: 2020-04-04T21:44:07+02:00
draft: true
tags: ["Azure", "Pulumi", "App Service", "ANCM", "ASP.NET Core"]
---

Last week I encountered a problem after I deployed an ASP.NET Core web app to an Azure App Service using Pulumi. Pulumi is an infrastructure deployment tool based on the Terraform platform. When I previewed my deployed site I a got an error.

> HTTP Error 500.32 - ANCM Failed to Load dll

According to the Azure portal my App Service was running on .NET Framework v4.7, which is obviously wrong. 
So my next question was how I can change the stack of an App Service using Pulumi. Well, the answer is you can't and the same applies for Terraform too.

I am used to use the C# Azure SDK where you could set the stack like this

{{< highlight csharp >}}
    .WithRuntimeStack(WebAppRuntimeStack.NETCore);
{{< / highlight >}}

But as I mentioned this is not option when you use Pulumi. So I thought  the problem might be in how I publish my web app.

Basically I did the following
{{< highlight csharp >}}
dotnet publish -c Release -r win10-x64
{{< / highlight >}}

That's the crux of the matter. You may not define the runtime using -r.

{{< highlight csharp >}}
dotnet publish -c Release
{{< / highlight >}}
Witout specifying the runtime you get all runtimes in a separate runtimes folder as part of your publish result.

If you deploy that, everything is going to work.

By the way if you think you will see the Azure Portal would show your app running on .NET Core ... that is strangefully not the case. Don't let you fool.

With that in mind, happy deploying!