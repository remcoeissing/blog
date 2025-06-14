---
title: "Updating Key Vault references in an App Service"
layout: post
---
A popular way to utilize secrets stored in Key Vault within an App Service is by utilizing a Key Vault reference in the configuration of the App Service. This allows you to create an app setting with a name of `ExampleSecret` and then specify a value that points to a secret stored in Key Vault `@Microsoft.KeyVault(SecretUri=https://kv-kvreference.vault.azure.net/secrets/example-secret)`. In your web application you can then simply fetch the value of `ExampleSecret` and you have the actual secret from Key Vault. For more details and introduction on Key Vault references: [Use Key Vault references as app settings in Azure App Service and Azure Functions](https://learn.microsoft.com/en-us/azure/app-service/app-service-key-vault-references?tabs=azure-cli)

## Initial configuration

In order for the App Service to fetch the secret from Key Vault it would need to have access. This is typically done by having a System Assigned Managed Identity on the App Service. You can then provide this identity access to read secrets from Key Vault. A personal preference is to use the Azure Role-Based Access Control option, an example would look something like this:

```bicep
resource keyVault 'Microsoft.KeyVault/vaults@2023-02-01' = {
  name: 'kv-${solutionName}'
  location: location
  properties: {
    sku: {
      family: 'A'
      name: 'standard'
    }
    tenantId: subscription().tenantId
    enableRbacAuthorization: true
    enabledForDeployment: true
  }
}

resource appServicePlan 'Microsoft.Web/serverfarms@2022-09-01' = {
  name: 'asp-${solutionName}'
  location: location
  sku: {
    name: 'F1'
    tier: 'Free'
    size: 'F1'
    capacity: 1
  }
}

resource webApp 'Microsoft.Web/sites@2022-09-01' = {
  name: 'web-${solutionName}'
  location: location
  properties: {
    serverFarmId: appServicePlan.id
    httpsOnly: true
  }
  identity: {
    type: 'SystemAssigned'
  }
}

resource keyVaultSecretReadAccess 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(webApp.id, 'Key Vault Secrets Reader')
  properties: {
    roleDefinitionId: resourceId('Microsoft.Authorization/roleDefinitions', '4633458b-17de-408a-b874-0445c86b69e6') // Key Vault Secrets User
    principalId: webApp.identity.principalId
  }
}
```

Within the Key Vault you can then store a secret named `example-secret`. Within a .NET web application you could then refer to it using `_configuration["ExampleSecret"]` as the setting on the App Service is called `ExampleSecret`. This would make you web application look like this:

![Web application with the initial version of the Key Vault Secret](/assets/images/key-vault-reference-initial.png)

## Updating the secret to a new value

Now it's time to change the value of our secret, let's update it to `Updated super secret value`. When you now refresh our web application you still see the initial value. That wasn't really the plan...

You now have the option to wait till the secret gets updated somewhere within the 24-hour window. Alternatively you could make a call to `https://management.azure.com/[Resource ID]/config/configreferences/appsettings/refresh?api-version=2022-03-01` to refresh the secrets. That already sounds a lot better! To make it easy you would typically call this using either `Invoke-AzRestMethod` in PowerShell or `az rest` in Azure CLI. This way you don't have to deal with authentication yourself.

```azure-powershell
Invoke-AzRestMethod -Method:POST -Uri https://management.azure.com/subscriptions/[subscription id]/resourceGroups/rg-kv-reference/providers/Microsoft.Web/sites/web-kvreference/config/configreferences/appsettings/refresh?api-version=2022-03-01
```

```azure-cli
az rest --method POST --url /subscriptions/[subscription id]/resourceGroups/rg-kv-reference/providers/Microsoft.Web/sites/web-kvreference/config/configreferences/appsettings/refresh?api-version=2022-03-01
```

After executing this you get back a response summarizing which secrets have been refreshed and if this was successful:

```json
{
  "id": null,
  "nextLink": null,
  "value": [
    {
      "id": "/subscriptions/[subscription id]/resourceGroups/rg-kv-reference/providers/Microsoft.Web/sites/web-kvreference/config/configreferences/appsettings/ExampleSecret",
      "location": "Sweden Central",
      "name": "ExampleSecret",
      "properties": {
        "activeVersion": null,
        "details": "Reference has been successfully resolved.",
        "identityType": "SystemAssigned",
        "reference": "@Microsoft.KeyVault(SecretUri=https://kv-kvreference.vault.azure.net/secrets/example-secret)",
        "secretName": "example-secret",
        "status": "Resolved",
        "vaultName": "kv-kvreference"
      },
      "type": "Microsoft.Web/sites/config/configreferences"
    }
  ]
}
```

When you now refresh your web application you will get the updated value:

![Web application with the updated version of the Key Vault Secret](/assets/images/key-vault-reference-update.png)

## Conclusion

This way you can tell your App Service to refresh the secrets when you update them in Key Vault. This is beneficial for systems that use a single secret at a time. If you for example have a primary key and a secondary key you can safely wait for the 24-hour rollover period. Whenever that's not the case you now have an Azure CLI and Azure PowerShell command to force the refresh, we could of course embed this inside of a GitHub Actions workflow or Azure Pipelines.
