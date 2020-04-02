---
title: "Deploying an Azure Web App with Pulumi and Nuke using WEBSITE_RUN_FROM_PACKAGE"
date: 2020-04-02T21:37:20+02:00
draft: true
tags: ["azure", "pulumi", "deployment"]
---

**Nuke build**

{{< highlight csharp >}}
using Nuke.Common;
using Nuke.Common.IO;
using Nuke.Common.Tools.DotNet;
using Nuke.Common.Utilities.Collections;
using static Nuke.Common.IO.FileSystemTasks;
using static Nuke.Common.IO.PathConstruction;
using static Nuke.Common.Tools.DotNet.DotNetTasks;

namespace Example.Build
{
    public class Program : NukeBuild
    {
        public static int Main() => Execute<Program>(x => x.Package);
        private static AbsolutePath BackendWorkspace => RootDirectory / "src" / "apps";
        private static AbsolutePath BuildDestination => RootDirectory / "build";

        private Target CleanBuildDestination => _ => _
            .Executes(() => EnsureCleanDirectory(BuildDestination));

        Target CleanBackend => _ => _
            .Executes(() => GlobDirectories(RootDirectory / "src" / "apps", "**/bin", "**/obj").ForEach(DeleteDirectory));

        Target RestoreBackend => _ => _
            .DependsOn(CleanBackend)
            .Executes(() => DotNetRestore(s => s.SetProjectFile(BackendWorkspace)));

        Target BuildBackend => _ => _
            .DependsOn(RestoreBackend)
            .Executes(() =>
                DotNetBuild(s => s
                    .SetProjectFile(BackendWorkspace)
                    .SetConfiguration(Configuration.Release)
                    .EnableNoRestore()));

        Target PublishExampleUI => _ => _
            .DependsOn(CleanBuildDestination)
            .Executes(() =>
                {
                    DotNetPublish(s => s
                        .SetProject(BackendWorkspace / "Example.UI" / "Example.UI.csproj")
                        .SetOutput(BuildDestination / "Example.UI")
                        .SetConfiguration("Release"));
                }
            );

        private Target CopyRelease => _ => _
            .DependsOn(CleanBuildDestination)
            .Executes(() => CopyDirectoryRecursively(RootDirectory / "src" / "release", BuildDestination / "release"));

        Target Package => _ => _
            .DependsOn(PublishExampleUI)
            .DependsOn(CopyRelease);
    }
}
{{< / highlight >}}

**build.cmd**
{{< highlight yml >}}
@SET DOTNET_CLI_TELEMETRY_OPTOUT=1
@dotnet run --project src\build\Example.Build\Example.Build.csproj -- %*
{{< / highlight >}}

**azure-pipelines.yml**

{{< highlight yml >}}
trigger:
 branches:
   include:
    - '*'
 paths:
   exclude:
     - docs/*
resources:
- repo: self
name: $(Date:yyyyMMdd)$(Rev:.rr)
pool:
  vmImage: windows-2019
jobs:
- job: Build
  steps:
  - task: BatchScript@1
    displayName: 'Run script build.cmd'
    inputs:
      filename: build.cmd
  - task: PublishPipelineArtifact@1
    inputs:
      targetPath: '$(System.DefaultWorkingDirectory)\\build'
      artifact: 'WebSite'
      publishLocation: 'pipeline'
{{< / highlight >}}

**Pulumi stack**

{{< highlight csharp >}}

using System.IO;
using Pulumi.Azure.AppService;
using Pulumi.Azure.AppService.Inputs;
using Pulumi.Azure.Storage;

// ReSharper disable ObjectCreationAsStatement

namespace Deployment.Infrastructure
{
    public class ExampleStack : Stack
    {
        privat const string ResourceGroupName = "my-resource-group"
        public ExampleStack()
        {

            var appServicePlan = new Plan("serviceplan", new PlanArgs
            {
                ResourceGroupName = ResourceGroupName,
                Kind = "App",
                Sku = new PlanSkuArgs
                {
                    Tier = "Basic",
                    Size = "B1",
                }
            });

            var appServiceStorageAccount = new Account("appServStore", new AccountArgs
            {
                ResourceGroupName = ResourceGroupName,
                AccountReplicationType = "LRS",
                AccountTier = "Standard",
            });

            var packagePath = "../../Example.API");

            var container = new Container(containerName, new ContainerArgs
            {
                ContainerAccessType = "private",
                StorageAccountName = appServiceStorageAccount.Name
            });

            var blob = new Blob($"package-{Guid.NewGuid()}", new BlobArgs
            {
                StorageAccountName = appServiceStorageAccount.Name,
                StorageContainerName = container.Name,
                Type = "Block",
                Source = new FileArchive(packagePath)
            });

            var blobUrl = SharedAccessSignature.SignedBlobReadUrl(blob, appServiceStorageAccount);

            new AppService(GetResourceName("searchapi"), new AppServiceArgs
            {
                Name = GetResourceName("searchapi"),
                ResourceGroupName = ResourceGroupName,
                AppServicePlanId = appServicePlan.Id,
                AppSettings =
                {
                    "WEBSITE_RUN_FROM_PACKAGE", blobUrl},
                },
                HttpsOnly = true
            });
        }
    }
}
{{< / highlight >}}

Deployment

run-pulumi-azure-devops.ps1

{{< highlight powershell >}}
[CmdletBinding()]
Param(
	$AZURE_STORAGE_KEY,
	$PULUMI_CONFIG_PASSPHRASE,
	$EndpointClientSecret)

$ErrorActionPreference = "Stop"
# Set via Azure Devops Environment variables
# $Env:ARM_CLIENT_ID 
# $Env:ARM_CLIENT_SECRET
# $Env:ARM_TENANT_ID
# $Env:ARM_SUBSCRIPTION_ID 
$Env:AZURE_STORAGE_KEY = $AZURE_STORAGE_KEY
$Env:PULUMI_CONFIG_PASSPHRASE = $PULUMI_CONFIG_PASSPHRASE

Write-Host ClientId: 		$Env:ARM_CLIENT_ID
Write-Host TenantId: 		$Env:ARM_TENANT_ID
Write-Host SubscriptionId: 	$Env:ARM_SUBSCRIPTION_ID

if (!$Env:Environment) {
	throw "No Environment set."
}

$pulumi = Resolve-Path ".\tools\Pulumi\bin"
$Env:Path += ";$pulumi"

pulumi login --cloud-url azblob://pulumi

Write-Host Running pulumi for environment $Env:Environment
pulumi stack select $Env:Environment
pulumi up -y
{{< / highlight >}}