# Azure SQL Entra ID compatible template

As of June 2024, the bicep template provided for SQL servers/databases/ManagedInstances isn't working with the AADOnly parameter properly (Entra Id)

> This solution answers this raised issue: [Github issue](https://github.com/Azure/bicep-types-az/issues/1436)

This is due to the requirement for a SQL account at resource creation, and even if Microsoft is working for a solution to find the resource existence before deployment (without failing it if it doesn't...), today there is no Microsoft provided solution for a "fit all cases" SQL template.

In this page, you'll find some logic to add to your template to make it work in all cases:
- Initial deployment using a SQL account
- Initial deployment forcing Entra ID only
- Redeployments for both cases
- This template is compatible with the following Azure policies on deny:
    - Azure SQL Database should have Microsoft Entra-only authentication enabled
    - Azure SQL Database should have Microsoft Entra-only authentication enabled during creation

I'll consider that if you are facing such an issue, you already possess basic bicep/arm skills and won't provide the basic things but the main solution.

To have this template work as expected, you will need to do in the specified order:
1. Create a user Assigned Identity for the SQL server
2. Give it reader RBAC on the resourcegroup level
3. Use a deployment script to wait 30 seconds for RBAC propagation
4. Use a deployment script to fetch if the resource already exists
5. Deploy the SQL server using the previous script result


### Let's start with the main deployment script

```csharp
// Showing some params for better understanding of the following code, use as you wish of course.
@description('The name of the SQL logical server.')
param name string = '${resourceGroup().name}-${uniqueId}-sql'

@description('Unique Id to be used on created resources')
param uniqueId string = take(uniqueString(resourceGroup().id), 4)

@description('This will enable Azure Active Directory Only Authentication')
param azureADOnlyAuthentication bool

param adminUsername string
administratorLoginPassword string = '' // Prefer some random function here using guid() function

// Step 1 - create a user Assigned identity using your own modules or bicep native template
module managedIdentity '../../Microsoft.ManagedIdentity/userAssignedIdentities/azuredeploy.bicep' = {
  name: '${name}-mi-deploy-${uniqueId}'
  params: {
    name: '${name}-mi'
  }
}

// Step 2 - give the new MI reader of the resourceGroup scop
module rbacMIReader 'submodules/roleAssignmentResourceGroup.bicep' = {
  name: 'rbac-rules-${name}-mi-Reader-${uniqueId}'
  params: {
    principalId: managedIdentity.outputs.principalID
    roleDefinitionId: 'acdd72a7-3385-48ef-bd42-f606fba81ae7'
  }
  dependsOn: [
    managedIdentity
  ]
}

// Step 3 (don't skip it) - force a 30 seconds sleep for the RBAC to properly propagate, or it may sometimes fail.
module waitBeforeCheck 'submodules/sleep.bicep' = {
  name: '${name}-check-sleep-${uniqueId}'
  params: {
    sleepingTime: 30
    name: name
    location: location
  }
  dependsOn: [
    rbacMIReader
  ]
}

// Step 4 - Invoke the deployment script used to see if the resource exists or not
module checkIfServerExists 'submodules/serverCheckScript.bicep' = {
  name: 'checkIf-${name}-Exists-${uniqueId}'
  params: {
    name: name
    location: location
  }
  dependsOn: [
    waitBeforeCheck
  ]
}

// Step 5 deploy the resource with logic based on previous inputs
resource sqlDeploy 'Microsoft.Sql/servers@2020-11-01-preview' = {
  name: name
  location: location
  identity: enableSystemManagedIdentity ? smiObject : null
  properties: {
    // If the deployment scripts finds the resource, the output checkIfServerExists.outputs.sqlServerNam won't be empty, and we can use it to apply some logic
    // If the output is NOT EMPTY AND param azureADOnlyAuthentication is true, then it provived null for both administratorLogin and administratorLoginPassword.
    // If either the resource does not exist yet or the param azureADOnlyAuthentication is false, then it will apply provided values
    administratorLogin: (!empty(checkIfServerExists.outputs.sqlServerName) && azureADOnlyAuthentication==true) ? null : adminUsername
    administratorLoginPassword: (!empty(checkIfServerExists.outputs.sqlServerName) && azureADOnlyAuthentication==true)  ? null : administratorLoginPassword
    version: '12.0'
    minimalTlsVersion: minimalTlsVersion
    publicNetworkAccess: publicNetworkAccess
    administrators: !empty(azureADAdministrator)
      ? union(
          defaultADAdministrator,
          azureADAdministrator,
          {
            azureADOnlyAuthentication: azureADOnlyAuthentication
          }
        )
      : {}
  }
  dependsOn:[
    checkIfServerExists
  ]
}
```

### waitBeforeCheck module

```csharp
param sleepingTime int
param name string
param location string
var waitSectionName = 'waitSection-${take(uniqueString(resourceGroup().id, name),8)}'

resource waitSection 'Microsoft.Resources/deploymentScripts@2020-10-01' = {
  name: waitSectionName
  location: location
  kind: 'AzurePowerShell'
  identity: {
    type: 'UserAssigned'
    userAssignedIdentities: {
      '${resourceId('Microsoft.ManagedIdentity/userAssignedIdentities', '${name}-mi')}': {}
    }
  }
  properties: {
    azPowerShellVersion: '6.5'
    environmentVariables: []
    scriptContent: 'start-sleep -Seconds ${sleepingTime}'
    cleanupPreference: 'Always'
    retentionInterval: 'PT1H'
  }
}
```

### CheckIfServerExists module

```csharp
param location string
param name string
param utcValue string = utcNow()

var resourceGroupName = resourceGroup().name

// This script is fetching a single value (due to the way QS is used)
resource resource_exists_script 'Microsoft.Resources/deploymentScripts@2020-10-01' = {
  name: '${name}-resource_exists'
  location: location
  kind: 'AzureCLI'
  identity: {
    type: 'UserAssigned'
    userAssignedIdentities: {
      '${resourceId('Microsoft.ManagedIdentity/userAssignedIdentities', '${name}-mi')}': {}
    }
  }
  properties: {
    forceUpdateTag: utcValue
    azCliVersion: '2.58.0'
    timeout: 'PT10M'
    arguments: '\'${resourceGroupName}\' \'${name}\''
    scriptContent: 'result=$(az resource list --resource-group ${resourceGroupName} --name ${name});echo $result | jq -c \'{Result: map({name: .name, resourceGroup: .resourceGroup})}\' > $AZ_SCRIPTS_OUTPUT_PATH'
    cleanupPreference: 'Always'
    retentionInterval: 'PT1H'
  }
}

output sqlServerName array = (!empty(resource_exists_script.properties.outputs.Result)) ? resource_exists_script.properties.outputs.Result : []
```

### Conclusion

This code was used as part of a quickstart template and then triggered (and deployed successfully) in the following use cases:

- Single deployment invoking the QS with AADonly enabled
- Single deployment invoking the QS with AADonly disabled
- Multiple deployments invoking the QS in a loop with AADOnly enabled (which explains the uniqueId and name used in deployment names)
- Multiple deployments with existing resources and new resources

So basically, this should allow you to deploy however you want your SQLs, and it makes deployments compatible with the policies specified earlier on *Deny*.

I hope you'll find this code and solution helpful and that it will save people a lot of time.