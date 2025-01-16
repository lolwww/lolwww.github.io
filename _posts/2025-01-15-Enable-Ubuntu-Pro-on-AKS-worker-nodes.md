---
layout: post
title: How to enable Ubuntu Pro on AKS worker nodes 
---

There are essentially two ways to consume Ubuntu Pro subscription from Canonical:
1. **Metered**
On a public cloud, you can either create a VM from Ubuntu Pro image or upgrade the VM to Ubuntu Pro.
Here is the [guide for Microsoft Azure](https://learn.microsoft.com/en-us/azure/virtual-machines/workloads/canonical/ubuntu-pro-in-place-upgrade).
You will be charged through a public cloud provider depending on how much this host was up and running.
2. **Using a token**
You can purchase a token from Canonical which will be valid for a number of hosts for a year and subscribe any machine to Pro with
```console
$ sudo pro attach %token%
```

Unfortunately, there is no way at the moment (as of Jan 2025) to use Metered method on Azure AKS hosts.
So if you would like to have Pro subscription and it's compliance and security features on AKS hosts, only token way is remaining.
Which presents us a challenge - of course we can run $pro attach on the existing AKS workers (hosts in respective VMSS), 
but what if there is an autoscaling event? New hosts would not be subscribed to Pro automatically (nor detached from Pro on removal).

Well, it's possible to achieve that with kubernetes DaemonSet.

[pro-aks-installer](https://github.com/lolwww/pro-aks-installer) repository has the necessary files to get started.

First, let's have a look at the [proconfigmap.yaml](https://github.com/lolwww/pro-aks-installer/blob/master/k8s/proconfigmap.yaml) in k8s folder.
It defines **install.sh** and **cleanup.sh** scripts - first one installs ubuntu-advantage-tools and attaches the host to Pro using a token.
Second one does the detach. It will be executed on removal of the host (for example scale-down of VMSS).

Second file, [daemonset.yaml](https://github.com/lolwww/pro-aks-installer/blob/master/k8s/daemonset.yaml) defines the daemonset itself.
It's pretty simple - the container image used is lolwww/test:1.3 - you can use this image or build a new one using the [Dockerfile](https://github.com/lolwww/pro-aks-installer/blob/master/Dockerfile) provided in this repository. 

It contains necessary scripts:
- [runOnHost.sh](https://github.com/lolwww/pro-aks-installer/blob/master/runOnHost.sh) copies install.sh and cleanup.sh into host path, 
changes the permissions and executes install.sh on the worker using nsenter (to execute it in the right namespaces, instead of container namespace)
- [runCleanup.sh](https://github.com/lolwww/pro-aks-installer/blob/master/runCleanup.sh) just executes cleanup.sh.
Also the daemonset mounts necessary paths to be able to copy scripts to the host and defines preStop: command so that runCleanup.sh is executed before container termination.

So the flow is the following:
1) daemonset launches a container with image from Dockerfile on every node in the cluster.
(you can define labels here if you only wish some of the hosts are subscribed to Pro)
2) runOnHost.sh is then launched which copies the install and cleanup scripts to the mounted host path, launches install.sh (subscribing host to Pro)
3) in case of autoscaling event up - daemonset will create a container on a newly added node (subscribing it to Pro)
4) in case of scaling down - preStop hook will be executed, detaching the node from Pro.

The daemonset could be adjusted according to your cluster config and don't forget to replace the %TOKEN% value in proconfigmap.yaml.
Apply the daemonset like this below and enjoy Ubuntu Pro features on your AKS worker nodes.
```console
; kubectl apply -f ./k8s
```

