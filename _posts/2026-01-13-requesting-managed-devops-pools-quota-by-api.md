---
title: "Requesting Managed DevOps Pools quota by API"
layout: post
---
A relatively new way of providing Azure Pipelines agents with private connectivity options inside an organization is through Managed DevOps. Managed DevOps allows you to create private connections to your Azure DevOps organization from within your virtual networks. This is done by creating a Managed DevOps resource in your subscription and linking it to your Azure DevOps organization. Once linked you can reach resources on the private network and secured with private endpoints.

When you get a new Azure subscription you have a default quota for how many VM's / agent you can create. Whenever you need more you can request to increase your quota through the Azure portal. This is also the case for Managed DevOps. But when you are automating the deployment of your subscriptions, also known as subscription vending, you might want to provision a Managed DevOps resource as well. This of course leads to the question of how to request a quota increase for Managed DevOps through an API, as we don't want to do this manually through the portal.

## Checking the current quota

Before we request a quota increase we first need to check what our current quota is. This can be done through the API. In this case we will use Azure PowerShell to make a GET request to the usage API.

```powershell
$response = Invoke-AzRestMethod -Method:Get -Uri 'https://management.azure.com/subscriptions/{subscriptionId}/providers/Microsoft.DevOpsInfrastructure/locations/{location}/usages?api-version=2024-04-04-preview'
$result = $response.Content | ConvertFrom-Json
$result.value | Select @{Name = "Family"; Expression = { $_.name.value}}, currentValue, limit
```

The result will be a table showing the current usage and limit for Managed DevOps pools. Within tis we can see the `standardBSFamily` where we have  limit of 5 but currently 0 in use.

```
Family                           currentValue limit
------                           ------------ -----
...
standardBSFamily                            0     5
...
```

## Requesting a quota increase

To request a quota increase we can make a PUT request to the Quota API. For the body of the request we need to specify the family we want to increase within the value, in the example this is `standardBSFamily`. Additionally we need to specify the new limit we want to have, in this case we will increase it to 10.

```powershell
$putBody = '{
    "properties": {
      "limit": {
        "limitObjectType": "LimitValue",
        "value": "10"
      },
      "name": {
        "value": "standardBSFamily"
      }
    }
  }'
$putRequest = Invoke-AzRestMethod -Method:Put -Uri 'https://management.azure.com/subscriptions/{subscriptionId}/providers/Microsoft.DevOpsInfrastructure/locations/{location}/providers/Microsoft.Quota/quotas/{family}?api-version=2021-03-15-preview' -Payload $putBody
```

This is all that it would take to raise a quota increase request for Managed DevOps Pools. You can check the status of your request by making a GET request to the URI in the `Location` header of the response. So for example:

```powershell
$quotaRequestStatus = Invoke-AzRestMethod -Method:Get -Uri $putRequest.Headers.Location.AbsoluteUri
$statusResult = $quotaRequestStatus.Content | ConvertFrom-Json
$statusResults
```

## Conclusion

With this simple API call you would be able to request a quota increase for Managed DevOps Pools in your subscription. This is especially useful when you are automating subscription vending and want to ensure that this can be done without manual intervention.
