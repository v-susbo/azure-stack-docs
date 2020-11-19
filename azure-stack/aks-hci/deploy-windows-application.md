---
title: Tutorial - Deploy a Windows application and join Windows nodes to Active Directory
description: In this tutorial, you deploy a Windows application to your cluster using a custom image stored in Azure Container Registry. You will also learn how to join a Windows nodes to an Active Directory domain.
author: abha
ms.topic: tutorial
ms.date: 09/22/2020
ms.author: abha
ms.reviewer: 
---

# Tutorial: Deploy Windows applications in Azure Kubernetes Service on Azure Stack HCI

In this tutorial, you deploy an ASP.NET sample application in a Windows Server container to the Kubernetes cluster. You then see how to test and scale your application. 
This tutorial assumes a basic understanding of Kubernetes concepts. For more information, see [Kubernetes core concepts for Azure Kubernetes Service on Azure Stack HCI](kubernetes-concepts.md).

## Before you begin

Verify you have the following requirements ready:

* An Azure Kubernetes Service on Azure Stack HCI cluster with at least one Windows worker node that is up and running. 
* A kubeconfig file to access the cluster.
* Have the Azure Kubernetes Service on Azure Stack HCI PowerShell module installed.
* Run the commands in this document in a PowerShell administrative window.
* Ensure that OS specific workloads land on the appropriate container host. If you have a mixed Linux and Windows worker nodes Kubernetes cluster, you can either use node selectors or taints and tolerations. For more information, see [using node selectors and taints and tolerations](adapt-apps-mixed-os-clusters.md).

## Deploy the application

A Kubernetes manifest file defines a desired state for the cluster, such as what container images to run. In this article, a manifest is used to create all objects needed to run the ASP.NET sample application in a Windows Server container. This manifest includes a Kubernetes deployment for the ASP.NET sample application and an external Kubernetes service to access the application from the internet.

The ASP.NET sample application is provided as part of the .NET Framework Samples and runs in a Windows Server container. Azure Kubernetes Service on Azure Stack HCI requires Windows Server containers to be based on images of *Windows Server 2019*. 

The Kubernetes manifest file must also define a node selector to tell your AKS cluster to run your ASP.NET sample application's pod on a node that can run Windows Server containers.

Create a file named `sample.yaml` and copy in the following YAML definition. 

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample
  labels:
    app: sample
spec:
  replicas: 1
  template:
    metadata:
      name: sample
      labels:
        app: sample
    spec:
      nodeSelector:
        "beta.kubernetes.io/os": windows
      containers:
      - name: sample
        image: mcr.microsoft.com/dotnet/framework/samples:aspnetapp
        resources:
          limits:
            cpu: 1
            memory: 800M
          requests:
            cpu: .1
            memory: 300M
        ports:
          - containerPort: 80
  selector:
    matchLabels:
      app: sample
---
apiVersion: v1
kind: Service
metadata:
  name: sample
spec:
  type: LoadBalancer
  ports:
  - protocol: TCP
    port: 80
  selector:
    app: sample
```

Deploy the application using the `kubectl apply` command and specify the name of your YAML manifest:

```console
kubectl apply -f sample.yaml
```

The following example output shows the deployment and service created successfully:

```output
deployment.apps/sample created
service/sample created
```

## Test the application

When the application runs, a Kubernetes service exposes the application front end to the internet. This process can take a few minutes to complete. Occasionally the service can take longer than a few minutes to provision. Allow up to 10 minutes in these cases.

To monitor progress, use the `kubectl get service` command with the `--watch` argument.

```PowerShell
kubectl get service sample --watch
```

Initially the *EXTERNAL-IP* for the *sample* service is shown as *pending*.

```output
NAME    TYPE           CLUSTER-IP   EXTERNAL-IP   PORT(S)        AGE
sample  LoadBalancer   10.0.37.27   <pending>     80:30572/TCP   6s
```

When the *EXTERNAL-IP* address changes from *pending* to an actual public IP address, use `CTRL-C` to stop the `kubectl` watch process. The following example output shows a valid public IP address assigned to the service:

```output
NAME    TYPE           CLUSTER-IP   EXTERNAL-IP     PORT(S)        AGE
sample  LoadBalancer   10.0.37.27   52.179.23.131   80:30572/TCP   2m
```

To see the sample app in action, open a web browser to the external IP address of your service.

![Image of browsing to ASP.NET sample application](media/deploy-windows-application/asp-net-sample-app.png)

If you receive a connection timeout when trying to load the page, verify if the sample app is ready with `kubectl get pods --watch` command. Sometimes, the external IP address is available before the windows container has started.

## Scale application pods

We have created a single replica of the application front-end. To see the number and state of pods in your cluster, use the `kubectl get` command as follows:

```console
kubectl get pods -n default
```

To change the number of pods in the *sample* deployment, use the `kubectl scale` command. The following example increases the number of front-end pods to *3*:

```console
kubectl scale --replicas=3 deployment/sample
```

Run `kubectl get pods` again to verify that additional pods have been created. After a minute or so, the additional pods are available in your cluster:

```console
kubectl get pods -n default
```

## Prepare Windows nodes for group Managed Service Account support on Windows nodes

Group Managed Service Accounts are a specific type of Active Directory account that provides automatic password management, simplified service principal name (SPN) management, and the ability to delegate the management to other administrators across multiple servers. To configure group Managed Service Accounts (gMSA) for pods and containers that will run on your Windows nodes, you first have to join your Windows nodes to an Active Directory domain.

To join your Windows worker nodes to a domain, log in to a Windows worker node, by running `kubectl get` and noting the `EXTERNAL-IP` value.

```PowerShell
kubectl get nodes -o wide
``` 

You can then SSH into the node using `ssh Administrator@ip`. 

After you've successfully logged in to your Windows worker node, run the following PowerShell command to join the node to a domain. You'll be prompted to enter your **domain administrator account** credentials. You can also use elevated user credentials that have been given rights to join computers to the given domain. You'll then need to reboot your Windows worker node.

```PowerShell
add-computer --domainame "YourDomainName" -restart
```

Once all Windows worker nodes have been joined to a domain, follow the steps detailed at [configuring gMSA](https://kubernetes.io/docs/tasks/configure-pod-container/configure-gmsa) to apply the Kubernetes gMSA custom resource definitions and webhooks on your Kubernetes cluster.

For more information on Windows container with gMSA, refer [Windows containers and gMSA](/virtualization/windowscontainers/manage-containers/manage-serviceaccounts). 


## Next steps

* [Use Azure Monitor to monitor your cluster and application](/azure/azure-monitor/insights/container-insights-enable-arc-enabled-clusters).
* [Use persistent storage on a Kubernetes cluster](persistent-storage.md).
