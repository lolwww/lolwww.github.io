---
layout: post
title: How to use Juju with Microsoft Azure
---

Based on the [Juju documentation](https://juju.is/docs/juju/microsoft-azure) and personal experience.

Juju version used is 3.6-beta.

As Juju documentation describes, there are 3 ways to make Juju work with Azure
(for it to be able create, delete, modify resources on Azure):

1. **pre-created [Managed Identity](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview) with managed-identity credentials** (recommended by Juju team, although only available from Juju 3.6-beta).
2. **pre-created Managed Identity + service-principal-secret credentials**
   
3. **service-principal-secret credentials, Managed Identity created on bootstrap** (similar to #2 in the end, but not exactly during the process).

4. Undocumented Entra ID app registrations way (arguably most commonly used).

In this guide I will try to attempt to document all 3 ways and see which one works better.

For Options 1 and 2 both we will be required to create Managed Identity first, so let's start with this part.

**Creating Managed Identity**

We will try to follow the suggested documentation there and use Azure cli(Bash).
Let's start from scratch as we don't have any resource groups, roles etc.

```
export group=jujuclitest
export location=eastus
export role=jujuclitest
export identityname=jujuclitest
export subscription=xxxxx
```

```
1) az group create --name "${group}" --location "${location}"
2) az identity create --resource-group "${group}" --name "${identityname}"
3) mid=$(az identity show --resource-group "${group}" --name "${identityname}" --query principalId --output tsv)
4) az role definition create --role-definition "{
  	\"Name\": \"${role}\",
  	\"Description\": \"Role definition for a Juju controller\",
  	\"Actions\": [
            	\"Microsoft.Compute/*\",
            	\"Microsoft.KeyVault/*\",
            	\"Microsoft.Network/*\",
            	\"Microsoft.Resources/*\",
            	\"Microsoft.Storage/*\",
            	\"Microsoft.ManagedIdentity/userAssignedIdentities/*\"
  	],
  	\"AssignableScopes\": [
        	\"/subscriptions/${subscription}\"
  	]
  }"
5) az role assignment create --assignee-object-id "${mid}" --assignee-principal-type "ServicePrincipal" --role "${role}" --scope "/subscriptions/${subscription}"
```
Everything is according to Juju documentation so far.

Now let's try to use this identity to actually bootstrap a controller and deploy something.

**Option 1 - Managed Identity with managed-identity credentials**

The doc here says "Add a credential type managed-identity 
(the “managed-identity-path” must be of the form <resourcegroup>/<identityname>)"
Let's try to craft the credentials.yaml
```
$ cat credentials.yaml
credentials:
 azure:
  azure-option-one:
   auth-type: managed-identity
   managed-identity-path: jujuclitest/jujuclitest
   subscription-id: xxx
```

Add the credential
```
sudo -u ubuntu juju add-credential -f /home/ubuntu/credentials.yaml --client azure
Credential "azure-option-one" added locally for cloud "azure".
```

Try to bootstrap the controller
```
$ juju bootstrap azure --config resource-group-name=jujuclitest mycontroller
ERROR ManagedIdentityCredential authentication failed. ManagedIdentityCredential authentication failed. the requested identity isn't assigned to this resource
```

Apparently there is a 




