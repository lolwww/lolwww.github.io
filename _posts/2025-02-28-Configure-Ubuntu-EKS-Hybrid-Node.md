---
layout: post
title: How to configure Ubuntu EKS Hybrid node.
---
How to configure Ubuntu EKS Hybrid node.

The purpose of this document is to test the enablement of an on-premise Ubuntu server as AWS Hybrid node with EKS cluster. \
Based on the [AWS guide](https://docs.aws.amazon.com/eks/latest/userguide/hybrid-nodes-overview.html). \
 \
**Hybrid node** is a way to configure your on-premise or locally hosted server as a worker node in an EKS cluster, with the Kubernetes control plane running on AWS.  \
This setup allows you to leverage AWS-provided management, networking, security, and monitoring features while keeping your workloads on-premise. \
You will also be able then to host workloads on both cloud (AWS-managed) and on-premise (Hybrid) worker nodes within the same EKS cluster, supporting high availability for workloads and a hybrid-cloud strategy. \
 \
Let’s have a look at what we require to set up a hybrid node. \
 \
**Prerequisites:** 
- AWS account with permissions to create EKS cluster, VPCs, Subnets etc. 
- on-prem server/VM machine 1 which will become a hybrid worker node. 
At least 1 vCPU and 1GiB RAM. Ubuntu 22.04.5 LTS.  
- on-prem server/VM machine 2 which will become a gateway for site-to-site connectivity. 
1 vCPU and 1GiB RAM would do for testing too. Ubuntu 22.04.5 LTS.  
- router which machines 1 and 2 are connected to. 

Key configuration parts and decisions: 

**1) Networking setup:** 

