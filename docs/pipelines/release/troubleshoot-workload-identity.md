---
title: Troubleshoot Azure Resource Manager workload identity service connections
ms.custom: devx-track-arm-template, arm2024
description: How to troubleshoot workload identity service connections in Azure Pipelines
ms.topic: troubleshooting-general
ms.author: jukullam
author: juliakm
ms.date: 02/08/2024
monikerRange: '>= azure-devops'
"recommendations": "true"
---


# Troubleshoot Azure Resource Manager workload identity service connections

Get help debugging common issues with workload identity service connections and learn how to manually create a service connection if necessary.

## Troubleshooting checklist

### Review pipeline tasks

Not all pipelines tasks support workload identity. During the preview, no Azure Marketplace tasks support workload identity service connections. The following tasks don't currently support with workload identity federation.

- `AzureCloudPowerShellDeploymentV1`
- `AzCopy` (`AzureFileCopyV1`, `AzureFileCopyV2`, `AzureFileCopyV3`, `AzureFileCopyV4`, `AzureFileCopyV5`)
- `AzureMonitorAlertsV0`
- `AzureNLBManagementV1`
- `PackerBuild` (`PackerBuildV0`, `PackerBuildV1`)
- `ServiceFabricComposeDeployV0`
- `ServiceFabricDeployV1`	

### Verify workload identity federation is active

If you see error message `AADSTS700223` or `AADSTS700238`, workload identity federation was disabled in your Microsoft Entra tenant. 

Verify that there are no Microsoft Entra policies in place that block federated credentials.

### Check the issuer URL for accuracy

If you see a message that there's `no matching federated identity record found`, either the issuer URL or federation subject doesn't match. The correct issuer URL show start with `https://vstoken.dev.azure.com`. 

You can fix the issuer URL by editing and saving the service connection to update the issuer URL. The issuer needs to be updated manually if Azure DevOps didn't create the identity. For Azure identities, the issuer URL automatically updates.  


## Common issues and causes

<a name='i-dont-have-permission-to-create-a-service-principal-in-the-azure-entra-id-tenant'></a>

### I don't have permission to create a service principal in the Azure Microsoft Entra tenant

You can't use the service connection configuration tool when you don't have the correct permissions. Your permission level is inadequate if you don't have permission to create service principals or you're using a different Azure Microsoft Entra tenant than your Azure DevOps user. 

You need to either have permission in Microsoft Entra ID to create app registrations or you need to have an appropriate role (example: Application Developer).

You have two options:

* [Solution 1: Manually configure workload identity with managed identity](configure-workload-identity.md#workload-identity-with-managed-identity)
* [Solution 2: Manually configure workload identity with a service principal](configure-workload-identity.md#workload-identity-with-a-service-principal)

## Error messages

| Message | Plausible issue |
|---------|-----------------|
| *cannot request token: Get `?audience=api://AzureADTokenExchange: unsupported protocol scheme`* | Task does not support workload identity federation. |
| *Identity not found* | Task doesn't support workload identity federation. |
| *Could not fetch access token for Azure* | Task doesn't support workload identity federation. |
| *AADSTS700016: Application with identifier '****' wasn't found* | The identity used for the service connection no longer exists, it may have been removed independent from the service connection. You'll need to create a new service connection. |
| *AADSTS7000215:  Invalid client secret provided.* | You're using a service connection with a secret that expired. [Convert the service connection to Workload identity federation](https://aka.ms/azdo-rm-workload-identity-conversion) and replace the expired secret with a federation. |
| *AADSTS700024: Client assertion is not within its valid time range* | This can happen in the following cases:<ul><li> You're using the AzureCLI task with `addSpnToEnvironment: true` to consume the `idToken` environment variable. The `idToken` has expired after 10 minutes.</li><li>Some Azure data plane (non-ARM) operations require a separate bearer token to authenticate. A bearer token is requested with the Azure CLI using `az account get-access-token` or PowerShell Az using `Get-AzAccessToken`. These tokens have a lifetime of 1 hour. Using the token after 1 hour results in `AADSTS700024`. Some tools and SDKs (for example, [Azure GO SDK](https://github.com/Azure/azure-sdk-for-go), [Azure Python SDK](/azure/developer/python/sdk/azure-sdk-overview)) use Azure CLI and `az account get-access-token` indirectly to obtain a bearer token. If you have tasks that (directly or indirectly) obtain a bearer token and run longer than 1 hour, use a service connection with a secret instead.</li> |
| *AADSTS70021: No matching federated identity record found for presented assertion. Assertion Issuer: `https://app.vstoken.visualstudio.com`.* | The issuer URL isn't correct. The correct issuer URL is `https://vstoken.dev.azure.com/XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX`. You can fix the issuer URL by editing and resaving a service connection. You need to manually update the issuer if Azure DevOps didn't create your identity. You can find the correct issuer in the edit dialog of the service connection, or in the response (under authorization parameters) when using the REST API. |
| *AADSTS70021: No matching federated identity record found for presented assertion. Assertion Issuer:  `https://vstoken.dev.azure.com/XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX`. Assertion Subject: `sc://\<org\>/\<project\>/\<service-connection\>.`* | Either the issuer url or federation subject doesn't match. The Azure DevOps organization or project was renamed, or a manually created service connection was renamed without updating the federation subject on the identity. |
| *AADSTS700211: No matching federated identity record found for presented assertion issuer* | The issuer url and/or federation subject is rejected by an Entra ID policy. |
| *AADSTS700223* | Workload identity federation is constrained or disabled on the Microsoft Entra tenant. In these cases it is often possible to use a Managed Identity for the federation instead, see [Workload identity with managed identity](https://aka.ms/azdo-rm-workload-identity-manual). |
| *Microsoft Entra rejected the token issued by Azure DevOps with error code AADSTS700238* | Workload identity federation has been constrained on the Microsoft Entra tenant. The issuer for your organization (`https://vstoken.dev.azure.com/XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX`) isn't allowed to use Workload identity federation. Please contact your Entra tenant administrator(s) to allow Workload identity federation for your Azure DevOps organization. |
|*Failed to obtain the Json Web Token(JWT) using service principal client ID*|Your federation identity credential is misconfigured or the Microsoft Entra tenant blocks OIDC.|
| *Script failed with error: UnrecognizedArgumentError: unrecognized arguments: --federated-token* | You're using the AzureCLI task on an agent that has an old version of the Azure CLI installed. Workload identity federation requires Azure CLI 2.30 or higher. |
| *Failed to create an app in Microsoft Entra ID. Error: Insufficient privileges to complete the operation in Microsoft Graph. Ensure that the user has permissions to create a Microsoft Entra Application.* | The ability to create App Registrations was [disabled](/azure/active-directory/roles/delegate-app-roles#restrict-who-can-create-applications) in the Microsoft Entra tenant. Assign the user creating the service connection to the [Application Developer](/azure/active-directory/roles/permissions-reference#application-developer) Microsoft Entra role. Alternatively, create the service connection manually with a Managed Identity. See [Workload identity with managed identity](https://aka.ms/azdo-rm-workload-identity-manual). |

### I use a Container Resource that specifies an Azure Container Registry

[Container resources](/azure/devops/pipelines/process/resources?view#define-a-containers-resource) that are pulled from Azure Container Registry don't support a workload identity federation service connection specified in `azureSubscription`.