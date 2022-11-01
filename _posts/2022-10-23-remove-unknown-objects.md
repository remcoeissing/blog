---
title: "Remove unknown objects from Azure Role Based Access Control"
layout: post
---
When working within your Azure environment you leverage Role Based Access Control, either on your Management Group, subscription, resource group or even on a resource level. Now due to a variety of reasons you eventually see a identity with the name *Identity not found*. So you now have a case where permissions are assigned to an identity that got removed; this typically happens in cases where users get removed from Azure Active Directory or for example a Managed Identity that got removed.

## Why should I care?

Because it's just so much nicer to work in an environment that's clean, you are simply less distracted. Also Role Based Access Control has it's [limits](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/azure-subscription-service-limits#azure-rbac-limits) just like many other things in Azure.

## How can I find these identities?

You could of course look through all your subscriptions etc. within the Azure Portal to identify any assignments that are unknown and clean them up. But an easier job would be to leverage automation to identify these objects. My personal favorite for these tasks is leveraging the Az PowerShell cmdlets, and in this case [Get-AzRoleAssignment](https://learn.microsoft.com/en-us/powershell/module/az.resources/get-azroleassignment?view=azps-9.0.0) specifically.

When we look at the documentation for this cmdlet it provides us with a nice clue on what we can leverage:

> Please notice that this cmdlet will mark ObjectType as Unknown in output if the object of role assignment is not found or current account has insufficient privileges to get object type.

So with this new bit of knowledge we can now write a script to get all our unknown objects, which would typically look like:

```powershell
Get-AzRoleAssignment | Where-Object {$_.ObjectType -eq 'Unknown'}
```

This will return all the objects with `ObjectType` of `Unknown` throughout the selected Azure subscription. So this includes all the assignments to resource group and resources as well.

## How can I remove them?

That's the easy part where [Remove-AzRoleAssignment](https://learn.microsoft.com/en-us/powershell/module/az.resources/remove-azroleassignment?view=azps-9.0.0) comes into play. And as it's the case when working with PowerShell objects we can just pipe the output of the get command to our remove command. You probably want to do something like this in a loop and write some logging information for the cases where you need to read back what you actually removed and when.

## How can we make this scale?

My personal favorite is to do this in a serverless way, so in my case I would leverage Azure Functions for executing this. It offers high scalability, reliability as well as many standard building blocks that I like to utilize. What our Function App needs to do:

- Get all subscriptions
- For each subscription get unknown role assignments. Log the assignment that it will remove and remove the actual assignment.

To run this the Function App needs to have a Managed Identity. The Managed Identity needs to have User Access Administrator rights on the subscriptions that it needs to clean.

I've opted to leverage PowerShell Durable Functions. I've opted for PowerShell as this is the language I see most people use for these kind of tasks, so that enables them to maintain it as well. For the durable functions I choose a [fan out / fan in pattern](https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-overview?tabs=csharp#fan-in-out).

It starts with the trigger and orchestrator. The orchestrator is rather simple; it invokes the `GetSubscriptions` activity and foreach subscription it invokes the `RemoveUnknownAssignments` activity and waits till all of them are completed.

`GetSubscriptions` connects to the Azure environment and will return all the subscriptions that the Managed Identity has access to.

`RemoveUnknownAssignments` selects the Azure subscription, fetches the unknown assignments and then removes them. This function also takes into account an Application Setting `whatif`; if this has a value of `1` then it will not process the actual removal but run in a log only mode.

### Configuring the Managed Identity with Graph permissions

I configure the Function App with a System Assigned Managed Identity so that this can be leveraged for authentication against other Azure services. By default the Managed Identity doesn't have any permissions to interact with the Microsoft Graph API, the API used for managing Azure Active Directory, so we need to provide those.

```powershell
# Connecting to the graph and requesting the scope to assign app roles.
Connect-MgGraph -Scopes Directory.ReadWrite.All, AppRoleAssignment.ReadWrite.All

# Get the Managed Identity from AAD
$MSI = Get-MgServicePrincipal -Filter "DisplayName eq 'rbaccleaner'"

# Get Microsoft Graph application
$graphApp = Get-MgServicePrincipal -Filter "appId eq '00000003-0000-0000-c000-000000000000'"

# Get graph permission
$permission = $graphApp.AppRoles | Where-Object { $_.Value -eq "Directory.Read.All" }

# Assign the permission to the Managed Identity
New-MgServicePrincipalAppRoleAssignment -AppRoleId $permission.Id -PrincipalId $MSI.Id -ServicePrincipalId $MSI.Id -ResourceId $graphApp.Id
```

In my case I assigned `Directory.Read.All`, `Directory.User.Read`. Keep in mind that if you don't have the right permissions set that all the role assignments will be marked as unknown and thus removed.

## Repository

An example of the code can be found here: [https://github.com/remcoeissing/azure-rbac-clean-ps](https://github.com/remcoeissing/azure-rbac-clean-ps)

It currently doesn't deploy any infrastructure components.

## What's next

In a future post will dive into the possibilities that we have available for doing this in an event driven fashion. That will feature a more direct approach where we don't rely on scheduling anymore. But it leverages the same basis.