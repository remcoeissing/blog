---
title: "Enabling Azure Policy for Kubernetes from Defender for Container with Infrastructure as Code"
layout: post
---
Recently got the question on how to enable certain environment settings in Microsoft Defender for Cloud. This post will be targeted at enabling the Azure Policy for Kubernetes feature and assumes that Defender for Containers itself is already enabled.

When we enable this in Azure what will happen under the hood is that two extra policies will be assigned at the subscription level. The two policy definitions are 'Deploy Azure Policy Add-on to Azure Kubernetes Service clusters' and 'Configure Azure Arc enabled Kubernetes clusters to install the Azure Policy extension'. But as we work with code we typically use the definition id's, which are `a8eff44f-8c92-45c3-a3fb-9880802d67a7` and `0adc5395-9169-4b9b-8687-af838d69410a`.

Before running the deployment:

![Before running the deployment](/assets/images/policy-for-kubernetes-pre.png)

## Bicep example

Putting this into a Bicep template is just like any other Policy Assignment that we create. However we keep the names and as these are deploy if not exists policies we also assign a System Managed Identity to the policy, without it we cannot deploy. If we would put this in a Bicep template, this would look like this:

```bicep
targetScope = 'subscription'

resource azurePolicyKubernetesArc 'Microsoft.Authorization/policyAssignments@2024-04-01' = {
  name: 'Defender for Containers provisioning Policy extension for Arc-e'
  location: deployment().location
  properties: {
    displayName: 'Configure Azure Arc enabled Kubernetes clusters to install the Azure Policy extension'
    description: 'Deploy Azure Policy\'s extension for Azure Arc to provide at-scale enforcements and safeguard your Arc enabled Kubernetes clusters in a centralized, consistent manner. Learn more at https://aka.ms/akspolicydoc.'
    policyDefinitionId: '/providers/Microsoft.Authorization/policyDefinitions/0adc5395-9169-4b9b-8687-af838d69410a'
    parameters: {
      effect: {
        value: 'DeployIfNotExists'
      }
    }
  }
  identity: {
    type: 'SystemAssigned'
  }
}

resource azurePolicyKubernetes 'Microsoft.Authorization/policyAssignments@2024-04-01' = {
  name: 'Defender for Containers provisioning Azure Policy Addon for Kub'
  location: deployment().location
  properties: {
    displayName: 'Deploy Azure Policy Add-on to Azure Kubernetes Service clusters'
    description: 'Use Azure Policy Add-on to manage and report on the compliance state of your Azure Kubernetes Service (AKS) clusters. For more information, see https://aka.ms/akspolicydoc.'
    policyDefinitionId: '/providers/Microsoft.Authorization/policyDefinitions/a8eff44f-8c92-45c3-a3fb-9880802d67a7'
    parameters: {
      effect: {
        value: 'DeployIfNotExists'
      }
    }
  }
  identity: {
    type: 'SystemAssigned'
  }
}
```

After running the deployment:

![After running the deployment](/assets/images/policy-for-kubernetes-post.png)

## Permissions

When we enable Azure Policy addon for Kubernetes via the portal you would see that it does not only create just plain Azure Policy assignments but also provides the identity the following role based permissions:

- Defender Kubernetes Agent Operator
- Kubernetes Agent Operator
- Log Analytics Contributor

So if we want to deploy this properly we should ensure that we grant these roles as well. We could extend the Bicep file with two additional resources that loop over an array of these 3 role definitions. As an alternative a module would also be a possibility, but for the simplicity of the example lets use two resources.

