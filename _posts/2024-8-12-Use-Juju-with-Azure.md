---
layout: post
title: How to use Juju with Microsoft Azure
---

Based on the Juju [documentation](https://juju.is/docs/juju/microsoft-azure) and personal experience.
Juju version used is 3.6-beta2. \
Generally there are 2 ways to make Juju work with Azure (for it to be able to create, delete, modify resources on Azure). \
Each of them has some variations:

**1. Old, non-recommended service-principal-credentials way**

Juju would copy the user’s credential secrets to the controller and use that when making API calls to the cloud. 
Credentials are stored.
However if you require juju version earlier than 3.6-beta - use this method.


**2. New, recommended Managed Identity (MI) way**

The controller gets the authorisation it needs to manage cloud resources from the Managed Identity. No credentials stored.

**Option a) pre-created MI with managed-identity credentials**

**Option b) pre-created MI with service-principal-secret credentials**

**Option c) service-principal-secret credentials, let juju create MI**

Limitations:
- Only supported starting juju 3.6 (currently beta)
- Juju cli should be on Azure VM for it to be able to reach cloud metadata endpoint.
- Managed Identity and the Juju resources should be on the same Azure subscription


I will try to document all options we have with examples.


**Method 1 : Service-principal credentials way**

Perform cli app registration in [Microsoft Entra ID](https://portal.azure.com/#view/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/~/RegisteredApps), to allow juju cli create/modify and delete Azure resources.

Let’s call this app ‘jujucli’ and create a secret for it:
```
az ad app create --display-name jujucli
appid=$(az ad app list --display-name jujucli --query "[].{id: appId}" -o tsv)
az ad app credential list --id $appid --query keyId -o tsv
az ad app credential reset --id $appid
{
  "appId": "xxxxxx",
  "password": "qqqqq",
  "tenant": "zzzzz"
}
```
Create service principal:
```
az ad sp create --id $appid
spObjectId=$(az ad sp show --id $appid --query id -o tsv)
```

Now let’s grant this application Owner role on a resource we want to use, for example resource group(1) of full subscription(2):

```
(1) az role assignment create --assignee $spObjectId --role Owner --scope /subscriptions/xxxxxx/resourcegroups/jujuclitest
(2) az role assignment create --assignee $spObjectId --role Owner --scope /subscriptions/xxxxxx
```
Now inside our resource group (or subscription) Access Control IAM tab we should see this application as owner:

![IMAGE 2024-08-16 12:28:04](https://github.com/user-attachments/assets/e3e703a2-f126-4633-aea8-cbfd3e99b29d)


Now we can use those credentials with:
```
cat credentials3.yaml
credentials:
  azure:
    azure-option-three:
      auth-type: service-principal-secret
      application-id: xxxxxx
      application-password: qqqq
      subscription-id: aaaaaa
```

Now setup the juju region for your controller and future operations so we don’t have to do it with every command (assuming we are working in the same region most of the time anyway): 

```
juju default-region azure eastus
```

Bootstrap the controller with:
```
$ juju bootstrap --constraints allocate-public-ip=false --config resource-group-name=jujuclitest --config network=VNET azure

$ juju add-model opensearch --config resource-group-name=jujuclitest --config network=VNET

$ juju add-machine --constraints="instance-type=Standard_D4s_v5 allocate-public-ip=false"
```

**Method 2: Managed Identity (MI) way**

Prerequisite:
This method requires us to have juju CLI inside Azure for it to be able to reach cloud metadata endpoint, so let’s create a small Azure VM for this and install juju there:
```
az vm create --resource-group MyResourceGroup --name jujucli  --image Ubuntu2204 --location eastus --size Standard_B1s --admin-username ubuntu --ssh-key-value "$(az sshkey show --resource-group TT --name azurekey --query publicKey -o tsv)" --public-ip-sku Standard --zone 1 
ssh ubuntu@VM_IP
sudo snap install juju --channel 3.6/beta
```

For Options a) and b) We will be required to pre-create Managed Identity first, so let’s start with this part.

**Creating Managed Identity**

Let's create 2 identities - subscription scoped and resource group scoped to test both:
```
export group=jujuclitest
export location=eastus
export role=jujuclitestrole
export role2=jujuclitestrole2
export identityname=jujuclitestidentity
export identityname2=jujuclitestidentity2
export subscription=xxxxx

az group create --name "${group}" --location "${location}"

# One subscription scope identity, role and role assignment:
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

# Second resource group scope identity, role and role assignment:
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

If we look at our Access control - Role Assignment tab inside our resource group ‘jujucli’ this is what we should see now:

![image](https://github.com/user-attachments/assets/f94d6740-763c-4d3d-aaab-d0baa60f29ad)


One last important thing now - the MIs are created now, but we should grant our VM with juju the permission to use the MIs.
Without this you will be getting and error (ManagedIdentityCredential authentication failed) trying to do the controller bootstrap and other operations.

For MI1:
```
az vm identity assign --resource-group MyResourceGroup --name jujucli --identities /subscriptions/${subscription}/resourceGroups/${group}/providers/Microsoft.ManagedIdentity/userAssignedIdentities/${identityname}
```

For MI2:
```
az vm identity assign --resource-group MyResourceGroup --name jujucli --identities /subscriptions/${subscription}/resourceGroups/${group}/providers/Microsoft.ManagedIdentity/userAssignedIdentities/${identityname2}
```

Now if we look at our VM - Security - Identity - User assigned tab we should see both MI’s there:

![image](https://github.com/user-attachments/assets/aef7b6e7-de0e-4c01-8056-0ff7dd44535e)

Now let’s try to use our MIs to actually bootstrap a controller and deploy something.

**Option a) - pre-created MI with managed-identity credentials**

```
$ cat credentials.yaml
credentials:
  azure:
    azure-option-one:
      auth-type: managed-identity
      managed-identity-path: jujuclitest/jujuclitestidentity
      subscription-id: xxxx
