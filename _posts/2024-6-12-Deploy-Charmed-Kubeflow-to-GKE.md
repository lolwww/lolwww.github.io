---
layout: post
title: How to Deploy Charmed Kubeflow to GKE
---

Welcome to the Deploy Charmed Kubeflow to GKE guide. This how-to guide will take you through the steps of deploying Kubeflow to an [Google Kubernetes Engine (GKE)](https://cloud.google.com/kubernetes-engine) cluster. From an architectural point of view, we will spin up an GKE cluster on Google Gloud using gcloud on our local machine. Then with kubectl and juju still on our local machine, we will interact with the cluster to deploy Kubeflow there.

**Requirements**

- Local machine with Ubuntu 22.04 or later
- Google Cloud account

**Prerequisites**

1. Install gcloud tool on your local machine.
Refer to [Google Cloud documentation](https://cloud.google.com/sdk/docs/install) for the latest version of gloud.
```
curl -O https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-cli-462.0.1-linux-x86_64.tar.gz
tar -xf google-cloud-cli-462.0.1-linux-x86_64.tar.gz
./google-cloud-sdk/install.sh
```

2. Install gcloud auth plugin to authenticate with Google Cloud
```
gcloud components install gke-gcloud-auth-plugin
```

3. Install Juju and kubectl (Juju 3.4.3 is used at the moment I am writing this tutorial)
```
sudo snap install kubectl --classic
sudo snap install juju
```

4. Login and enable services on your Google Cloud project
```
gcloud auth login
export PROJECT_ID=test-project
gcloud config set project test-project
gcloud --project=${PROJECT_ID} services enable \
    container.googleapis.com \
    containerregistry.googleapis.com \
    binaryauthorization.googleapis.com
```

**Deploy GKE cluster**

1.  Deploy GKE cluster.
[Kubeflow documentation](https://charmed-kubeflow.io/docs/get-started-with-charmed-kubeflow) suggests at least 4 cores, 32G RAM and 50G of disk for the cluster machines.
So we are going to use n1-standard-16 machine type for our cluster, however n1-standard-8 migh be enough for testing purposes.
```
gcloud container clusters create --binauthz-evaluation-mode=PROJECT_SINGLETON_POLICY_ENFORCE --zone us-central1-a --machine-type n1-standard-16 --disk-size=100G test-cluster
```

2. After the cluster is bootstrapped, save the credentials to be used with kubectl
```
gcloud container clusters get-credentials --zone us-central1-a test-cluster
kubectl config rename-context gke_test-project_us-central1-a_test-cluster gke-cluster
```

3. Bootstrap juju controller to GKE cluster
(we are using /snap/juju/current/bin/juju as a workaround for [bug](https://bugs.launchpad.net/juju/+bug/2007575) temporarily)
```
/snap/juju/current/bin/juju bootstrap gke-cluster
```

**Deploy Charmed Kubeflow**

1. Deploy Charmed Kubeflow bundle with the following command.
```
juju add-model kubeflow
juju deploy kubeflow --trust  --channel=1.8/stable
```
2. Wait until all charms are in green/active state. You can check the state of the charms with the following command. In case you face any issues, refer to the Known issues section below. Keep in mind that oidc-gatekeeper will go to Blocked status until we configure it as shown in next steps.
```
juju status --watch 5s --relations
```
3. Make Kubeflow dashboard accessible by configuring its public URL to be the same as the LoadBalancer’s DNS record.
```
PUBLIC_URL="http://$(kubectl -n kubeflow get svc istio-ingressgateway-workload -o jsonpath='{.status.loadBalancer.ingress[0].ip}')"
echo PUBLIC_URL: $PUBLIC_URL
juju config dex-auth public-url=$PUBLIC_URL 
juju config oidc-gatekeeper public-url=$PUBLIC_URL
```
4. Configure Dex-auth credentials. Feel free to use a different (more secure!) password if you wish.
```
juju config dex-auth static-username=user@example.com 
juju config dex-auth static-password=user
```
5. Navigate to the PUBLIC_URL printed above to access Kubeflow dashboard. You should first see the Dex login screen. Once logged in with the credentials set above, you should now see the Kubeflow “Welcome” page.

**Known issues**

Oidc-gatekeeper “Waiting for pod startup to complete”.

If you see the oidc-gatekeeper/0 unit in juju status output in waiting state with
```
oidc-gatekeeper/0*         waiting      idle   10.1.121.241                 Waiting for pod startup to complete.
```

You can reconfigure the public-url configuration for the charm with following commands
```
PUBLIC_URL="http://$(kubectl -n kubeflow get svc istio-ingressgateway-workload -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')"
juju config oidc-gatekeeper public-url=""
juju config oidc-gatekeeper public-url=$PUBLIC_URL
```

**Clean up resources**
```
juju destroy-model kubeflow --destroy-storage
gcloud container clusters delete test-cluster --zone us-central1-a
```
