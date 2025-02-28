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
- on-prem server/VM machine 1 which will become a hybrid worker node. \
At least 1 vCPU and 1GiB RAM. Ubuntu 22.04.5 LTS.  
- on-prem server/VM machine 2 which will become a gateway for site-to-site connectivity. \
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
- [AWS IAM Roles Anywhere](https://docs.aws.amazon.com/eks/latest/userguide/hybrid-nodes-creds.html#_setup_iam_roles_anywhere) - if you have an existing Public Key Infrastructure (PKI) with a Certificate Authority (CA) and certificates for your on-premises environment 
- [AWS SSM hybrid activations](https://docs.aws.amazon.com/eks/latest/userguide/hybrid-nodes-creds.html#_setup_ssm_hybrid_activations) - if you don’t have a PKI setup 

 
We will be using SSM hybrid activations as a simpler way without the need to maintain PKI infrastructure.  
  
**3) EKS cluster and CNI configuration:** 
AWS supports Cilium and Calico for use with hybrid nodes. 
We will use Cilium in this guide, but the configuration between the two is very similar, so it could be easily adopted for Calico as well. \

**Configuration steps.**  

Let’s start with the networking part. 

1.1 Create a VPC with AWS cli: 
<pre>
<b>$ aws ec2 create-vpc --region us-east-2 --cidr-block 10.226.0.0/16</b>
…
"VpcId": "vpc-030161cb125878088",
…
</pre>


1.2 Create 2 subnets as required:  

<pre>
<b>$ aws ec2 create-subnet --region us-east-2 --vpc-id vpc-030161cb125878088 --cidr-block 10.226.1.0/24 /
--availability-zone us-east-2a</b>
…
 "SubnetId": "subnet-01ceb7d774e21214e",
…

<b>$ aws ec2 create-subnet --region us-east-2 --vpc-id vpc-030161cb125878088 --cidr-block 10.226.2.0/24 /
 --availability-zone us-east-2b</b>
…
"SubnetId": "subnet-01c18a7ffa473fe1e",
…
</pre>

1.3 Create an internet gateway and attach it to the VPC, using the VPC and gateway IDs. \
This step is essential to enable internet connectivity for your VPC. 

<pre>
<b>$ aws ec2 create-internet-gateway --region us-east-2</b>
…
 "InternetGatewayId": "igw-0479a9f3db641f9dd",
…
<b>$ aws ec2 attach-internet-gateway --internet-gateway-id igw-0479a9f3db641f9dd --vpc-id vpc-030161cb125878088 /
 --region us-east-2</b>
</pre>


1.4 Create a VPN gateway and attach it to the VPC.  
<pre><b>$ aws ec2 create-vpn-gateway --region us-east-2 --type ipsec.1</b>
…
"VpnGatewayId": "vgw-0b6e5f5f0f1d7f7ff",
…
<b>$ aws ec2 attach-vpn-gateway --region us-east-2 --vpn-gateway-id vgw-0b6e5f5f0f1d7f7ff --vpc-id vpc-030161cb125878088</b>
</pre>

1.5 Create a customer gateway, this is essentially a gateway (machine 2) on premise, and we have to specify its public IP during creation> 
 
<pre>
<b>$ aws ec2 create-customer-gateway --region us-east-2 --type ipsec.1 --public-ip XX.XX.XX.XX</b>
…
"CustomerGatewayId": "cgw-02ad11ff10c2b161a",
…
</pre>

1.6 Create a VPN connection using VPN gateway and customer gateway IDs: 
<pre>
<b>$ aws ec2 create-vpn-connection --region us-east-2  --vpn-gateway-id vgw-0b6e5f5f0f1d7f7ff --type ipsec.1 /
 --customer-gateway-id cgw-02ad11ff10c2b161a --options '{"StaticRoutesOnly": true}'</b> 
…
"VpnConnectionId": "vpn-0d87031ec374c6c79"
…
</pre>

1.7 Configure VPC routing table.  \
First let’t get the route table ID of the VPC we created: 

<pre>
<b>$ aws ec2 describe-route-tables --filters "Name=vpc-id,Values=vpc-030161cb125878088"</b>
…
"RouteTableId": "rtb-0d41e058e5d2a64ed",
…
</pre>

1.8 Now let’s create a route to the internet through internet gateway we created earlier: 

<pre>
<b>$ aws ec2 create-route --route-table-id rtb-0d41e058e5d2a64ed --destination-cidr-block 0.0.0.0/0 /
 --gateway-id igw-0479a9f3db641f9dd</b>
</pre>

1.9 Enable route propagation through our VPN gateway: 
 
<pre>
<b>$ aws ec2 enable-vgw-route-propagation --gateway-id vgw-0b6e5f5f0f1d7f7ff --route-table-id rtb-0d41e058e5d2a64ed </b>
</pre>

1.10 On the VPN gateway side create routes to on-prem subnets. \
As we enabled the propagation earlier, those routes will automatically appear on our VPC routing table:

<pre>
<b>$ aws ec2 create-vpn-connection-route --vpn-connection-id vpn-0d87031ec374c6c79 --destination-cidr-block 10.80.0.0/16

$ aws ec2 create-vpn-connection-route   --vpn-connection-id vpn-0d87031ec374c6c79 --destination-cidr-block 10.85.0.0/16</b>
</pre>

1.11 Now we can configure VPN on the on-prem side. \
First download VPN configuration file from AWS. \
Notice the vpn-connection-device-type-id - the ID I’m specifying is of ‘Generic' device type which will work with Strongswan.  

<pre>
<b>$ aws ec2 get-vpn-connection-device-sample-configuration --vpn-connection-id vpn-0f0cde758878abe4c /
 --vpn-connection-device-type-id 9005b6c1</b>
</pre>

1.12 On our on-prem gateway (machine2), install and configure the VPN. \
LeftID is a public IP of our on-prem gateway, right ID is a public IP from the AWS VPN settings file. Leftsubnet and rightsubnet are respective subnets. \
In a separate file ipsec.secrets configure Pre-Shared Key secret from the VPN settings file in the format specified: \

<pre>
<b>$ sudo apt-get update
$ sudo apt install strongswan 

$ sudo cat /etc/ipsec.conf</b>
 conn tun
	authby=secret
	auto=start
	left=%defaultroute
	<b>leftid=XX.XX.XX.XX</b>
	<b>right=3.147.130.26</b>
	type=tunnel
	ikelifetime=8h
	keylife=1h
	esp=aes128-sha1-modp1024
	ike=aes128-sha1-modp1024
	keyingtries=%forever
	keyexchange=ike
	<b>leftsubnet=10.80.0.0/16</b>
	<b>rightsubnet=10.226.0.0/16</b>
	dpddelay=10
	dpdtimeout=30

<b>$ sudo cat /etc/ipsec.secrets</b>
XX.XX.XX.XX 3.147.130.26 : PSK "xxxxx"
</pre>
 
Configure sysctl to enable ip forwarding: 

<pre>
<b>$ sudo vim /etc/sysctl.conf</b>
...
net.ipv4.ip_forward = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
...
</pre>

Apply the changes with: 

<pre>
<b>$ sudo sysctl -p</b>
</pre>

Now if everything was configured correctly, ipsec should start successfully after: 

<pre>
<b>$ sudo ipsec restart

$ sudo ipsec statusall</b>
…
Listening IP addresses:
  10.80.0.4
Connections:
     	tun:  10.80.0.4...3.147.130.26  IKEv1/2
     	tun:   local:  [XX.XX.XX.XX] uses pre-shared key authentication
     	tun:   remote: [3.147.130.26] uses pre-shared key authentication
     	tun:   child:  10.80.0.0/16 === 10.226.0.0/16 TUNNEL
Security Associations (1 up, 0 connecting):
     	tun[1]: ESTABLISHED 4 minutes ago
…
</pre>

You should see the tunnel up and running from the AWS side as well now. 
For HA setup with 2 tunnels, just add a second tunnel section to ipsec.conf and restart. 

![image](https://github.com/user-attachments/assets/33513d78-7b31-4268-853b-b69c55d4e853)

1.13 Create security group for EKS cluster with 443 port open: 

<pre>
<b>$ aws ec2 create-security-group --group-name hybdircluster --description "security group for hybrid nodes" /
 --vpc-id vpc-030161cb125878088</b>
…
"GroupId": "sg-035e934b7c3f64f3e"
…

<b>$ aws ec2 authorize-security-group-ingress --group-id sg-035e934b7c3f64f3e --ip-permissions /
 '[{"IpProtocol": "tcp", "FromPort": 443, "ToPort": 443, "IpRanges": [{"CidrIp": "10.80.0.0/16"}, {"CidrIp": "10.85.0.0/16"}]}]'</b>
</pre>

Now let’s configure the credentials / authentication part. 

2.1 Before setting up AWS SSM hybrid activations, we must create a Hybrid Nodes IAM role. \
To save time we will be using a cloudformation template prepared by AWS. 

<pre>
<b>$ curl -OL 'https://raw.githubusercontent.com/aws/eks-hybrid/refs/heads/main/example/hybrid-ssm-cfn.yaml'</b>
</pre>

Create a file with parameters. We will leave them as default. Make sure the arn matches, including your AWS_ACCOUNT_ID and name of the cluster you are planning to use: 

<pre>
<b>$ cat cfn-ssm-parameters.json</b>
{
  "Parameters": {
	"RoleName": "AmazonEKSHybridNodesRole",
	"SSMDeregisterConditionTagKey": "EKSClusterARN",
	"SSMDeregisterConditionTagValue": "arn:aws:eks:us-east-2:AWS_ACCOUNT_ID:cluster/hybridnodescluster"
  }
}
</pre>

Deploy the cloudformation template to finish role setup: 

<pre>
<b>$ aws cloudformation deploy --stack-name hybridnodes-stack --template-file hybrid-ssm-cfn.yaml /
 --parameter-overrides file://cfn-ssm-parameters.json --capabilities CAPABILITY_NAMED_IAM </b>
…
Successfully created/updated stack - hybridnodes-stack
…
</pre>

2.2 Now setup SSM hybrid activation. Check your AWS_ACCOUNT_ID, name of the cluster and region name before applying. As an output, we will get an ActivationCode and ActivationID, which will later be used to connect the hybrid node to the cluster.

<pre>
<b>$ aws ssm create-activation --region us-east-2 --default-instance-name eks-hybrid-nodes --description /
 "Activation for EKS hybrid nodes" --iam-role AmazonEKSHybridNodesRole /
 --tags Key=EKSClusterARN,Value=arn:aws:eks:us-east-2:AWS_ACCOUNT_ID:cluster/hybridnodescluster  /
 --registration-limit 10 </b>
{
"ActivationId": "xxxx",
"ActivationCode": "zzzz"
}
</pre>

3.1 Now lets prepare configuration file of the EKS cluster, specifying our VPC, subnets and remote network config: 

<pre>
<b>$ cat cluster-config.yaml</b>
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: hybridnodescluster
  region: us-east-2
  version: "1.32"

vpc:
  id: vpc-030161cb125878088
  subnets:
	private:
  	us-east-2a:
    	id: subnet-01ceb7d774e21214e
  	us-east-2b:
    	id: subnet-01c18a7ffa473fe1e

remoteNetworkConfig:
  iam:
	provider: ssm
  remoteNodeNetworks:
  - cidrs: ["10.80.0.0/16"]
  remotePodNetworks:
  - cidrs: ["10.85.0.0/16"]
</pre>

3.2 Finally create your EKS cluster: 

<pre>
<b>$ eksctl create cluster -f cluster-config.yaml</b>
…
…
EKS cluster "hybridnodescluster" in "us-east-2" region is ready
…
</pre>

3.3 Create the access entry for your cluster for your hybrid node to be able to connect to it. \
Notice the type is HYBRID_LINUX. \
Initially I created an entry with the wrong type and had issues during node setup on a later stage.  \

<pre>
<b>$ aws eks create-access-entry --cluster-name hybridnodescluster /
 --principal-arn arn:aws:iam::AWS_ACCOUNT_ID:role/AmazonEKSHybridNodesRole --type HYBRID_LINUX</b>
…
{
	"accessEntry": {
    	"clusterName": "hybridnodescluster",
    	"principalArn": "arn:aws:iam::AWS_ACCOUNT_ID:role/AmazonEKSHybridNodesRole",
…
</pre>

3.4 Time to set up our hybrid node now.  \
First, make sure nodeadm is installed.  \
It is possible also to pre-create an operating system image with nodeadm built-in as well as nodeadm configuration passed with userdata, so that your hybrid node automatically connects to the EKS cluster. Refer to[ this AWS guide](https://docs.aws.amazon.com/eks/latest/userguide/hybrid-nodes-os.html#_building_operating_system_images). We are doing this manually for now, for testing purposes.  

<pre>
<b>$ curl -OL 'https://hybrid-assets.eks.amazonaws.com/releases/latest/bin/linux/amd64/nodeadm'

$ chmod +x nodeadm

$ sudo ./nodeadm install 1.32 --credential-provider ssm </b>
…
Finishing up install
…
</pre>

3.5 Configure nodeconfig. We have to specify ActivationCode and ActivationID from step 2.2: 

<pre>
<b>$ cat nodeconfig.yaml</b>
apiVersion: node.eks.aws/v1alpha1
kind: NodeConfig
spec:
  cluster:
	name: hybridnodescluster
	region: us-east-2
  hybrid:
	ssm:
  	activationCode: zzzz
  	activationId: xxxx
</pre>

3.6 Now launch the hybrid node init process: 

<pre>
<b>$ sudo ./nodeadm init -c file://nodeconfig.yaml</b>
…
…
{"level":"info","ts":1740140387.4794908,"caller":"flows/init.go:79","msg":"Ensuring daemon is running..","name":"kubelet"}
{"level":"info","ts":1740140387.745159,"caller":"flows/init.go:83","msg":"Daemon is running","name":"kubelet"}
{"level":"info","ts":1740140387.7452102,"caller":"flows/init.go:89","msg":"Finished post-launch tasks","name":"kubelet"}
…
</pre>


The node should now appear in EKS cluster compute section: \
It’s marked as ‘not ready’ at the moment as CNI is not configured yet. \
 \
![image](https://github.com/user-attachments/assets/7b57c389-bc09-4016-bc50-bc8d15b24db1)


 \
 \
3.7 To install the CNI we will require helm, kubectl and the kubeconfig for the cluster: 

<pre>
<b>$ sudo snap install helm --classic
$ sudo snap install kubectl --classic
$ aws eks update-kubeconfig --region us-east-2 --name hybridnodescluster</b>
</pre>

3.8 Now let's configure and install the CNI.  \
Double-check that your Site-to-Site VPN is up and running at this step. \
CNI will not come up if there is a connectivity issue with EKS cluster. \

<pre>
<b>$ cat cilium.yaml</b>
affinity:
  nodeAffinity:
	requiredDuringSchedulingIgnoredDuringExecution:
  	nodeSelectorTerms:
  	- matchExpressions:
    	- key: eks.amazonaws.com/compute-type
      	operator: In
      	values:
      	- hybrid
ipam:
  mode: cluster-pool
  operator:
	clusterPoolIPv4MaskSize: 25
	clusterPoolIPv4PodCIDRList:
	- 10.85.0.0/16
operator:
  affinity:
	nodeAffinity:
  	requiredDuringSchedulingIgnoredDuringExecution:
    	nodeSelectorTerms:
    	- matchExpressions:
      	- key: eks.amazonaws.com/compute-type
        	operator: In
        	values:
          	- hybrid
  unmanagedPodWatcher:
	restart: false
envoy:
  enabled: false

<b>$ helm repo add cilium https://helm.cilium.io/</b>

<b>$ helm install cilium cilium/cilium --version v1.17 --namespace kube-system --values cilium.yaml</b>
</pre>


If Cilium was configured correctly and your Site-to-Site VPN is up, you should see cilium-operator and cilium-xxx pods up and running. 

<pre>
<b>$ kubectl get all -n kube-system</b>
</pre>

![image](https://github.com/user-attachments/assets/99c16795-77ee-4da7-bdb9-38eb18ef0a83)

At this point your hybrid node should be in Ready state: 

![image](https://github.com/user-attachments/assets/93933c38-03d0-4dfc-880f-218c187a53b3)

It should also become Ready from kubectl point of view: 

<pre>
<b>$ kubectl get nodes</b>
NAME               	STATUS   ROLES	AGE 	VERSION
mi-0e0cf153e532e2b6a   Ready	<none>   1d2h   v1.32.1-eks-5d632ec
</pre>

4.0 Let’s make sure this worker node is indeed functional by deploying something, for example Nginx: 

<pre>
<b>$ kubectl create -f https://k8s.io/examples/application/deployment.yaml</b>
deployment.apps/nginx-deployment created
</pre>
 
Expose the deployment and check the status: 

<pre>
<b>$ kubectl expose deployment.apps/nginx-deployment --port=80 --target-port=80 --type NodePort /
service/nginx-deployment exposed</b>

<b>$ kubectl get all </b>
</pre>

![image](https://github.com/user-attachments/assets/d3d13d5b-c1a4-4f51-ab01-dc6c6a58dd5c)

Now if we curl the the appropriate port from the hybrid node, we should see and Nginx response: 

<pre>
<b>$ curl localhost:30445</b>
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
	body {
    	width: 35em;
    	margin: 0 auto;
    	font-family: Tahoma, Verdana, Arial, sans-serif;
	}
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>
<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>
<p><em>Thank you for using nginx.</em></p>
</body>
</html>
</pre>

We’ve been able to setup Site-to-Site VPN with AWS, configure networking and credentials, launch the EKS cluster and successfully connect a Hybrid worker node running Ubuntu 22.04 to it.
