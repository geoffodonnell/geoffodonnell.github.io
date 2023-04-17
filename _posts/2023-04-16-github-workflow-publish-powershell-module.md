---
title: "GitHub Workflow: Publish a PowerShell Module to GitHub Packages"
date: 2023-04-16T20:40:00-04:00
categories:
  - How-To
tags:
  - GitHub
  - Workflow
  - Packages
  - PowerShell
  - Module
---

Publishing your PowerShell module is straightfoward within a GitHub workflow. 

{% raw %}
```yml
jobs:
    publish:
        runs-on: ubuntu-latest
        permissions:
          actions: read
          packages: write
        steps:
          # Publish module step assumes the module files are located in ${{ runner.temp }}
          - name: Publish module
            shell: pwsh
            run: |-
              $user = "${{ github.actor }}"
              $token = "${{ github.token }}" | ConvertTo-SecureString -AsPlainText -Force
              $creds = New-Object System.Management.Automation.PSCredential -ArgumentList @($user, $token)
              $feed = "https://nuget.pkg.github.com/${{ github.repository_owner }}/index.json"
              $moduleName = "${{ env.moduleName }}"
              $repositoryName = "PowershellNugetServices"
              
              $dropPath = "${{ runner.temp }}"
              $modulePath = [System.IO.Path]::GetFullPath((Join-Path -Path $dropPath -ChildPath $moduleName))
              
              ## Force TLS1.2
              [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
              
              ## Register repository
              $registerArgs = @{
                  Name = $repositoryName
                  SourceLocation = $feed
                  PublishLocation = $feed
                  InstallationPolicy = 'Trusted'
                  Credential = $creds
              }
              
              Register-PSRepository @registerArgs
              
              Publish-Module -Path $modulePath `
                -Repository $repositoryName `
                -NuGetApiKey "${{ secrets.GITHUB_TOKEN }}"  
```
{% endraw %}


Check out the [GitHub Packages docs][github-packages-docs] for more info on publishing packages to GitHub.

[github-packages-docs]: https://docs.github.com/en/packages/learn-github-packages/introduction-to-github-packages
