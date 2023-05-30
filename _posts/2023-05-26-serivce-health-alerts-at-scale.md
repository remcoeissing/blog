---
title: "Service Health alert at scale"
layout: post
---
When we run our solution in Azure there will always be a time where there is an incident or maintenance event. As we are responsible for hosting our solutions we should get updates on these events so that we can take the right actions. One way that Azure provides these kind of notifications is through the use of [Service Health](https://learn.microsoft.com/en-us/azure/service-health/overview). I quite regularly run into workloads that haven't yet set this up and then experience challenges because they miss the events that have been published. This can of course be configured manually, but this will have to be done per subscription or resource group and thus it can be easy to miss.

## How to deploy this at scale?

The obvious answer for this is typically Azure Policies. They can stay around and as new subscriptions get added to the environment they will configure Service Health alerts as well. Luckily the Azure Community Policy already contains [Deploy Service Health Alerts and corresponding Action Group to notify of Service Health Incidents](https://github.com/Azure/Community-Policy/blob/master/Policies/Monitoring/deploy-service-health-alert-incidents/azurepolicy.json). When we enable this policy it will create an Action Group per subscription and setup Service Health alerts attached to it.

## How to set unique receivers per subscription?

One of the challenges with the current policy is that you can assign it on a management group level and it will configure the same e-mail receivers for all subscriptions. In many organizations different subscriptions have different technical owners or engineers that are responsible for this. One way many of them already do this is by setting a tag on each subscription with this information. So how can we leverage this. If we look at the current policy it has a parameter `emailAddress`, within the deployment part of the policy definition this is used like this:

```json
"emailAddress": {
    "value": "[parameters('emailAddress')]"
}
```

As we have set a tag TechnicalContact at the subscription level we could leverage this by altering this block to this:

```json
"emailAddress": {
    "value": "[subscription().tags['TechnicalContact']]"
}
```

Once this policy gets remediated the service health alerts will be sent to the technical contact.

## Conclusion

Setting up service health alerts is important to ensure workload teams don't miss out on important service issue, maintenance events etc.. Missing these could result in very short and difficult planning for these events, or time spent troubleshooting an issue. So many reasons to ensure that this gets configured at scale, which can easily be done with an Azure Policy as shown in this post.
