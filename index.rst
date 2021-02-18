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



.. sectnum::