```bicep
var roleDefinitions = [
  '8bb6f106-b146-4ee6-a3f9-b9c5a96e0ae5'
  '5e93ba01-8f92-4c7a-b12a-801e3df23824'
  '92aaf0da-9dab-42b6-94a3-d43ce8d16293'
]

resource arcRoleAssignments 'Microsoft.Authorization/roleAssignments@2022-04-01' = [for roleDefinition in roleDefinitions: {
  name: guid(roleDefinition, 'defender for containers arc')
  properties: {
    principalId: azurePolicyKubernetesArc.identity.principalId
    roleDefinitionId: '/providers/Microsoft.Authorization/roleDefinitions/${roleDefinition}'
    principalType: 'ServicePrincipal'
  }
}]

resource azurePolicyRoleAssignments 'Microsoft.Authorization/roleAssignments@2022-04-01' = [for roleDefinition in roleDefinitions: {
  name: guid(roleDefinition, 'defender for containers')
  properties: {
    principalId: azurePolicyKubernetes.identity.principalId
    roleDefinitionId: '/providers/Microsoft.Authorization/roleDefinitions/${roleDefinition}'
    principalType: 'ServicePrincipal'
  }
}]
```

## Conclusion

When we combine the policy assignments and role assignments together we have successfully enabled the Azure Policy for Kubernetes feature of Defender for Containers. With this it has become possible to report and enforce the compliance of our Kubernetes clusters.

The full template I've used:

```bicep
targetScope = 'subscription'

var roleDefinitions = [
  '8bb6f106-b146-4ee6-a3f9-b9c5a96e0ae5'
  '5e93ba01-8f92-4c7a-b12a-801e3df23824'
  '92aaf0da-9dab-42b6-94a3-d43ce8d16293'
]

resource azurePolicyKubernetesArc 'Microsoft.Authorization/policyAssignments@2024-04-01' = {
  name: 'Defender for Containers provisioning Policy extension for Arc-e'
  location: deployment().location
  properties: {
    displayName: 'Configure Azure Arc enabled Kubernetes clusters to install the Azure Policy extension'
    description: 'Deploy Azure Policy\'s extension for Azure Arc to provide at-scale enforcements and safeguard your Arc enabled Kubernetes clusters in a centralized, consistent manner. Learn more at https://aka.ms/akspolicydoc.'
    policyDefinitionId: '/providers/Microsoft.Authorization/policyDefinitions/0adc5395-9169-4b9b-8687-af838d69410a'
    parameters: {
      effect: {
        value: 'DeployIfNotExists'
      }
    }
  }
  identity: {
    type: 'SystemAssigned'
  }
}

resource arcRoleAssignments 'Microsoft.Authorization/roleAssignments@2022-04-01' = [for roleDefinition in roleDefinitions: {
  name: guid(roleDefinition, 'defender for containers arc')
  properties: {
    principalId: azurePolicyKubernetesArc.identity.principalId
    roleDefinitionId: '/providers/Microsoft.Authorization/roleDefinitions/${roleDefinition}'
    principalType: 'ServicePrincipal'
  }
}]

resource azurePolicyKubernetes 'Microsoft.Authorization/policyAssignments@2024-04-01' = {
  name: 'Defender for Containers provisioning Azure Policy Addon for Kub'
  location: deployment().location
  properties: {
    displayName: 'Deploy Azure Policy Add-on to Azure Kubernetes Service clusters'
    description: 'Use Azure Policy Add-on to manage and report on the compliance state of your Azure Kubernetes Service (AKS) clusters. For more information, see https://aka.ms/akspolicydoc.'
    policyDefinitionId: '/providers/Microsoft.Authorization/policyDefinitions/a8eff44f-8c92-45c3-a3fb-9880802d67a7'
    parameters: {
      effect: {
        value: 'DeployIfNotExists'
      }
    }
  }
  identity: {
    type: 'SystemAssigned'
  }
}

resource azurePolicyRoleAssignments 'Microsoft.Authorization/roleAssignments@2022-04-01' = [for roleDefinition in roleDefinitions: {
  name: guid(roleDefinition, 'defender for containers')
  properties: {
    principalId: azurePolicyKubernetes.identity.principalId
    roleDefinitionId: '/providers/Microsoft.Authorization/roleDefinitions/${roleDefinition}'
    principalType: 'ServicePrincipal'
  }
}]
```
