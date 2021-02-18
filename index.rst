Introduction
============

Design the estructure and topology that the VMWare clusters - both Base and Summit - must be constructed. 
This must be a resilient and robust solution, so that it can run and support the core services, backup infrastructure, 
lsst repo and future other applications.


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

  - The OS (ESXi or vSphere) is mounted in a RAID1, construct over 2x1.6TB SSD, with a failure toleration of 1 SSD and a total pool storage of 1.6TB.
  - 2x RAID5, construct over 4x2TB SSD , with a failure toleration of 1 SSD per RAID and a storage pool of 6TB (each RAID).
  - The RAIDs - from now on, VD (Virtual Device) - are hardware build and directly mapped to the OS.
  - Over the local storage - the RAID1, common to the OS - a CentOS Virtual Machine is created.
  - To each Virtual Machine - one per server - the local VD are mapped into them.
  - The VMs are interconnected and a Gluster File System is created, composed by 6 Blocks - gluster storage metric, been 1 Block a volume storage, such as one HDD/SDD, VD and LVM. Only one Block is allowed per one of said units.
  - The newly created GlusterFS (Gluster File System), with a 3 Replica configuration, would have a Storage Pool of 12TB.
  - The Gluster Storage Pool is then mapped into vCenter Server, and serves as a common pool, for all servers, to allocate the VMs. 


.. sectnum::




