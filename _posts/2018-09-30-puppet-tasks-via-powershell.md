---
title: "Executing Puppet Tasks with PowerShell via the Puppet Orchestrator API"
tags:
  - puppet
  - api
  - powershell
  - automation
  - orchestration
---

In this example we'll use the Puppet task `powershell_tasks::disablesmbv1`. If we look at the task's metadata file we see the following parameters. To execute this task we'll make a call to the Puppet Orchestrator's `commands` endpoint.

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

The output is as follows. From here we can see we successfully executed the task and produced the job ID of `494`.

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

The output.

```
name                      result
----                      ------
den3-node-3.ad.piccola.us @{Enable_SMB1Protocol=True; Installed_SMB1Protocol=True}
den3-node-1.ad.piccola.us @{Enable_SMB1Protocol=True; Installed_SMB1Protocol=True}
```