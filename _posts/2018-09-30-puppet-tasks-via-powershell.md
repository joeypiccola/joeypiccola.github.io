---
title: "Executing Puppet Tasks with PowerShell via the Puppet Orchestrator API"
excerpt: Execute PowerShell on remote systems via a Puppet Task triggered via the Puppet API.
tags:
  - puppet
  - api
  - powershell
  - automation
  - orchestration
---

<!-- TOC -->

- [The Hard Way](#the-hard-way)
- [The Easy Way](#the-easy-way)
- [The Functions](#the-functions)

<!-- /TOC -->

## The Hard Way

Lets begin by writing the code out manually so we understand what's going on with the `URI`'s and `Invoke-WebRequest`. In this example we'll use the Puppet task `powershell_tasks::disablesmbv1`. If we look at the task's metadata file we see what it does via the description and what parameters it allows. To execute this task we'll make a call to the Puppet Orchestrator's `commands` endpoint.

```json
{
    "puppet_task_version": 1,
    "description": "A task to test if SMBv1 is enabled and optionally disable it.",
    "input_method": "powershell",
    "parameters": {
        "action": {
            "description": "Valid actions are get and set.",
            "type": "Enum[get, set]"
        },
        "reboot": {
            "description": "Do we want to reboot the system if changes were made?",
            "type": "Optional[Boolean]"
        },
        "forcereboot": {
            "description": "Do we want to reboot the system regardless if changes were made?",
            "type": "Optional[Boolean]"
        }
    }
}
```

Lets begin by calling this task with its `action` parameter and the value of `get`.

```powershell
$master = 'puppet.piccola.us'
$token = '***'

$targetNodes = @('den3-node-1.ad.piccola.us','den3-node-3.ad.piccola.us')

$req = [PSCustomObject]@{
    environment = 'production'
    task        = 'powershell_tasks::disablesmbv1'
    params      = [PSCustomOBject]@{
        action = 'get'
    }
    description = 'get smbv1 status'
    scope       = [PSCustomObject]@{
        nodes = $targetNodes
    }
} | ConvertTo-Json

$hoststr = "https://$master`:8143/orchestrator/v1/command/task"
$headers = @{'X-Authentication' = $token}

Invoke-WebRequest -Uri $hoststr -Method Post -Headers $headers -Body $req
```

The output. From here we can see we successfully executed the task and produced the job ID of `494`.

```
StatusCode        : 202
StatusDescription : Accepted
Content           : {
                      "job" : {
                        "id" : "https://puppet.piccola.us:8143/orchestrator/v1/jobs/494",
                        "name" : "494"
                      }
                    }
RawContent        : HTTP/1.1 202 Accepted
                    Vary: Accept-Encoding, User-Agent
                    Content-Length: 108
                    Content-Type: application/json;charset=utf-8
                    Date: Sun, 30 Sep 2018 17:16:11 GMT
                    Server: Jetty(9.4.z-SNAPSHOT)

                    {
                      "...
Forms             : {}
Headers           : {[Vary, Accept-Encoding, User-Agent], [Content-Length, 108], [Content-Type, application/json;charset=utf-8], [Date, Sun, 30 Sep 2018 17:16:11 GMT]...}
Images            : {}
InputFields       : {}
Links             : {}
ParsedHtml        : mshtml.HTMLDocumentClass
RawContentLength  : 108
```

That's great, but lets get the output results from each of the nodes we just ran the task on. To do this we'll make a call to the Puppet Orchestrator's `jobs` endpoint. Lets use the `494` job ID from before.

```powershell
$master = 'puppet.piccola.us'
$token = '***'

$hoststr = "https://$master`:8143/orchestrator/v1/jobs/494/nodes"
$headers = @{'X-Authentication' = $Token}

$result  = Invoke-WebRequest -Uri $hoststr -Method Get -Headers $headers
$content = $result.content | ConvertFrom-Json

foreach ($item in $content.items) {
    $item | Select-Object name, result
}
```

The output. Nice, we can see that on our systems we not only have SMBv1 as an enabled protocol but the feature itself is also installed. Lets disable it!

```
name                      result
----                      ------
den3-node-3.ad.piccola.us @{Enable_SMB1Protocol=True; Installed_SMB1Protocol=True}
den3-node-1.ad.piccola.us @{Enable_SMB1Protocol=True; Installed_SMB1Protocol=True}
```

From here, lets modify our task parameters to disable SMBv1. We've changed our `action` param to `set` and also provided the `reboot` parameter with a value of `$true`. We want our systems to reboot so they take the SMBv1 changes.


```powershell
$master = 'puppet.piccola.us'
$token = '***'

$targetNodes = @('den3-node-1.ad.piccola.us','den3-node-3.ad.piccola.us')

$req = [PSCustomObject]@{
    environment = 'production'
    task        = 'powershell_tasks::disablesmbv1'
    params      = [PSCustomOBject]@{
        action = 'set'
        reboot = $true
    }
    description = 'set smbv1 status'
    scope       = [PSCustomObject]@{
        nodes = $targetNodes
    }
} | ConvertTo-Json

$hoststr = "https://$master`:8143/orchestrator/v1/command/task"
$headers = @{'X-Authentication' = $token}

Invoke-WebRequest -Uri $hoststr -Method Post -Headers $headers -Body $req
```

If we run our task again with `get` we can see that SMBv1 is disabled and uninstalled. Awesome.

```
name                      result
----                      ------
den3-node-3.ad.piccola.us @{Enable_SMB1Protocol=False; Installed_SMB1Protocol=False}
den3-node-1.ad.piccola.us @{Enable_SMB1Protocol=False; Installed_SMB1Protocol=False}
```

## The Easy Way

While this has been fun, lets wrap all this PowerShell up into a few functions. `Invoke-PuppetTask`, `Get-PuppetJobNodes`, and `Get-PuppetJob`. Note we added `Get-PuppetJob`, this function is going to simply get job details from a supplied Job ID. `Invoke-PuppetTask` will use this function internally to monitor our job's `state`. You'll see this below with `Invoke-PuppetTask`'s `-Wait` and `-Timeout` parameters.

**Get SMBv1 status.**

```powershell
$master = 'puppet.piccola.us'
$token = '***'

$scope = @('den3-node-1.ad.piccola.us','den3-node-5.ad.piccola.us')
$splat = @{
    Token = $Token
    Master = $Master
    Task = 'powershell_tasks::disablesmbv1'
    Environment = 'production'
    Parameters = [PSCustomObject]@{
        action = 'get'
    }
    Description = 'Get SMBv1'
    Scope = $scope
    ScopeType = 'nodes'
}
$taskRunGet = Invoke-PuppetTask @splat -Wait -Timeout 120
Get-PuppetJobNodes -Token $Token -Master $Master -ID $taskRunGet.job.name | select name, result
```
*Output.*
```
name                      result
----                      ------
den3-node-5.ad.piccola.us @{Enable_SMB1Protocol=True; Installed_SMB1Protocol=True}
den3-node-1.ad.piccola.us @{Enable_SMB1Protocol=True; Installed_SMB1Protocol=True}
```

**Set SMBv1 status.**


```powershell
$master = 'puppet.piccola.us'
$token = '***'

$scope = @('den3-node-1.ad.piccola.us','den3-node-5.ad.piccola.us')
$splat = @{
    Token = $Token
    Master = $Master
    Task = 'powershell_tasks::disablesmbv1'
    Environment = 'production'
    Parameters = [PSCustomObject]@{
        action = 'set'
        reboot = $true
    }
    Description = 'Set SMBv1'
    Scope = $scope
    ScopeType = 'nodes'
}
$taskRunSet = Invoke-PuppetTask @splat -Wait -Timeout 120
# let the systems reboot and come back up
Start-Sleep -Seconds 120
Get-PuppetJobNodes -Token $Token -Master $Master -ID $taskRunGet.job.name | select name, result
```
*Output.*
```
name                      result
----                      ------
den3-node-5.ad.piccola.us @{Enable_SMB1Protocol=False; Installed_SMB1Protocol=False}
den3-node-1.ad.piccola.us @{Enable_SMB1Protocol=False; Installed_SMB1Protocol=False}
```

Sweet, we've successfully executed PowerShell on remote systems via a Puppet Task triggered via the Puppet API.

## The Functions

**Get-PuppetJobNodes**
```powershell
Function Get-PuppetJobNodes {
    Param(
        [Parameter(Mandatory)]
        [int]$ID,
        [Parameter(Mandatory)]
        [string]$Token,
        [Parameter(Mandatory)]
        [string]$Master
    )

    $hoststr = "https://$master`:8143/orchestrator/v1/jobs/$id/nodes"
    $headers = @{'X-Authentication' = $Token}

    $result  = Invoke-WebRequest -Uri $hoststr -Method Get -Headers $headers
    $content = $result.content | ConvertFrom-Json

    Write-Output $content.items
}
```

**Invoke-PuppetTask**
```powershell
Function Invoke-PuppetTask {
    Param(
        [Parameter(Mandatory)]
        [string]$Token,
        [Parameter(Mandatory)]
        [string]$Master,
        [Parameter(Mandatory)]
        [string]$Task,
        [Parameter()]
        [string]$Environment = 'production',
        [Parameter()]
        [PSCustomObject]$Parameters = @{},
        [Parameter()]
        [string]$Description = '',
        [Parameter(Mandatory)]
        [PSCustomObject[]]$Scope,
        [Parameter(Mandatory)]
        [ValidateSet('nodes')]
        [string]$ScopeType,
        [Parameter()]
        [switch]$Wait,
        [Parameter()]
        [int]$Timeout = 300
    )

    $req = [PSCustomObject]@{
        environment = $Environment
        task        = $Task
        params      = $Parameters
        description = $Description
        scope       = [PSCustomObject]@{
        $ScopeType = $Scope
        }
    } | ConvertTo-Json

    $hoststr = "https://$master`:8143/orchestrator/v1/command/task"
    $headers = @{'X-Authentication' = $Token}

    $result  = Invoke-WebRequest -Uri $hoststr -Method Post -Headers $headers -Body $req
    $content = $result.content | ConvertFrom-Json

    if ($wait) {
        # sleep 5s for the job to register
        Start-Sleep -Seconds 5

        $jobSplat = @{
            token = $Token
            master = $master
            id = $content.job.name
        }

        # create a timespan
        $timespan = New-TimeSpan -Seconds $timeout
        # start a timer
        $stopwatch = [diagnostics.stopwatch]::StartNew()

        # get the job state every 5 seconds until our timeout is met
        while ($stopwatch.elapsed -lt $timespan) {
            # optoins are new, ready, running, stopping, stopped, finished, or failed
            $job = Get-PuppetJob @jobSplat
            if (($job.State -eq 'stopped') -or ($job.State -eq 'finished') -or ($job.State -eq 'failed')) {
                $taskJobContent = [PSCustomObject]@{
                    task = $content
                    job = $job
                }
                Write-Output $taskJobContent
                break
            }
            Start-Sleep -Seconds 5
        }
        if ($stopwatch.elapsed -ge $timespan) {
            Write-Error "Timeout of $Timeout`s has exceeded."
            break
        }
    } else {
        Write-Output $content
    }
}
```

**Get-PuppetJob**
```powershell
Function Get-PuppetJob {
    Param(
        [Parameter(Mandatory)]
        [int]$ID,
        [Parameter(Mandatory)]
        [string]$Token,
        [Parameter(Mandatory)]
        [string]$Master
    )

    $hoststr = "https://$master`:8143/orchestrator/v1/jobs/$id"
    $headers = @{'X-Authentication' = $Token}

    $result  = Invoke-WebRequest -Uri $hoststr -Method Get -Headers $headers
    $content = $result.content | ConvertFrom-Json

    Write-Output $content
}
```