---
title: Taking an Azure Stack HCI server offline for maintenance
description: This topic provides guidance on how to properly pause, drain, and resume servers running the Azure Stack HCI operating system using Windows Admin Center and PowerShell.
author: khdownie
ms.author: v-kedow
ms.topic: how-to
ms.service: azure-stack
ms.subservice: azure-stack-hci
ms.date: 10/23/2020
---

# Taking an Azure Stack HCI server offline for maintenance

> Applies to: Azure Stack HCI, version 20H2; Windows Server 2019

With Azure Stack HCI, taking a server offline for maintenance requires taking portions of storage offline that are shared across all servers in the cluster. This requires pausing the server that you want to take offline, moving roles and virtual machines (VMs) to other servers in the cluster, and verifying that all data is available on the other servers in the cluster. This process ensures that the data remains safe and accessible throughout the maintenance period.

You can either use either Windows Admin Center or PowerShell to take a server offline for maintenance. This topic covers both methods.

   > [!IMPORTANT]
   > This topic assumes that you need to power down a physical server down to perform maintenance, or restart it for some other reason. To install updates on an Azure Stack HCI cluster, see [Update Azure Stack HCI clusters](update-cluster.md), which explains how to use Cluster-Aware Updating (CAU) to automatically perform all the steps in this topic while also updating servers and restarting them if necessary.

## Take a server offline using Windows Admin Center

The simplest way to prepare to take a server in an Azure Stack HCI cluster offline is by using Windows Admin Center.

### Verify it's safe to take the server offline

1. Using Windows Admin Center, connect to the server you want to take offline. Select **Storage > Disks** from the **Tools** menu, and verify that the **Status** column for every virtual disk shows **Online**.

2. Then, select **Storage > Volumes** and verify that the **Health** column for every volume shows **Healthy** and that the **Status** column for every volume shows **OK**.

### Pause and drain the server

Before either shutting down or restarting a server, you should pause the server and drain (move off) any clustered roles such as VMs running on it to ensure that the server shutdown does not affect application state. Always pause and drain clustered servers before taking them offline for maintenance.

1. Using Windows Admin Center, connect to the cluster and then select **Compute > Nodes** from the **Tools** menu in Cluster Manager.

2. Click on the name of the server you wish to pause and drain, and select **Pause**. You should see the following prompt:

   *If you pause this node, all clustered roles move to other nodes and no roles can be added to this node until it's resumed. Are you sure you want to pause cluster node?*

3. Select **yes** to pause the server and initiate the drain process. The cluster node status will show as **Draining**, and roles such as Hyper-V and VMs will immediately begin live migrating to other server(s) in the cluster. This can take a few minutes.

   > [!NOTE]
   > When you pause and drain the server properly, Azure Stack HCI performs an automatic safety check to ensure it is safe to proceed. If there are unhealthy volumes, it will stop and alert you that it's not safe to proceed.

### Shut down the server

Once the server has completed draining, it status will show as **Paused** in Windows Admin Center. You can now safely shut the server down for maintenance or reboot it.

### Resume the server

When you are ready for the server to begin hosting clustered roles and VMs again, simply turn the server on, wait for it to boot up, and resume the server using the following steps.

1. In Cluster Manager, select **Compute > Nodes** from the **Tools** menu at the left.

2. Select the name of the server you wish to resume, and then click **Resume**. You should see the following prompt:

   *Are you sure you want to resume cluster node?*

3. In most situations, you should select the checkbox that says *Transfer clustered roles back to this node*. Select **yes** to resume the server.

If you checked the box in step 3 above, clustered roles and VMs will immediately begin live migrating back to the server. This can take a few minutes.

### Wait for storage to resync

When the server resumes, any new writes that happened while it was unavailable need to resync. This happens automatically, using intelligent change tracking. It's not necessary for *all* data to be scanned or synchronized; only the changes. This process is throttled to mitigate impact to production workloads. Depending on how long the server was paused and how much new data was written, it may take many minutes to complete.

   > [!IMPORTANT]
   > You must wait for re-syncing to complete before taking any other servers in the cluster offline.

To check if resyncing has completed, connect to the server using Windows Admin Center and select **Storage > Volumes** from the **Tools** menu at the left, then select **Volumes** near the top of the page. If the **Health** column for every volume shows **Healthy** and the **Status** column for every volume shows **OK**, then re-syncing has completed, and it's now safe to take other servers in the cluster offline.

