
Small-Scale Setup
=================

In a small-scale setup, a single NFS server can function as both primary and secondary storage. The NFS
server must export two separate shares, one for primary storage and the other for secondary storage. This
could be a VM or physical host running an NFS service on a Linux OS or a virtual software appliance. Disk
and network performance are still important in a small scale setup to get a good experience when deploying,
running or snapshotting VMs.


Large-Scale Setup
=================

In large-scale environments primary and secondary storage typically consist of independent physical storage arrays.

Primary storage is likely to have to support mostly random read/write I/O once a template has been
deployed.  Secondary storage is only going to experience sustained sequential reads or writes.

In clouds which will experience a large number of users taking snapshots or deploying VMs at the
same time, secondary storage performance will be important to maintain a good user experience.

It is important to start the design of your storage with the a rough profile of the workloads which it will
be required to support. Care should be taken to consider the IOPS demands of your guest VMs as much as the
volume of data to be stored and the bandwidth (MB/s) available at the storage interfaces.

Storage Architecture
====================

There are many different storage types available which are generally suitable for CloudStack environments.
Specific use cases should be considered when deciding the best one for your environment and financial
constraints often make the 'perfect' storage architecture economically unrealistic.

Broadly, the architectures of the available primary storage types can be split into 3 types:

Local Storage
-------------

Local storage works best for pure 'cloud-era' workloads which rarely need to be migrated between storage
pools and where HA of individual VMs is not required. As SSDs become more mainstream/affordable, local
storage based VMs can now be served with the size of IOPS which previously could only be generated by
large arrays with 10s of spindles. Local storage is highly scalable because as you add hosts you would
add the same proportion of storage. Local Storage is relatively inefficent as it can not take advantage
of linked clones or any deduplication.


'Traditional' node-based Shared Storage
---------------------------------------

Traditional node-based storage are arrays which consist of a controller/controller pair attached to a
number of disks in shelves.
Ideally a cloud architecture would have one of these physical arrays per CloudStack pod to limit the
'blast-radius' of a failure to a single pod.  This is often not economically viable, however one should
look to try to reduce the scale of any incident relative to any zone with any single array where
possible.
The use of shared storage enables workloads to be immediately restarted on an alternate host should a
host fail. These shared storage arrays often have the ability to create 'tiers' of storage utilising
say large SATA disks, 15k SAS disks and SSDs. These differently performing tiers can then be presented as
different offerings to users.
The sizing of an array should take into account the IOPS required by the workload as well as the volume
of data to be stored.  One should also consider the number of VMs which a storage array will be expected
to support, and the maximum network bandwidth possible through the controllers.


Clustered Shared Storage
------------------------

Clustered shared storage arrays are the new generation of storage which do not have a single set of
interfaces where data enters and exits the array.  Instead it is distributed between all of the active
nodes giving greatly improved scalability and performance.  Some shared storage arrays enable all data
to continue to be accessible even in the event of the loss of an entire node.

The network topology should be carefully considered when using clustered shared storage to avoid creating
bottlenecks in the network fabric.


Network Configuration For Storage
=================================

Care should be taken when designing your cloud to take into consideration not only the performance
of your disk arrays but also the bandwidth available to move that traffic between the switch fabric and
the array interfaces.

CloudStack Networking For Storage
---------------------------------

The first thing to understand is the process of provisioning primary storage. When you create a primary
storage pool for any given cluster, the CloudStack management server tells each hosts’ hypervisor to
mount the NFS share or (iSCSI LUN). The storage pool will be presented within the hypervisor as a
datastore (VMware), storage repository (XenServer/XCP) or a mount point (KVM), the important point is
that it is the hypervisor itself that communicates with the primary storage, the CloudStack management
server only communicates with the host hypervisor. Now, all hypervisors communicate with the outside
world via some kind of management interface – think VMKernel port on ESXi or ‘Management Interface’ on
XenServer. As the CloudStack management server needs to communicate with the hypervisor in the host,
this management interface must be on the CloudStack ‘management’ or ‘private’ network.  There may be
other interfaces configured on your host carrying guest and public traffic to/from VMs within the hosts
but the hypervisor itself doesn’t/can’t communicate over these interfaces.

|hypervisorcomms.png|
*Figure 1*: Hypervisor communications

Separating Primary Storage traffic
For those from a pure virtualisation background, the concept of creating a specific interface for storage
traffic will not be new; it has long been best practice for iSCSI traffic to have a dedicated switch
fabric to avoid any latency or contention issues.
Sometimes in the cloud(Stack) world we forget that we are simply orchestrating processes that the
hypervisors already carry out and that many ‘normal’ hypervisor configurations still apply.
The logical reasoning which explains how this splitting of traffic works is as follows:

1. If you want an additional interface over which the hypervisor can communicate (excluding teamed or bonded interfaces) you need to give it an IP address.
#. The mechanism to create an additional interface that the hypervisor can use is to create an additional management interface
#. So that the hypervisor can differentiate between the management interfaces they have to be in different (non-overlapping) subnets
#. In order for the ‘primary storage’ management interface to communicate with the primary storage, the interfaces on the primary storage arrays must be in the same CIDR as the ‘primary storage’ management interface.
#. Therefore the primary storage must be in a different subnet to the management network

|subnetting storage.png|
*Figure 2*: Subnetting of Storage Traffic

|hypervisorcomms-secstorage.png|
*Figure 3*: Hypervisor Communications with Separated Storage Traffic

Other Primary Storage Types
If you are using PreSetup or SharedMountPoints to connect to IP based storage then the same principles
apply; if the primary storage and ‘primary storage interface’ are in a different subnet to the ‘management
subnet’ then the hypervisor will use the ‘primary storage interface’ to communicate with the primary
storage.