---
title: "Executing Puppet Tasks with PowerShell via the Puppet Orchestrator API"
tags:
  - puppet
  - api
  - powershell
  - automation
  - orchestration
---

In this example we'll use the Puppet task `powershell_tasks::disablesmbv1`. If we look at the task's metadata file we see the following parameters.

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

The output is as follows.

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