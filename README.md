# Scaling WordPress application using Rook on Azure Kubernetes Service 


WordPress is a popular free and open-source content management system based on PHP & MySQL.
One of the great benefits of using cloud computing is having the ability to support your application with scaling based on client traffic. There are currently many ways to support WordPress scaling such as Reverse Proxy Caching, Persistent Object caching, DB distribution, using CDN and beyond. However, there are challenges with elastic (horizontal) scaling related to stateful configurations. A WordPress application is composed of plugins, themes, content images and many php files. All of these must be centrally shared for the same application and are not hosted on the MySQL DB tier.

## The challenge 
Azure Kubernetes Service supports several Storage options. All of them provide a central persistent storage layer with different performance capabilities.
Using [Azure Managed Disks](https://https://docs.microsoft.com/en-us/azure/aks/azure-disks-dynamic-pv) As  provided adequate performance but by nature, a disk is bound to a single Pod, hence not supporting Pod scaling. Also, a VM (a Kubernetes Node for the matter) can support only up to a certain number of attached disks.
Azure [Files](https://docs.microsoft.com/en-us/azure/aks/azure-files-dynamic-pv) and [Blob Fuse](https://github.com/Azure/azure-storage-fuse) can serve multiple Pods simultaneously however we opted for even better performance levels while leveraging the Kubernetes extensive Stack and Plugins (In-Kubernetes Storage Solutions).

## Rook ![alt-text](https://github.com/rook/rook/blob/master/Documentation/media/logo.png "logo")
[Rook](https://rook.io/) is an Open Source incubating level project of the CNCF running as a native Kubernetes service.
Automating Ceph (open source Software Defined Storage), Rook offers storage for your Kubernetes applications through persistent volumes, providing optimal integration with applications in need of storage. Rook handles the heavy lifting behind the scenes such as provisioning and management.
We’ll use Rook as our persistent storage layer for the WordPress frontend.

## Prerequisites 
[Create an Azure account](https://azure.microsoft.com/en-us/free/) with an Azure [AKS](https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough-portal)

## The Process 
Installing Rook is quite easy and consists of 4 steps. The official installation guide can be found [here](https://rook.io/docs/rook/v0.9/ceph-quickstart.html).
Here’s a tailored guide for installing Rook on AKS:
1.	Install FlexVolume for AKS:
> kubectl apply -f https://raw.githubusercontent.com/Azure/kubernetes-keyvault-flexvol/master/dep> loyment/kv-flexvol-installer.yaml

2.	Install the Rook Operator: 

>kubectl apply -f https://raw.githubusercontent.com/shayshahak/Rook-for-AKS/master/operator.yaml

verify the rook-ceph-operator, rook-ceph-agent, and rook-discover pods are in the `Running` state before proceeding (should take about two minutes):

> kubectl -n rook-ceph-system get pod

3.	Install the Rook Cluster:
>kubectl apply -f https://raw.githubusercontent.com/shayshahak/Rook-for-AKS/master/cluster.yaml

Use kubectl to list pods in the rook namespace. You should be able to see the following pods once they are all running:

>kubectl -n rook-ceph get pod

```NAME                                                   READY     STATUS      RESTARTS   
rook-ceph-mgr-a-5977cffcf8-vsmqz                       1/1       Running     0          
rook-ceph-mon-a-74dc9d4bcd-62g8s                       1/1       Running     0          
rook-ceph-mon-b-558cf6f9d4-7sh24                       1/1       Running     0          
rook-ceph-osd-0-585bf769cc-jp5sw                       1/1       Running     0          
rook-ceph-osd-1-76576b697f-v9zwz                       1/1       Running     0          
rook-ceph-osd-prepare-aks-agentpool-19295063-0-btkbb   0/2       Completed   0          
rook-ceph-osd-prepare-aks-agentpool-19295063-1-2fsfz   0/2       Completed   0          
```
4.	Install the Shared Files System:

>kubectl apply -f https://raw.githubusercontent.com/shayshahak/Rook-for-AKS/master/filesystem.yaml

To confirm the file system is configured, wait for the mds pods to start:

>kubectl -n rook-ceph get pod -l app=rook-ceph-mds

You should be able to see the following pods once they are all running:

```NAME                                    READY  STATUS    AGE  Node
rook-ceph-mds-myfs-a-5b7cbd4977-6tgv2   1/1    Running   1m   aks-agentpool-33026748-0
rook-ceph-mds-myfs-b-85dc94778c-8qfkb   1/1    Running   1m   aks-agentpool-33026748-1
```


that’s it! The Rook Cluster and the Shared File System are ready. Let’s deploy a simple WordPress application using what we just created:

> kubectl apply -f https://raw.githubusercontent.com/shayshahak/Rook-for-AKS/master/deploy.yaml

> kubectl apply -f https://raw.githubusercontent.com/shayshahak/Rook-for-AKS/master/service.yaml

>kubectl get svc wordpress-service

Next, connect to the WordPress application and configure its initial MySQL configuration.
Notice that /var/www/html is hosted on the Shared File System. Change the ‘replicas’ under the deployment Yaml and verify that in fact all of your Pods are using the same persistent storage.

## Summary
By using Rook on AKS, we achieved:
-	Scaling WordPress Frontend with a centralized persistent storage
-	Gained HA for the Frontend without data being bound to a single Pod nor to a single Node
-	Great performance in terms of end-to-end site response time
-	Used a well know Kubernetes integrated plugin

In this example we used Rook for a Shared File System but there are other things you can achieve with Rook such as Object and Block Storage for your containerized application running on Kubernetes. Rook provides additional operations tools such as its [Dashboard](https://rook.io/docs/rook/v0.9/ceph-dashboard.html) and [Monitoring](https://rook.io/docs/rook/v0.9/ceph-monitoring.html).

Note: these configurations examples are for testing and learning purposes. Do not use these configurations examples for production. For production consider changing the cluster properties, filesystem properties as well as the namespaces in which your cluster and application resides.
