Introduction
============

Design the estructure and topology that the VMWare clusters - both Base and Summit - must be constructed. 
This must be a resilient and robust solution, so that it can run and support the core services, backup infrastructure, 
lsst repo and future other applications.


Motivation
==========

New technologies and developments are been discovered at an outstanding rythm. Virtualization became a standard for the industry many years ago, but it has been replaced or substitud by new technologies, like micro-services - containerize applications, but will not be the subject of this documen - but there are still reasons to keep certain amount of services running in a Virtual Machine instead of a Container.

Many of the services that are going to be mentioned as crucial to run in a Virtual Machine, can in fact be deployed in a container, but there are security and loops that are important to prevent:
  - Rubin Kubernetes clusters run over Fixed DHCP IP, meaning that the members of the cluster are set to use DHCP, and on the other side, the DHCP Server holds a DHCP reservation for the server's MAC Address, so it always will get the same one.
  - Same reasoning applies for DNS Servers; if the service doesn't exists until the container is created, it will create a conflic at kube-dns.
  - The first time servers are booted - and provisioned - requires an TFTP server for PXE booting.



Requirements
============

So that a cluster is called resilient and robust, there are certain criteria and recommendations that have to be met:

  - The Cluster size has to be an odd number. 
  - A common storage pool that all cluster nodes can access at all time.
  - Said storage pool, has to be able to tolerate catastrophic events, such us disk failures, servers maintanance or failure, network outage and prevent data corruption.
  - Each node must be able to be taken offline or shutdown, allowing all services and Virtual Machines to remain ongoing.
  - Each node must have the capacity to run, single handed, the Virtual Machines of the whole cluster.
  - If a node is abruptly taken offline, the Virtual Machines running in such node must be able and capable of live auto-migration.
  - If a node irrevocably shuts down - complete hardware failure - the node must be easily replaced with a new one, with no data corruption.


Design
======

The following diagram shows the proposed architecture that will sustain the Virtualization Platform in vCenter:

  .. figure:: /_static/vmware_cluster_design.png
     :name: vmware_cluster_design

     VMWare Cluster Design

  - The OS (ESXi or vSphere) is mounted in a RAID1, construct over 2x1.6TB SSD, with a failure toleration of 1 SSD and a total pool storage of 1.6TB.
  - 2x RAID5, construct over 4x2TB SSD , with a failure toleration of 1 SSD per RAID and a storage pool of 6TB (each RAID).
  - The RAIDs - from now on, VB (Virtual Block) - are hardware build and directly mapped to the OS.
  - Over the local storage - the RAID1, common to the OS - a CentOS Virtual Machine is created.
  - To each Virtual Machine - one per server - the local VB are mapped into them.
  - The VMs are interconnected and a Gluster File System is created, composed by 6 Blocks - gluster storage metric, been 1 Block a volume storage, such as one HDD/SDD, VB, VD and LVM. Only one Block is allowed per one of said units.
  - The newly created GlusterFS (Gluster File System), with a 3 Replica configuration, would have a Storage Pool of 12TB.
  - The Gluster Storage Pool is then mapped into vCenter Server, and serves as a common pool, for all servers, to allocate the VMs. 


Failover and WCS
================

WCS stands for Worst Case Scenario. The suggested design takes under consideration the following scenarios:


System Disk Failure
-------------------

In the first scenario, we consider the loss of one of the two System Disks. By System Disks we are reffering to the drives from were the OS is mounted on. Also, in this case, were one of the Gluster's VM will be allocated:


  .. figure:: /_static/system_disk_failure.png
     :name: system_disk_failure

     Losing one System Disk


Since the System Disks are arrange in a RAID1, the system will continue to go one and a hot-swap can be performed to replace the fail drive. No downtime necessarry.


Data Disks Failure
------------------

The Data Disks consists on were the Virtual Machines "virtual drives" are allocate. Virtual Drive is a logical drive created by the virtualization agent, in which reserves a space disk on the common storage, but from the VM perspective, is a regular drive.

  .. figure:: /_static/data_disk_failure.png
     :name: data_disk_failure.png

     Losing one Data Disk

Similar to the RAID1, the RAID5 configuration used in the Data Disks tolerate 1 Drive per RAID arrangement. This means, that we can loose 2 drives per server - only one per RAID - and keep working as usual, allowing to perform hot-swap as well and no downtime.


1 VM or Server loss
-------------------

The base OS runs a modified Linux Kernel, that allows to run many tasks and services, but they are pretty limited for external services - externals to vSphere -, which is why a Virtual Machine is mounted on top of the System Disks, to be able to run services - such as gluster - and be able to monitor it as well.

  .. figure:: /_static/server_failure.png
     :name: server_failure.png

     Server Failure

What this means, is that is basically the same - only in this setup - to loose a server than to loose a VM. Since the replication of gluster is set to 3 - keeps one copy per node -, in case one Server/VM is powered off or distroyed, the gluster storage would still have quorum and the data would remain uncorrupted. The repair procedure does not contemplate timeout or downtime, but since there is going to be a remaping and data duplication when the Server/VM comes back online, stressfull operation with high I/O - such VMs creation - are not recommended to be performed.


Network Outage
--------------

In order to explain what would happen during a Network Outage, the "Network Layer" was add to the diagram:

  .. figure:: /_static/vmware_cluster_with_network.png
     :name: vmware_cluster_with_network.png

     VMWare Cluster with Network Connection

Each link - numbers 1, 2 and 3 over the left of the servers - are composed by 2 connections: A primary and a failover. Keep in mind this are not PC or VPC (Port-Channel or Virtual Port-Channel), but a failover, meaning that if the primary link is lost, the failover kicks in. 

If both links - primary and secondary - are lost, we face a similar scenario than a Server Loss with a sutil difference: when the network connection to a server is lost, a new instance of the Virtual Machines that were running in that server, will be replicated into one of the others, but the one that was running will remain running. This will produce that the local data - from the gluster VM - and the data from the other nodes - the other gluster VMs - will form a discrepancy. Fortunately, Gluster operates in a quorum base, which means that if two out of three nodes have the same data, the data is overwritten in the one that differs, so when the Server comes back online, a syncronization process will start and the Virtual Machines that were migrated, will be destroyed in the recently recover node. This mechanism is provided by the vCenter VMWare platform called vMotion, that ensures that the Virtual Machines are always running and auto-migrate them if any of the mentioned events happened.


2 VMs or Servers loss
---------------------

As mentioned before, Gluster is based on a quorum algorithm, which means that if two out of three nodes are unreachable or down but still reachable whitin each other, there is a high chance of data corruption.

The failsafe mechanisms that gluster uses here, are based on: "I cannot reach one of my agents and I'm getting timeouts to the network" in the 2 isolated nodes, and in the one still connected "I don't have a quorum, due to only one of the three nodes is available", then what happens is the gluster storage pool will fall in a state called "Read Only" to prevent data corruption.

vMotion is going to attempt to migrate the Virtual Machines from the fallen servers to the live one, but gluster won't allow it in order to prevent data corruption or a phenomena called "Split-Brain". Split-Brain happens when the metadata from one node differs from another, and it takes an arbitrary node to act as arbiter.

In this fatalistic scenario, if the at least one of the servers can be placed back online, the gluster storage will start again and the live-migration will begin; but if neither of the two servers are recoverable, the only option is redeem the data, reconstruct the gluster and drop the data on top of the new gluster. This will cause an outage and downtime.


.. sectnum::