## Take a server offline using PowerShell

Use the following procedures to properly pause, drain, and resume a server in an Azure Stack HCI cluster using PowerShell.

### Verify it's safe to take the server offline

To verify that all your volumes are healthy, run the following cmdlet as an administrator:

```PowerShell
Get-VirtualDisk
```

Here's an example of what the output might look like:

```
FriendlyName              ResiliencySettingName FaultDomainRedundancy OperationalStatus HealthStatus    Size FootprintOnPool StorageEfficiency
------------              --------------------- --------------------- ----------------- ------------    ---- --------------- -----------------
Mirror II                 Mirror                1                     OK                Healthy         4 TB         8.01 TB            49.99%
Mirror-accelerated parity                                             OK                Healthy      1002 GB         1.96 TB            49.98%
Mirror                    Mirror                1                     OK                Healthy         1 TB            2 TB            49.98%
ClusterPerformanceHistory Mirror                1                     OK                Healthy        24 GB           49 GB            48.98%
```

Verify that the **HealthStatus** property for every volume is **Healthy** and the **OperationalStatus** shows OK.

### Pause and drain the server

Run the following cmdlet as an administrator to pause and drain the server:

```PowerShell
Suspend-ClusterNode -Drain
```

### Shut down the server

Once the server has completed draining, it will show as **Paused** in PowerShell.

You can now safely shut the server down or restart it by using the `Stop-Computer` or `Restart-Computer` PowerShell cmdlets.

   > [!NOTE]
   > When running a `Get-VirtualDisk` command on servers that are shutting down or starting/stopping the cluster service, the server's Operational Status may be reported as incomplete or degraded, and the Health Status column may list a warning. This is normal and should not cause concern. All your volumes remain online and accessible.

### Resume the server

Run the following cmdlet as an administrator to resume the server into the cluster. To return the clustered roles and VMs that were previously running on the server, use the optional **-Failback** flag:

```PowerShell
Resume-ClusterNode –Failback Immediate
```

Once the server has resumed, it will show as **Up** in PowerShell.

### Wait for storage to resync

When the server resumes, you must wait for re-syncing to complete before taking any other servers in the cluster offline.

Run the following cmdlet as administrator to monitor progress:

```PowerShell
Get-StorageJob
```

If re-syncing has already completed, you won't get any output.

Here's some example output showing resync (repair) jobs still running:

```
Name   IsBackgroundTask ElapsedTime JobState  PercentComplete BytesProcessed BytesTotal
----   ---------------- ----------- --------  --------------- -------------- ----------
Repair True             00:06:23    Running   65              11477975040    17448304640
Repair True             00:06:40    Running   66              15987900416    23890755584
Repair True             00:06:52    Running   68              20104802841    22104819713
```

The **BytesTotal** column shows how much storage needs to resync. The **PercentComplete** column displays progress.

   > [!WARNING]
   > It's not safe to take another server offline until these repair jobs finish.

During this time, under **HealthStatus**, your volumes will continue to show as **Warning**, which is normal.

For example, if you use the `Get-VirtualDisk` cmdlet while storage is re-syncing, you might see the following output:

```
FriendlyName ResiliencySettingName OperationalStatus HealthStatus IsManualAttach Size
------------ --------------------- ----------------- ------------ -------------- ----
MyVolume1    Mirror                InService         Warning      True           1 TB
MyVolume2    Mirror                InService         Warning      True           1 TB
MyVolume3    Mirror                InService         Warning      True           1 TB
```

Once the jobs complete, verify that volumes show **Healthy** again by using the `Get-VirtualDisk` cmdlet. Here's some example output:

```
FriendlyName ResiliencySettingName OperationalStatus HealthStatus IsManualAttach Size
------------ --------------------- ----------------- ------------ -------------- ----
MyVolume1    Mirror                OK                Healthy      True           1 TB
MyVolume2    Mirror                OK                Healthy      True           1 TB
MyVolume3    Mirror                OK                Healthy      True           1 TB
```

It's now safe to pause and restart other servers in the cluster.

## Next steps

For related information, see also:

- [Storage Spaces Direct overview](/windows-server/storage/storage-spaces/storage-spaces-direct-overview)
- [Update Azure Stack HCI clusters](update-cluster.md)