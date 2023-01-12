---
title: "Azure Policy the DevOps Way"
layout: post
---
Azure Policy is a great way to enforce or assess if the configuration of Azure services within Azure environment is compliant with standards set by the organization. Policies can be utilized in many ways; of course there are the builtin policies that are available on the platform and can just be assigned. Of course builtin policies will not always cover each and every scenario and you often have to create your own custom policies as well.

What happens a lot is that an engineer creates a policy, tests it, fine tune's it and then is done. A common use case is that we first do this in an isolated fashion on a single subscription or resource group level. When the engineer is happy it then gets applied onto a Management Group level for testing purposes. Which usually results in discovering some bugs, and after fixing those it will get applied to the production management group as well.

Most of the time there is some basic Infrastructure as Code and pieces of automation surrounding this. [Design Azure Policy as Code workflows](https://docs.microsoft.com/en-us/azure/governance/policy/concepts/policy-as-code) describes a nice proccess that creates a workflow covering the creation, testing and deployment of the Azure Policy. In this post I want to describe how such a proccess could look in practice by leveraging GitHub Actions for the automation. For the example we will create an Azure Policy that audits if there are undesired role assignments of type Owner on a subscription or resource group that has a certain tag.

## Creating a policy repo

Obviously I would first need a GitHub repository, I usually start with a local git repo.

## Creating a GitHub Actions workflow

When we create a GitHub Actions workflow we of course want to trigger it on a push into the main branch. And then we want to deploy the policies and apply the test assignments.

```yml
name: policy-deployment

on:
  push:
    branches:
      - main

jobs:
  apply-azure-policy:
    runs-on: ubuntu-latest
    name: Deploy policy definitions + test assignments
    steps:
    - name: Checkout
      uses: actions/checkout@v2
    - name: Login to Azure
      uses: azure/login@v1
      with:
        creds: ${{secrets.AZURE_CREDENTIALS}}
        allow-no-subscriptions: true
    - name: Create or update Azure Policies
      uses: azure/manage-azure-policy@v0
      with:
        paths: |
          policies/rbac-owner/**
        assignments: |
          assign.test.*.json
```

The above workflow contains a single job that will apply the policy definition and apply all the assignments that are in files following the `assign.test.*.json` pattern. So a file with `assign.dev.rbac-owner-disallowed.json` will be applied, but the file named `assign.prd.rbac-owner.json` won't get applied as it doesn't match with the pattern.

## Testing the policy

Most of the DevOps teams that I have met so far are testing these policies manually and test them for the change they want to make. Over the last years I've seen many of them experience regressions, where a change in the policy had a bunch of unintended side effects.

If we take these previous thoughts into consideration a DevOps process would facilitate some way of automated testing. For a policy that would mean a few things;

- For an audit policy we deploy resources that satisfy and that don't satifsy the policy.
- A deny policy should be tested by trying to deploy resources and validate if they are really denied. But we also want to see a _good_ resource go through succesfully.
- For a modify policy I like to see that it takes the right actions, but also that it doesn't fiddle or conflict with valid deployments.

For all of these we would have to follow a similar process:
- Deploy Azure resources on which we want to test
- Run a Policy Compliance Scan
- Evaluate the output of the policy compliance scan.

For the rest of the example I will use an Audit policy. For the other policy types some of the test steps will be slightly different.

### Deploying test resources

Deploying the test resources for me is usually a dedicated stage. This stage deploys a Bicep, ARM, Terraform template or any other mechanism that you like for creating these resources.

```yml
  deploy-test-resources:
    runs-on: ubuntu-latest
    name: Deploy test resources
    needs: apply-azure-policy
    steps:
    - name: Checkout
      uses: actions/checkout@v2
    - name: Login to Azure
      uses: azure/login@v1
      with:
        creds: ${{secrets.AZURE_CREDENTIALS}}
    - name: Deploy
      uses: azure/arm-deploy@v1
      with:
        scope: subscription
        subscriptionId: ${{secrets.AZURE_SUBSCRIPTION}}
        template: ./tests/deployment/rbac-owner.bicep
        region: westeurope
        failOnStdErr: false
```

### Testing your policies

Now that we test resources deployed and our policy assigned we can take a look at testing our policy. Typically you would run a policy scan and then look at the output, but we will trigger this scan in the next step already. We will use the `Azure/policy-compliance-scan` action in GitHub. This will provide us with a CSV reporting the compliance of our resources. This CSV is great input for running some automated test cases against.

I could pick a number of testing frameworks out there, in this case I would leverage Pester; so would write it in PowerShell. My reasoning behind this is that most of the time policies are written and maintained by engineers that are already familiar with PowerShell.

```powershell
Describe "Check disallowed resource group owners" {
    BeforeAll {
        $results = Import-Csv policy-compliance.csv
        $result = $results | Where-Object { $_.POLICY_DEF_ID.endswith('audit-rbac-owner') -and $_.POLICY_ASSG_ID.Contains('/resourcegroups/rg-disallowed-owner') }
    }

    It "Should have one result" {
        $result | Should -Not -BeNullOrEmpty
        $result.Count | Should -Be 1
    }

    It "Should be resource group rg-disallowed-owner" {
        $result.RESOURCE_ID.Contains('rg-disallowed-owners/providers/microsoft.authorization/') | Should -Be $true
    }

    It "Should be non compliant" {
        $result.COMPLIANCE_STATE | Should -Be 'NonCompliant'
    }
}
```

In the example above we run a validation on a single resource group in the describe. In the `BeforeAll` section we grab the results for this particular resource group, as we only want to validate that single resource group.

Then the different cases will be validating that our policy behaved as expected:

1. We should only have a single entry for our policy for the single resource group. So we validate that the count of the results is `1`.
2. Validate that the resource id of the result is actually the resource that we expect.
3. As the deployed resource should be non-compliant we validate the compliance state.

### Automating the tests

Of course we can run all these steps using GitHub Actions.

```yml
  test-policy:
    runs-on: ubuntu-latest
    name: Test policies on resources
    needs:
      - apply-azure-policy
      - deploy-test-resources
    steps:
    - name: Login to Azure
      uses: azure/login@v1
      with:
        creds: ${{secrets.AZURE_CREDENTIALS}}
    - name: Azure Policy Compliance Scan
      uses: Azure/policy-compliance-scan@v0
      continue-on-error: true
      with:
        scopes: |
          /subscriptions/${{secrets.AZURE_SUBSCRIPTION}}/resourceGroups/rg-disallowed-owners
        policy-assignments-ignore: |
          /subscriptions/${{secrets.AZURE_SUBSCRIPTION}}/providers/microsoft.authorization/policyassignments/*
        wait: true
        report-name: policy-compliance
    - name: Checkout
      uses: actions/checkout@v2
    - name: Download policy compliance artifact
      uses: actions/download-artifact@v2
      with:
        name: policy-compliance.csv
    - name: Test policy compliance results file
      shell: pwsh
      run: Test-Path policy-compliance.csv | Should -Be $true
    - name: Analyze policy compliance results
      shell: pwsh
      run: |
        Invoke-Pester ./tests/PolicyCompliance.Tests.ps1 -Passthru
```

## Promoting the policy

Once that all the test have completed succesfully our policy should be consided safe to be promoted to production. As we already have the definition there it can be just a matter of assigning it to the next resources. In general I would prefer to do this at the Management Group level.

For assigning them I leverage another GitHub Actions Job. This job will just look at the assignments that are described in `assign.prd.*.json`. An example of such an assignment would be:

```json
{
    "sku": {
     "name": "A0",
     "tier": "Free"
    },
    "properties": {
     "displayName": "No unknown RBAC owners allowed on subscription and resource groups",
     "policyDefinitionId": "/subscriptions/[subscriptionid]/providers/Microsoft.Authorization/policyDefinitions/audit-rbac-owner",
     "scope": "/subscriptions/[subscriptionid]",
     "notScopes": [],
     "parameters": {},
     "metadata": {
      "assignedBy": "Remco Eissing"
     },
     "enforcementMode": "Default"
    },
    "id": "/subscriptions/[subscriptionid]/resourceGroups/rg-disallowed-owners/providers/Microsoft.Authorization/policyAssignments/audit-rbac-owner",
    "type": "Microsoft.Authorization/policyAssignments",
    "name": "audit-rbac-owner",
    "location": "westeurope"
   }
```

The deployment job would look like below. After this job has ran it will have assigned the policies at the production scopes.

```yml
  assign-azure-policy-production:
    runs-on: ubuntu-latest
    name: Assign policy to production
    needs: test-policy
    steps:
    - name: Checkout
      uses: actions/checkout@v2
    - name: Login to Azure
      uses: azure/login@v1
      with:
        creds: ${{secrets.AZURE_CREDENTIALS}}
        allow-no-subscriptions: true
    - name: Create or update Azure Policies
      uses: azure/manage-azure-policy@v0
      with:
        paths: |
          policies/**/**
        assignments: |
          assign.prd.*.json
```

## Cleaning up

We've now seen how we can automatically test a policy and promote it to production. We of course want to be cost effective here as well; so we should clean up all our testing resources in an automated fashion. Using GitHub Actions this can be a simple extra job, that looks something like this:

```yml
  remove-test-resources:
    runs-on: ubuntu-latest
    name: Remove test resources
    needs: test-policy
    steps:
    - name: Checkout
      uses: actions/checkout@v2
    - name: Login to Azure
      uses: azure/login@v1
      with:
        creds: ${{secrets.AZURE_CREDENTIALS}}
    - name: Remove
      uses: azure/CLI@v1
      with:
        azcliversion: 2.0.72
        inlineScript: |
          resources="$(az resource list --resource-group rg-policy-test | grep id | awk -F \" '{print $4}')"
          for id in $resources; do
              az resource delete --resource-group rg-policy-test --ids "$id" --verbose
          done
```

## Conclusion

There is lots of room for improvement to learn from regular development processes when working with Azure Policies. This post is just meant as a first starter for some inspiration.
