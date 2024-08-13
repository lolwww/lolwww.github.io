---
layout: post
title: How to use Juju with Microsoft Azure
---

Based on the [Juju documentation](https://juju.is/docs/juju/microsoft-azure) and personal experience.

Juju version used is 3.6-beta.

As Juju documentation describes, there are 3 ways to make Juju work with Azure
(for it to be able create, delete, modify resources on Azure):

1. **pre-created [Managed Identity](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview) with managed-identity credentials** (recommended by Juju team, although only available from Juju 3.6-beta).
Limitations: 
- juju cli should be on Azure cloud to be able to reach cloud metadata endpoint.
- managed identity and the Juju resources are on the same subscription
2. **pre-created Managed Identity + service-principal-secret credentials**
   
3. **service-principal-secret credentials on bootstrap, then Managed Identity created on and used by Juju** 

And one other way that I know of:

 **Undocumented Entra ID app registrations way**.

I will try to attempt to document all ways and see which one works better.

For Options 1 and 2 both we will be required to create Managed Identity first, so let's start with this part.

**Creating Managed Identity**

We will try to follow the suggested documentation there and use Azure cli(Bash).
Let's start from scratch as we don't have any resource groups, roles etc.
We'll create 2 identities - subscription scoped and resource group scoped to test both.

```
export group=jujuclitest
export location=eastus
export role=jujuclitestrole
export role2=jujuclitestrole2
export identityname=jujuclitestidentity
export identityname2=jujuclitestidentity2
export subscription=xxxxx

az group create --name "${group}" --location "${location}"

One subscription scope identity, role and role assignment:
az identity create --resource-group "${group}" --name "${identityname}"
mid=$(az identity show --resource-group "${group}" --name "${identityname}" --query principalId --output tsv)
az role definition create --role-definition "{
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
az role assignment create --assignee-object-id "${mid}" --assignee-principal-type "ServicePrincipal" --role "${role}" --scope "/subscriptions/${subscription}"

and second resource group scope identity, role and role assignment:
az identity create --resource-group "${group}" --name "${identityname2}"
mid=$(az identity show --resource-group "${group}" --name "${identityname2}" --query principalId --output tsv)
az role definition create --role-definition "{
  	\"Name\": \"${role2}\",
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
        	\"/subscriptions/${subscription}/resourcegroups/${group}\"
  	]
  }"
az role assignment create --assignee-object-id "${mid}" --assignee-principal-type "ServicePrincipal" --role "${role2}" --scope "/subscriptions/${subscription}/resourcegroups/${group}"

```
Everything is according to Juju documentation so far.
Now let's try to use our identities to actually bootstrap a controller and deploy something.

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
   managed-identity-path: jujuclitest/jujuclitestidentity
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
Doesn't seem to work.
Let's try the same with resource scoped managed identity.
```
$ cat credentials2.yaml
credentials:
 azure:
  azure-option-two:
   auth-type: managed-identity
   managed-identity-path: jujuclitest/jujuclitestidentity2
   subscription-id: xxx
```

Add the credential
```
sudo -u ubuntu juju add-credential -f /home/ubuntu/credentials2.yaml --client azure
Credential "azure-option-two" added locally for cloud "azure".
```

Try to bootstrap the controller
```
$ juju bootstrap azure --config resource-group-name=jujuclitest --credential azure-option-two mycontroller
ERROR ManagedIdentityCredential authentication failed. ManagedIdentityCredential authentication failed. the requested identity isn't assigned to this resource
```
Doesn't seem to work either.

Let's see if we can make it work with role 'Owner' instead of documented way.

```
mid=$(az identity show --resource-group "${group}" --name "${identityname}" --query principalId --output tsv)
az role assignment create --assignee-object-id "${mid}" --scope "/subscriptions/${subscription}" --role "Owner"
mid=$(az identity show --resource-group "${group}" --name "${identityname2}" --query principalId --output tsv)
az role assignment create --assignee-object-id "${mid}" --scope "/subscriptions/${subscription}/resourcegroups/${group}" --role "Owner"

$ juju bootstrap azure --config resource-group-name=jujuclitest --credential azure-option-one mycontroller
ERROR ManagedIdentityCredential authentication failed. ManagedIdentityCredential authentication failed. the requested identity isn't assigned to this resource
$ juju bootstrap azure --config resource-group-name=jujuclitest --credential azure-option-two mycontroller
ERROR ManagedIdentityCredential authentication failed. ManagedIdentityCredential authentication failed. the requested identity isn't assigned to this resource
```

How about we add ServicePrincipal type?
```
az role assignment create --assignee-object-id "${mid}" --assignee-principal-type "ServicePrincipal" --scope "/subscriptions/${subscription}" --role "Owner"
az role assignment create --assignee-object-id "${mid}" --assignee-principal-type "ServicePrincipal" --scope "/subscriptions/${subscription}/resourcegroups/${group}" --role "Owner"

$ juju bootstrap azure --config resource-group-name=jujuclitest --credential azure-option-one mycontroller
ERROR ManagedIdentityCredential authentication failed. ManagedIdentityCredential authentication failed. the requested identity isn't assigned to this resource
$ juju bootstrap azure --config resource-group-name=jujuclitest --credential azure-option-two mycontroller
ERROR ManagedIdentityCredential authentication failed. ManagedIdentityCredential authentication failed. the requested identity isn't assigned to this resource
```

No luck either.
Let's look at bootstrap with --verbose and --debug.
One particular line that I don't like seeing :
```
11:11:39 DEBUG juju.cmd.juju.commands bootstrap.go:1418 authenticating with region "" and credential "azure-option-one" ()
```
Seems like we need to give it the region before trying the bootstrap.
```
sudo -u ubuntu juju default-region azure eastus
```
Try again:
```
$ juju bootstrap azure --config resource-group-name=jujuclitest --credential azure-option-one mycontroller
ERROR ManagedIdentityCredential authentication failed. ManagedIdentityCredential authentication failed. the requested identity isn't assigned to this resource
```

no luck either