```

Add the credential:
```
juju add-credential -f credentials.yaml --client azure
Credential "azure-option-one" added locally for cloud "azure".
```

Now setup the juju region for your controller and future operations so we don’t have to do it with every command (assuming we are working in the same region most of the time anyway): 

```
juju default-region azure eastus
```

Try to bootstrap the controller:
```
$ juju bootstrap azure mycontroller
...
Bootstrap complete, controller "mycontroller" is now available
Controller machines are in the "controller" model
```

As your credentials are subscription scope, you can operate in any resource group, the controller VM will be created of type Standard_DS1_v2 in random resource group like ‘juju-controller-xxxx’ and placed in 
‘juju-internal-network/juju-controller-subnet’.

You can also specify the resource group and network for controller like this:
```
juju bootstrap azure --config resource-group-name=jujuclitest --config network=jujuclitest mycontroller 
```

Now if you :
```
juju add-model test
juju add-machine
```

As your credentials are subscription scope, you can operate in any resource group, the VM of type Standard A1 v2 will be created in random resource group like ‘juju-test-xxxx’ and placed in ‘‘juju-internal-network/juju-internal-subnet’

If you would like to specify those, you will have to do:

```
juju add-model test --config resource-group-name=jujuclitest --config network=jujuclitest 
```

For adding machines, I usually use the following way:
```
juju add-machine --constraints="instance-type=Standard_B1s allocate-public-ip=false"
```

Now let’s try resource group scope credential:
```
$ cat credentials2.yaml
credentials:
  azure:
    azure-option-two:
      auth-type: managed-identity
      managed-identity-path jujuclitest/jujuclitestidentity2
      subscription-id: xxxx
```

Add the credential:
```
juju add-credential -f credentials2.yaml --client azure
Credential "azure-option-two" added locally for cloud "azure".
```

Try to bootstrap the controller:
```
$ juju bootstrap azure --credential azure-option-two mycontroller
```

you will see the following error message:
*{
  "error": {
	"code": "AuthorizationFailed",
	"message": "The client with object id 'xxx' does not have authorization to perform action 'Microsoft.Authorization/roleAssignments/read' over scope '/subscriptions/xxx/resourceGroups/juju-controller-4f1c32d9/providers/Microsoft.Authorization' or the scope is invalid. If access was recently granted, please refresh your credentials."
  }
}*

this is because our MI is only for the particular resource group, so we can only operate there, let's specify the group explicitly:

```
$ juju bootstrap azure --config resource-group-name=jujuclitest --credential azure-option-two mycontroller
```

The rest of the model/VM operations are similar, except that resource group and network in that resource group could be used only:
```
juju add-model test --config resource-group-name=jujuclitest --config network=jujuclitest 

juju add-machine --constraints="instance-type=Standard_B1s allocate-public-ip=false"
```

**Option b) pre-created MI with service-principal-secret credentials**

This method is similar - we will be using same MI’s but different type of credentials.
But first let’s create service-principal secret.

Follow the **Method 1 : Service-principal credentials way** to create credentials, but don't bootstrap the controller using Method 1.
Once you have the credentials using Method1, bootstrap the controller and Juju will create another MI for us:
```
juju bootstrap azure --constraints "instance-role=X”
```

Instance role could be:
a) subscription/resourcegroup/identityname (MI in different subscription and resourcegroup)
b) resourcegroup/identityname (MI in different resourcegroup)
c) identityname (MI in same resource group)

Let’s use option C here as our MI is in the same resource group:
```
juju bootstrap azure --config resource-group-name=jujuclitest --constraints "instance-role=jujuclitestidentity" --credential azure-option-three mycontroller
…
Bootstrap complete, controller "mycontroller" is now available
Controller machines are in the "controller" model
```
The issue with this method however is that when you run
Juju kill-controller mycontroller, the controller VM stays alive on Azure.

**Option c) service-principal-secret credentials, let juju create MI**

To use this method juju will try to create a deployment with MI, and if we use only resource group scoped credentials as in previous method bootstrap will fail.

Follow the **Method 1 : Service-principal credentials way** to create credentials, but don't bootstrap the controller using Method 1.

We will need to wider the scope for our credentials first, let’s give it full subscription scope:
```
az role assignment create --assignee $spObjectId --role Owner --scope /subscriptions/xxxxxx
```

Now we can bootstrap the controller:
```
juju bootstrap azure --config resource-group-name=jujuclitest --constraints "instance-role=auto" --credential azure-option-three mycontroller
Bootstrap complete, controller "mycontroller" is now available
Controller machines are in the "controller" model
```

We will see an MI created in our resource group:
![image](https://github.com/user-attachments/assets/d6eac9c9-9a7b-4282-b5dc-2f36d6e33df4)

The issue with this method however is that when you run
Juju kill-controller mycontroller, the controller VM stays alive on Azure.

Hope you enjoyed this tutorial and successfully deployed your project with Juju on Azure.

Please feel free to create issues / suggest improvements.
