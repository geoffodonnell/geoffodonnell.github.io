---
title: "Create a Binary PowerShell Module"
date: 2023-05-01T08:00:00-04:00
categories:
  - How-To
tags:
  - .NET Core
  - PowerShell
  - Module
---

PowerShell is a popular and powerful scripting language developed by Microsoft. PowerShell modules are collections of cmdlets, functions, variables, and other assets that can be used to simplify administrative tasks. In this post, I will walk through the process of creating a binary PowerShell module.

<!-- More -->

# Create the module project

First, create the project files for a module named `Demo` in a new directory named `powershell-demo-module`. Run the following statements in a PowerShell Terminal window.

You can accomplish the same results using Visual Studio or VS Code; the commands below will create a solution, create a folder for a project, and create the project itself, and remove the `Class1.cs` file from the template.

Create the project scaffolding:

```powershell
mkdir powershell-demo-module
cd .\powershell-demo-module
mkdir Demo.PowerShell
dotnet new sln -n Demo.PowerShell -o .\
dotnet new classlib -n Demo.PowerShell -f net6.0 -o .\Demo.PowerShell
dotnet sln .\Demo.PowerShell.sln add .\Demo.PowerShell\Demo.PowerShell.csproj
rm .\Demo.PowerShell\Class1.cs
```

Next, add a reference to `System.Management.Automation`:

```powershell
dotnet add .\Demo.PowerShell\Demo.PowerShell.csproj package System.Management.Automation -v 7.0.0
```

In order to avoid publishing unnecessary assemblies, the `Demo.PowerShell.csproj` project file needs to be manually updated. Open `Demo.PowerShell.csproj` and change:

```xml
<PackageReference Include="System.Management.Automation" Version="7.0.0" />
```

to:

```xml
<PackageReference Include="System.Management.Automation" Version="7.0.0">
    <PrivateAssets>all</PrivateAssets>
</PackageReference>
```

# Add a cmdlet

Next, we will create the `Get-Greeting` cmdlet. This cmdlet will output `"Hello, World!"`.

Create the cmdlet class `Get_Greeting.cs`:

```powershell
New-Item -Type File -Path .\Demo.PowerShell\Get_Greeting.cs
```

Open `Get_Greeting.cs`, update the content:

```csharp
using System.Management.Automation;

namespace Demo.PowerShell
{
    [Cmdlet(VerbsCommon.Get, "Greeting")]
    public class Get_Greeting : Cmdlet
    {
        protected override void ProcessRecord()
        {
            WriteObject("Hello, World!");
        }
    }
}
```

In order for a class to become an exported cmdlet, it must inherit from `System.Management.Automation.Cmdlet` and be annotated with `System.Management.Automation.CmdletAttribute`. The annotation gives the cmdlet it's full name, which is a combination of a verb (usually from the set of [approved verbs](https://learn.microsoft.com/en-us/powershell/scripting/developer/cmdlet/approved-verbs-for-windows-powershell-commands?view=powershell-7.3)) and a noun.

## Create the module manifest

While optional at this point, including a module manifest is a best practice, and will allow you to include certain metadata about your module.

In PowerShell:

```powershell
$newModuleManifestArgs = @{
    Author                  = "Demo"
    CmdletsToExport         = "*"
    CompanyName             = "Demo"
    CompatiblePSEditions    = "Core"
    Description             = "Demo PowerShell tools"
    Guid                    = "16A9AB81-7731-4098-BCE0-F8B27B3990DB"
    ModuleVersion           = "1.0.0"
    Path                    = ".\Demo.PowerShell\Demo.PowerShell.psd1"
    PowerShellVersion       = "7.2"
    RootModule              = "Demo.PowerShell.dll"
}

New-ModuleManifest @newModuleManifestArgs
```

Calling `New-ModuleManifest` with the parameter values above will create a `Demo.PowerShell.psd1` in the `Demo.PowerShell` project. The next step is to ensure that file is included alongside the build output. Add the following to the `Demo.PowerShell.csproj` file:

```xml
<ItemGroup>
    <None Update="Demo.PowerShell.psd1">
        <CopyToOutputDirectory>Always</CopyToOutputDirectory>
    </None>
</ItemGroup>
```

## Use the module locally

Build the project:

```powershell
dotnet publish ".\Demo.PowerShell\Demo.PowerShell.csproj" `
  --configuration Debug `
  --output ".\Demo.PowerShell\bin\publish" `
  --no-self-contained
```

Import the module:

```powershell
Import-Module -Name ".\Demo.PowerShell\bin\publish\Demo.PowerShell.psd1"
```

Call the cmdlet:

```powershell
Get-Greeting
```

## Publish the module

Once you have tested a module locally, you may want to publish it to a public or private repository for others to use. The script below demonstrates adding a new module repository and publishing your package.

This script is shown for demonstration purposes, in practice, this action would likely be part of a build pipeline, such as a GitHub Workflow, Azure DevOps Release, etc. You can see it used in a GitHub Workflow in [this post](http://localhost:4000/how-to/github-workflow-publish-powershell-module/). 

```powershell
## Variables
$key = "<GITHUB_TOKEN>"
$user = "<GITHUB_EMAIL>"
$feed = "https://nuget.pkg.github.com/<YOUR_GITHUB_USERNAME>/index.json"
$token = $key | ConvertTo-SecureString -AsPlainText -Force
$creds = New-Object System.Management.Automation.PSCredential -ArgumentList @($user, $token)
              
## Force TLS1.2
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
              
## Register repository
$registerArgs = @{
    Name = "GITHUB_REPOSITORY"
    SourceLocation = $feed
    PublishLocation = $feed
    InstallationPolicy = 'Trusted'
    Credential = $creds
}
              
Register-PSRepository @registerArgs

## Publish the module              
Publish-Module -Path ".\Demo.PowerShell\bin\publish\" -Repository $repositoryName -NuGetApiKey $key
```

## References

PowerShell Docs
* [Binary Module](https://learn.microsoft.com/en-us/powershell/scripting/developer/module/how-to-write-a-powershell-binary-module)
* [Module Manifest](https://learn.microsoft.com/en-us/powershell/scripting/developer/module/how-to-write-a-powershell-module-manifest)

PowerShell Verbs
* [VerbsCommon](https://learn.microsoft.com/en-us/dotnet/api/System.Management.Automation.VerbsCommon)
* [VerbsCommunications](https://learn.microsoft.com/en-us/dotnet/api/System.Management.Automation.VerbsCommunications)
* [VerbsData](https://learn.microsoft.com/en-us/dotnet/api/System.Management.Automation.VerbsData)
* [VerbsDiagnostic](https://learn.microsoft.com/en-us/dotnet/api/System.Management.Automation.VerbsDiagnostic)
* [VerbsLifecycle](https://learn.microsoft.com/en-us/dotnet/api/System.Management.Automation.VerbsLifecycle)
* [VerbsOther](https://learn.microsoft.com/en-us/dotnet/api/System.Management.Automation.VerbsOther)
* [VerbsSecurity](https://learn.microsoft.com/en-us/dotnet/api/System.Management.Automation.VerbsSecurity)