Essentially there should be a stable communication channel between Hybrid node and Kubernetes control plane living in AWS.  \
AWS documentation suggest 3 ways of achieving that: 
- [Site-to-Site VPN](https://docs.aws.amazon.com/vpn/latest/s2svpn/VPC_VPN.html) for relatively small workloads 
- [AWS Direct Connect](https://docs.aws.amazon.com/directconnect/latest/UserGuide/Welcome.html) for high-performance connectivity, large data volumes etc 
- own VPN solution 

  
We will be using **Site-to-Site VPN** as part of this setup. \
On the AWS side we would require to create a VPC with a minimum of 2 subnets and configure the routing table for them, create internet gateway, VPN and customer gateways and finally VPN connection.  \
On the on-prem side, we need to configure VPN with strongSwan, 2 subnets and a routing table as well. \
As part of this exercise we will replicate exactly the picture below, using the same CIDRs. 

![image](https://github.com/user-attachments/assets/3a262af0-c25e-4a37-82a6-3f274d34984c)

 \
The following has already been configured on the on-prem environment side, so I’m not documenting this part, as it is specific to the on-prem environment: \
 \
Gateway - machine 2: \
Public IP XX.XX.XX.XX \
Internal IP 10.80.0.4 \
Enabled IP forwarding \
Opened ports 500, 4500 UDP \
 \
Subnet for nodes: 10.80.0.0/16  \
Subnet for pods: 10.85.0.0/16  \
 \
Route table: \
Route 10.226.0.0/16 - via 10.80.0.4 
 
**2) Credentials/authentication:** 

Hybrid nodes need a way to securely authenticate with the specific Amazon EKS cluster. \
AWS suggest 2 following ways: 
- [AWS IAM Roles Anywhere](https://docs.aws.amazon.com/eks/latest/userguide/hybrid-nodes-creds.html#_setup_iam_roles_anywhere) - (if you have existing Public Key Infrastructure (PKI) with a Certificate Authority (CA) and certificates for your on-premises environment 
- [AWS SSM hybrid activations](https://docs.aws.amazon.com/eks/latest/userguide/hybrid-nodes-creds.html#_setup_ssm_hybrid_activations) - if you don’t have PKI setup 
 \
We will be using SSM hybrid activations as a simpler way without the need to maintain PKI infrastructure.  
  
**3) EKS cluster and CNI configuration:** 
AWS supports Cilium and Calico for use with hybrid nodes. 
We will use Cilium in this guide, but the configuration between the two is very similar, so it could be easily adopted for Calico as well. 
** Configuration steps.** 
Let’s start with the networking part.  
 \
1.1 Create a VPC with AWS cli: \


 \
1.2 Create 2 subnets as required:  \


 \
 \
1.3 Create an internet gateway and attach it to the VPC, using the VPC and gateway IDs. \
This step is essential to enable internet connectivity for your VPC. \


 \
1.4 Create a VPN gateway and attach it to the VPC.  \


1.5 Create a customer gateway, this is essentially a gateway (machine 2) on premise, and we have to specify its public IP during creation> \
 \


 \
1.6 Create a VPN connection using VPN gateway and customer gateway IDs: \


1.7 Configure VPC routing table.  \
First let’t get the route table ID of the VPC we created: \
 \


 \
1.8 Now let’s create a route to the internet through internet gateway we created earlier: \


 \
1.9 Enable route propagation through our VPN gateway: \
 \


 \
1.10 On the VPN gateway side create routes to on-prem subnets. \
As we enabled the propagation earlier, those routes will automatically appear on our VPC routing table:

 \
1.11 Now we can configure VPN on the on-prem side. \
First download VPN configuration file from AWS. \
Notice the vpn-connection-device-type-id - the ID I’m specifying is of ‘Generic' device type which will work with Strongswan.  \


1.12 On our on-prem gateway (machine2), install and configure the VPN. \
LeftID is a public IP of our on-prem gateway, right ID is a public IP from the AWS VPN settings file. Leftsubnet and rightsubnet are respective subnets. \
In a separate file ipsec.secrets configure Pre-Shared Key secret from the VPN settings file in the format specified: \


 \
Configure sysctl to enable ip forwarding: \


 \
Apply the changes with: \


 \
Now if everything was configured correctly, ipsec should start successfully after: \


 \
 \
You should see the tunnel up and running from the AWS side as well now. \
For HA setup with 2 tunnels, just add a second tunnel section to ipsec.conf and restart. \
 \


![image](https://github.com/user-attachments/assets/33513d78-7b31-4268-853b-b69c55d4e853)



1.13 Create security group for EKS cluster with 443 port open: \


Now let’s configure the credentials / authentication part. \
 \
2.1 Before setting up AWS SSM hybrid activations, we must create a Hybrid Nodes IAM role. \
To save time we will be using a cloudformation template prepared by AWS. \


 \
Create a file with parameters. We will leave them as default. Make sure the arn matches, including your AWS_ACCOUNT_ID and name of the cluster you are planning to use: \


 \
Deploy the cloudformation template to finish role setup: \


 \
2.2 Now setup SSM hybrid activation. Check your AWS_ACCOUNT_ID, name of the cluster and region name before applying. As an output, we will get an ActivationCode and ActivationID, which will later be used to connect the hybrid node to the cluster.

3.1 Now lets prepare configuration file of the EKS cluster, specifying our VPC, subnets and remote network config: \


 \
3.2 Finally create your EKS cluster: \


 \
3.3 Create the access entry for your cluster for your hybrid node to be able to connect to it. \
Notice the type is HYBRID_LINUX. Initially I created an entry with the wrong type and had issues during node setup on a later stage.  \


3.4 Time to set up our hybrid node now.  \
First, make sure nodeadm is installed.  \
It is possible also to pre-create an operating system image with nodeadm built-in as well as nodeadm configuration passed with userdata, so that your hybrid node automatically connects to the EKS cluster. Refer to[ this AWS guide](https://docs.aws.amazon.com/eks/latest/userguide/hybrid-nodes-os.html#_building_operating_system_images). We are doing this manually for now, for testing purposes.  \
 \


 \
3.5 Configure nodeconfig. We have to specify ActivationCode and ActivationID from step 2.2: \


 \
3.6 Now launch the hybrid node init process: \


 \
The node should now appear in EKS cluster compute section: \
It’s marked as ‘not ready’ at the moment as CNI is not configured yet. \
 \
![image](https://github.com/user-attachments/assets/7b57c389-bc09-4016-bc50-bc8d15b24db1)


 \
 \
3.7 To install the CNI we will require helm, kubectl and the kubeconfig for the cluster: \


 \
3.8 Now let's configure and install the CNI.  \
Double-check that your Site-to-Site VPN is up and running at this step. \
CNI will not come up if there is a connectivity issue with EKS cluster. \
 \


 \
If Cilium was configured correctly and your Site-to-Site VPN is up, you should see cilium-operator and cilium-xxx pods up and running. \
 \


 \

![image](https://github.com/user-attachments/assets/99c16795-77ee-4da7-bdb9-38eb18ef0a83)


 \
 \
At this point your hybrid node should be in Ready state: \
 \

![image](https://github.com/user-attachments/assets/93933c38-03d0-4dfc-880f-218c187a53b3)


 \
 \
It should also become Ready from kubectl point of view: \


 \
 \
4.0 Let’s make sure this worker node is indeed functional by deploying something, for example Nginx: \


 \
Expose the deployment and check the status: \


 \

![image](https://github.com/user-attachments/assets/d3d13d5b-c1a4-4f51-ab01-dc6c6a58dd5c)


 \
 \
  \
Now if we curl the the appropriate port from the hybrid node, we should see and Nginx response: \
 \


 \
We’ve been able to setup Site-to-Site VPN with AWS, configure networking and credentials, launch the EKS cluster and successfully connect a Hybrid worker node running Ubuntu 22.04 to it.
