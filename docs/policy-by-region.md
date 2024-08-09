# Region assignable policies

You may want to ask why?

Because we're lazy ! And we may need to manage multi region resources

Why would we make 2 different policies with 2 different assignments if we can make 1 policy with 2 assignments?

Let's use (even if deprecated) a diagnostic settings policy like this one : https://www.azadvertizer.net/azpolicyadvertizer/Deploy-Diagnostics-Firewall.html

If we look at this part of code, we notice that it is already creating the Diagnostic Setting in the same region as the resource (it makes sense right?)

```
    resources: [
        {
            type: "Microsoft.Network/azureFirewalls/providers/diagnosticSettings",
            apiVersion: "2017-05-01-preview",
            name: 🔍"[
                concat(
                    parameters('resourceName'),
                   '/',
                    'Microsoft.Insights/',
                     parameters('profileName')
                )
            ]",
            location: "[parameters('location')]",
            dependsOn: [],
            properties: {
                workspaceId: "[parameters('logAnalytics')]",
                logAnalyticsDestinationType: "[parameters('logAnalyticsDestinationType')]",
                metrics: [
                    {
                        category: "AllMetrics",
                        enabled: "[parameters('metricsEnabled')]",
                        retentionPolicy: {
                            days: 0,
                            enabled: false
                        },
                        timeGrain<span class="jsonseparators">: </span>null
                    }
                ],
                logs: [
                    {
                        category: "AzureFirewallApplicationRule",
                        enabled: "[parameters('logsEnabled')]"
                    },
                    {
                        category: "AZFWFlowTrace",
                        enabled: "[parameters('logsEnabled')]"
                    }
                ]
            }
        }
    ],
    outputs: {}
}
```

So, now that we established that we're lazy, why not re-use this existing parameter "Location" would you ask? Because we can't provide a value to it !

But now, why would we provide a value? Maybe to assign different log destinations depending on the value to limit cost...? Sounds like a nice idea [management approved] for sure.

So now, let's see what we actually need to add to an existing policy to use location:
```
// We need to add a new parameter called anything BUT Location, so I'll use resourceLocation
   "parameters": {
      "resourceLocation": {
        "type": "String",
        "metadata": {
          "displayName": "Resource Location",
          "description": "The resource Location to set the right LAW region"
        },
        "allowedValues": [
          "westeurope",
          "northeurope",
          "etc"
        ],
        "defaultValue": "westeurope"
      },
```


```
// Now we add the resourceLocation parameter as part of the conditions, this way it will trigger one specific assignment ONLY on the desired location of resources
    "policyRule": {
      "if": {
        "allOf": [
          {
            "field": "type",
            "equals": "Microsoft.Network/azureFirewalls"
          },
          {
            "field": "location",
            "equals":"[parameters('resourceLocation')]"
          },
          {
            "anyOf": [
              {
                "value": "[length(parameters('tagValuesToExclude'))]",
                "equals": 0
              },
              {
                "field": "[concat('tags[', parameters('tagName'), ']')]",
                "notIn": "[parameters('tagValuesToExclude')]"
              }
            ]
          }
        ]
      },
```


```
// Basically need to add it in the other steps like this:
          "deployment": {
            "properties": {
              "mode": "incremental",
              "parameters": {
                "resourceName": {
                  "value": "[field('fullName')]"
                },
                "location": {
                  "value": "[field('location')]"
                },
                "resourceLocation": {
                  "value": "[parameters('resourceLocation')]"
                },
                "workspaceId": {
                  "value": "[parameters('workspaceId')]"
                },
                "logAnalyticsDestinationType": {
                  "value": "[parameters('logAnalyticsDestinationType')]"
                },
                "storageAccountId": {
                  "value": "[parameters('storageAccountId')]"
                },
                "profileName": {
                  "value": "[parameters('profileName')]"
                }
              },
              "template": {
                "$schema": https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#,
                "contentVersion": "1.0.0.0",
                "parameters": {
                  "resourceName": {
                    "type": "string"
                  },
                  "location": {
                    "type": "string"
                  },
                  "resourceLocation": {
                    "type": "string"
                  },
                  "workspaceId": {
                    "type": "string",
                    "defaultValue": ""
                  },
                  "storageAccountId": {
                    "type": "string",
                    "defaultValue": ""
                  },
                  "profileName": {
                    "type": "string"
                  },
                  "logAnalyticsDestinationType": {
                    "type": "string"
                  }
                },
```

```
// And finally, we use it for the resource deployment (Could also keep using location, but for consistency I prefered to use resourceLocation)
    resources: [
        {
            type: "Microsoft.Network/azureFirewalls/providers/diagnosticSettings",
            apiVersion: "2017-05-01-preview",
            name: 🔍"[
                concat(
                    parameters('resourceName'),
                   '/',
                    'Microsoft.Insights/',
                     parameters('profileName')
                )
            ]",
            location: "[parameters('resourceLocation')]",
            dependsOn: [],
            properties: {
                workspaceId: "[parameters('logAnalytics')]",
                logAnalyticsDestinationType: "[parameters('logAnalyticsDestinationType')]",
                metrics: [
                    {
                        category: "AllMetrics",
                        enabled: "[parameters('metricsEnabled')]",
                        retentionPolicy: {
                            days: 0,
                            enabled: false
                        },
                        timeGrain<span class="jsonseparators">: </span>null
                    }
                ],
                logs: [
                    {
                        category: "AzureFirewallApplicationRule",
                        enabled: "[parameters('logsEnabled')]"
                    },
                    {
                        category: "AZFWFlowTrace",
                        enabled: "[parameters('logsEnabled')]"
                    }
                ]
            }
        }
    ],
    outputs: {}
```

Now all you need to do is create different assignments for each of your regions with those :
- resourceLocation parameter provided
- WorskpaceId of the Log Analytics Workspace in the same region
- Eventually the same logic with storage accounts

And enjoy the fact that you only need 1 diagnostic settings policy (per resource) for your whole tenant instead of one per region :)