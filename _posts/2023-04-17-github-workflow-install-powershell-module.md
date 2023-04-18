---
title: "Install a PowerShell Module from GitHub Packages"
date: 2023-04-17T20:40:00-04:00
categories:
  - How-To
tags:
  - GitHub
  - Workflow
  - Packages
  - PowerShell
  - Module
---

I arrived at the steps below after some trial and error. At the time this writing (04/17/2023) there seems to be some incompatibility with the default version of [PowerShellGet](https://learn.microsoft.com/en-us/powershell/module/powershellget/?view=powershellget-2.x) and the version of NuGet server that powers GitHub Packages. As a workaround to these incompatibilities, my approach is as follows:

  * Set up the NuGet client to read packages from private GitHub Packages registry
  * Create a local PowerShell Module Registry
  * Using the NuGet client, download the module package directly to local PowerShell Module Registry
  * Install package from local PowerShell Module Registry

Note that [publishing a PowerShell module to GitHub Packages]({% post_url 2023-04-16-github-workflow-publish-powershell-module %}) is discussed in a previous post.

{% raw %}
```yml
jobs:
    publish:
        runs-on: ubuntu-latest
        permissions:
          actions: read
          packages: write
        steps:
          - uses: nuget/setup-nuget@v1
            with:
              nuget-version: '5.x'
          - name: Add private Nuget source
            shell: pwsh
            run: |-
              $user = "${{ github.actor }}"
              $token = "${{ env.nugetSourceKey }}"
              $feed = "${{ env.nugetSourceUrl }}"
              $packageSourceName = "PrivateNugetSource"

              ## Add the package source
              nuget sources add `
                -Name $packageSourceName `
                -UserName $user `
                -Password $token `
                -Source $feed `
                -Verbosity detailed `
                -StorePasswordInClearText

              ## Set the API key
              nuget setapikey $token -Source $feed

              ## Create a directory for local repository
              mkdir ./__modules

              ## Register the local repository
              Register-PSRepository `
                -Name 'localRepository' `
                -SourceLocation ./__modules `
                -InstallationPolicy Trusted 

          - name: Install PowerShell module
            shell: pwsh
            run: |-

              ## Install the package containing the module to the local repository
              nuget install $moduleName `
                -DirectDownload `
                -OutputDirectory ./__modules `
                -Source ${{ env.nugetSourceUrl }} `
                -Verbosity detailed

              ## Install the module from the local repository
              Install-Module -Name $moduleName -Repository 'localRepository'
```
{% endraw %}

Check out the [GitHub Packages docs][github-packages-docs] for more info on publishing packages to GitHub.

[github-packages-docs]: https://docs.github.com/en/packages/learn-github-packages/introduction-to-github-packages
