---
title: "Remove Microsoft-Hosted Agents from an Azure DevOps Project"
layout: post
---
As part of Azure DevOps the Azure Pipelines allow you to run automated builds and deployments for your applications. You can use different types of agents to run your pipelines, such as hosted agents, self-hosted agents, or virtual machine scale sets. Agents are grouped into agent pools, which define the configuration and capabilities of the agents. Agent pools are associated with agent pool queues, which are used to assign work to the agents in the pool.

Sometimes, you may need to remove an agent pool queue from an Azure DevOps project, for example, if you know that the project cannot execute builds and deployments on Microsoft-hosted agents. This typically occurs when there are security requirements and the network is locked down. In this post, I will show you how to manually remove an agent pool queue from an Azure DevOps project, as well as how to use the Azure DevOps REST API to automate the process. I will also show you how to restore an agent pool queue using the Azure DevOps REST API, in case you need to undo the removal.

## How to manually remove an agent pool queue from an Azure DevOps project

The manual method to remove an agent pool queue from an Azure DevOps project involves deleting the agent pool that is associated with the queue. This will also delete the queue from the project, but not from the organization. To do this, follow these steps:

1. Log in to your Azure DevOps portal and navigate to the project that contains the agent pool queue that you want to remove.
2. Go to Project settings > Pipelines > Agent pools.
3. Hover over the agent pool you like to delete. You will see a delete button on the right and can click on it.
4. You get a confirmation if you want to delete the queue from the project. Confirm it if you are sure.
5. Refresh the page and verify that the agent pool and the queue are no longer listed in the project.

Note that this method will delete the agent pool and the queue from the project, if required you can of course add it back again.

## How to remove an agent pool queue from an Azure DevOps project using the Azure DevOps REST API

The Azure DevOps REST API allows you to programmatically interact with Azure DevOps services, such as projects, pipelines, agents, and queues. You can use the Azure DevOps REST API to remove an agent pool queue from an Azure DevOps project without deleting the agent pool. To do this, you will need to use the following endpoints:

- GET https://dev.azure.com/{organization}/_apis/distributedtask/queues?api-version=6.0-preview.1 to get the list of agent pool queues in the organization and their IDs.
- DELETE https://dev.azure.com/{organization}/_apis/distributedtask/queues/{queueId}?api-version=6.0-preview.1 to delete the agent pool queue from the organization.

To use the Azure DevOps REST API, you will need to authenticate with an access token or a personal access token (PAT). However, for security reasons, I recommend using the Get-AzAccessToken cmdlet from the Az.Accounts module to get an access token for Azure DevOps. This cmdlet will use the credentials of the current Azure session to generate an access token that can be used to call the Azure DevOps REST API. To use the Get-AzAccessToken cmdlet, you will need to connect to your Azure account that contains your Azure DevOps organization. Once you have connected to your Azure account, you can use the Get-AzAccessToken cmdlet to get an access token for Azure DevOps. To do this, run the following command in PowerShell: `$token = Get-AzAccessToken -ResourceUrl 499b84ac-1321-427f-aa17-267ca6975798`. Where `499b84ac-1321-427f-aa17-267ca6975798` is the well known guid of Azure DevOps. I do the same in the below example script.

In the below script I first get the access token for Azure DevOps using the Get-AzAccessToken cmdlet. Then I use the access token to call the Azure DevOps REST API to get the list of agent pool queues in the organization. I filter the list of queues by name to find the queue that I want to remove. If the queue is present, I get its ID and use it to call the DELETE endpoint to remove the queue from the organization. Here is the complete script in PowerShell:

```powershell
$organization = 'your-organization-name'
$project = 'your-project-name'
$queueName = 'Azure Pipelines'

# Get access token for Azure DevOps
$accessToken = Get-AzAccessToken -ResourceUrl '499b84ac-1321-427f-aa17-267ca6975798'

$headers = @{
    Authorization = "Bearer $($accessToken.Token)"
    Accept        = 'application/json'
}

$getQueuesParameters = @{
    Method  = 'Get'
    Uri     = "https://dev.azure.com/$organization/$project/_apis/distributedtask/queues?queueNames=$queueName&api-version=7.2-preview.1"
    Headers = $headers
}

$getQueuesResponse = Invoke-RestMethod @getQueuesParameters

if ($getQueuesResponse.count -eq 1) {
    $queueId = $getQueuesResponse.value[0].id

    $removeParameters = @{
        Method  = 'Delete'
        Uri     = "https://dev.azure.com/$organization/$project/_apis/distributedtask/queues/$queueId/?api-version=7.2-preview.1"
        Headers = $headers
    }

    $response = Invoke-RestMethod @removeParameters

    Write-Host 'Queue deleted'
} else {
    Write-Host 'Queue not present'
}
```

## How to restore an agent pool queue to an Azure DevOps project using the Azure DevOps REST API

If you accidentally deleted an agent pool queue from an Azure DevOps project or organization, you can use the Azure DevOps REST API to restore it. To do this, you will need to use the following endpoints:

-	GET https://dev.azure.com/{organization}/_apis/distributedtask/queues?api-version=6.0-preview.1 to get the list of agent pool queues in the organization and their IDs.
-	GET https://dev.azure.com/{organization}/_apis/projects?api-version=6.0 to get the list of projects in the organization and their IDs.
-	POST https://dev.azure.com/{organization}/_apis/distributedtask/queues?api-version=6.0-preview.1 to create a new agent pool queue in the organization and associate it with a project.

In the script below I first get the access token for Azure DevOps using the Get-AzAccessToken cmdlet. Then I use the access token to call the Azure DevOps REST API to get the list of agent pool queues in the organization. I filter the list of queues by name to find the queue that I want to restore. If the queue is present, I get its ID and use it to call the POST endpoint to create a new agent pool queue in the organization and associate it with a project. Here is the complete script in PowerShell:

```powershell
$organization = 'your-organization-name'
$project = 'your-project-name'
$poolName = 'Azure Pipelines'

# Get access token for Azure DevOps
$accessToken = Get-AzAccessToken -ResourceUrl '499b84ac-1321-427f-aa17-267ca6975798'

$headers = @{
    Authorization = "Bearer $($accessToken.Token)"
    Accept        = 'application/json'
    "Content-Type"  = 'application/json'
}

$getPoolParameters = @{
    Method  = 'Get'
    Uri     = "https://dev.azure.com/$organization/_apis/distributedtask/pools?poolName=$queueName&api-version=7.2-preview.1"
    Headers = $headers
}

$getPoolResponse = Invoke-RestMethod @getPoolParameters

if ($getPoolResponse.count -eq 1) {
    $poolId = $getPoolResponse.value[0].id

    $body = @{
        Name = $poolName
        Pool = @{
            Id = $poolId
        }
    }

    $addParameters = @{
        Method  = 'Post'
        Uri     = "https://dev.azure.com/$organization/$project/_apis/distributedtask/queues?api-version=7.2-preview.1"
        Headers = $headers
        Body    = $body | ConvertTo-Json
    }

    $response = Invoke-RestMethod @addParameters

    Write-Host 'Queue added'
} else {
    Write-Host 'Pool not found'
}
```

You can verify that the agent pool queue is restored to the project by using the GET endpoint again or by going to Project settings > Pipelines > Agent pools in the Azure DevOps portal.

## Conclusion

In this post, I showed you how to remove and restore an agent pool queue from an Azure DevOps project using manual and automated methods. I hope that this guide will help you to manage your agent pool queues more easily and efficiently and above all to ensure that your project meets your compliance requirements.
