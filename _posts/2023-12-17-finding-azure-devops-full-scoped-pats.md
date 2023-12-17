---
title: "Finding Azure DevOps full scoped PATs"
layout: post
---
Personal Access Tokens (PATs) are a way of authenticating and accessing Azure DevOps resources without using a username and password. They can be scoped to limit the access level and duration of the token, and they can be revoked at any time. However, PATs also pose a security risk if they are not managed properly, especially if they have full scope permissions.

Full scope PATs grant unrestricted access to all Azure DevOps organizations, projects, and resources that the user has access to. This means that anyone who obtains a full scope PAT can potentially perform any action on behalf of the user, such as deleting repositories, modifying pipelines, accessing secrets, and so on. Full scope PATs should only be used for specific scenarios where no other scope is sufficient, and they should be treated with extreme caution.

Azure DevOps offers the possibility to authenticate processes using a Managed Identity or Service Principal. This reduces the need for running automation processes with a PAT token.

## How to prevent these from being created?

Even though using Managed Identity or Service Principal is a recommended practice, there might be scenarios where using a PAT token is necessary or more convenient. In that case, it is important to restrict the scope and duration of the PAT token as much as possible, to reduce the risk of unauthorized access or misuse.

One way to restrict PAT tokens is by applying policies in Azure DevOps / Microsoft Entra ID. Policies can help enforce certain rules or standards for creating and using PAT tokens, such as:

- Require a maximum lifetime for PAT Personal Access Tokens
- Restrict creation of global Personal Access Tokens
- Restrict creation of full-scoped Personal Access Tokens

To apply policies for Personal Access Tokens, you need to have an organization associated to Microsoft Entra ID. And as user doing the configuration you need to have been assigned the Azure DevOps Administrator role in Microsoft Entra ID. If that’s all in place you can go to the organization settings and navigate to Microsoft Entra.

Keep in mind that these policies will only apply to newly created PATs, so any existing PATs will remain unaffected.

## How to list all full scope PATs?

One way to list all the full scope PATs in your organization is to use a PowerShell script that can query the Azure DevOps API. The script will require an administrator to run it. Luckily we don’t need to have a PAT to authenticate and access the data, instead we can use an Access Token.

This script will fetch all users and query foreach user if they have any PAT, if they do it will validate the scope of the token. If the scope matches it will write an entry to the results. So all the results will be users with a full scope PAT. If there is no value for targetAccounts it’s even worse, this means that the PAT is valid for all organizations that the user belongs to.

```powershell
param(
    [String]
    $OrganizationName,

    [String]
    $TenantId,

    [String]
    $Scope = 'app_token'
)

$AccessToken = Get-AzAccessToken -ResourceUrl "499b84ac-1321-427f-aa17-267ca6975798" -TenantId $TenantId
$Headers = @{
    Accept = "application/json"
    Authorization = "Bearer $($accessToken.Token)"
}

function GetUsers
{
    $UsersUrl = "https://vssps.dev.azure.com/$OrganizationName/_apis/graph/users?api-version=7.2-preview.1"
    $UsersResult = Invoke-RestMethod -Method:Get -Uri $UsersUrl -Headers $Headers

    return $UsersResult.value
}

$Users = GetUsers
$FilteredUsers = $Users | Where-Object {$_.domain -eq $TenantId}

$Results = @()

foreach($User in $FilteredUsers)
{
    $PatUrl = "https://vssps.dev.azure.com/$OrganizationName/_apis/tokenadmin/personalaccesstokens/$($User.descriptor)?api-version=6.0-preview.1"
    $PatResult = Invoke-RestMethod -Method:Get -Uri $PatUrl -Headers $Headers

    if ([String]::IsNullOrEmpty($PatResult.value))
    {
        Write-Verbose "No PAT found for $($User.displayName)"
    }
    else
    {
        $PatResult.value | Where-Object {$_.scope -eq $Scope} | ForEach-Object {
            Write-Verbose "PAT found for $($User.displayName) with scope $($_.scope)"

            $Results += [PSCustomObject] @{
                User = $User.displayName
                Scope = $_.scope
                TargetAccounts = $_.targetAccounts
                ExpirationDate = $_.validTo
            }
        }
    }
}

$Results | Format-Table
```

The script will generate an output like this:

```powershell
User  Scope     TargetAccounts ExpirationDate
----  -----     -------------- --------------
alice app_token {some guid}    1/14/2024 12:00:00 AM
alice app_token                1/14/2024 12:00:00 AM
```

## Conclusion

In this document, I have shown you how to list all the full scope PATs in your organization using a PowerShell script and the Azure DevOps API. This can help you identify and revoke any unnecessary or risky tokens that could compromise your security. I have also taught you how to limit the scope of a PAT, and why this is crucial to protect your data and resources. I recommend that you review your PATs regularly and follow the best practices for creating and managing them. By doing so, you can enhance the security and compliance of your Azure DevOps environment.
