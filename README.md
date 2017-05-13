# beegfsinazure
BeeGFS parallel file system in Azure

# Introduction

BeeGFS is a parallel filesystem solution for HPC cluster workloads running on premise or in cloud environments.

With Storage-, Metadata servers, Buddy Mirroring and a management service solution it provides vast ranges of configuration options, to accomodate almost any design requirement, with easily scaleout options.

This article is a How-To article with templates to deploy a High-Available BeeGFS cluster in Azure. 
The core components are basic Azure resources with properties easy to modify. The article also contains some test results.

The solution was implemented in the Azure West Europe region, on standard services, like Virtual Machines, Storage accounts.
The high level design looks like the following: 

![N|BeeGFS](https://image.ibb.co/d2QnJ5/BeeGFS.png)

Within a single region, I have deployed a single Resource Group with a VNET, within that VNET a Subnet. If required the storage servers can be separeted to another subnet from the clients. In a production environment I would advise to have them separated.
I have deployed a single management server, an Availability Set with two metadata- and another Availability set two storage nodes, along with two clients. In an production HPC cluster solution I aso would recomment to put the clients to an Availability set, for HA purposes.

- All the nodes were Standard D2_v2 with all of its limitations on IOPS, network throughput 
- The operating system was CentOS 7.3 for the storage environment.
- As a start a single metadata node with two 1 TB disks was deployed, later I have introduced a second metadata node with the same setup.
- There were two storage nodes with 4 standard 1TB disk / node.
- I have used 2 client nodes for test CentOS 6.8 and CentOS 7.3

# Installation & Configuration

I have installed the following packages as prerequisites on the environment.

    yum -y install epel-release zlib zlib-devel bzip2 bzip2-devel bzip2-libs \ 
    openssl openssl-devel openssl-libs gcc gcc-c++ nfs-utils rpcbind \
    wget python-pip kernel kernel-devel openmpi openmpi-devel automake autoconf

The following modofied kernel parameters are tuning the node performance. These prevent a neighbor table overflow system error from happening.

    echo "net.ipv4.neigh.default.gc_thresh1=1100" >> /etc/sysctl.conf
    echo "net.ipv4.neigh.default.gc_thresh2=2200" >> /etc/sysctl.conf
    echo "net.ipv4.neigh.default.gc_thresh3=4400" >> /etc/sysctl.conf

I have also disabled the frirewall on all nodes

    systemctl stop firewalld
    systemctl disable firewalld
And also disabled selinux

    sed -i 's/SELINUX=.*/SELINUX=permissive/g' /etc/selinux/config
    setenforce 0

## MGMT server

This host is used for administration and monitoring the running services

    wget -O /etc/yum.repos.d/beegfs-rhel7.repo http://www.beegfs.com/release/latest-stable/dists/beegfs-rhel7.repo yum install beegfs-mgmtd -y
    yum install beegfs-admon -y
    yum install beegfs-client beegfs-helperd beegfs-utils -y

After installation the storage location for the MGMT service has to be configured

    /opt/beegfs/sbin/beegfs-setup-mgmtd -p /data/beegfs/mgmtd
    sed -i 's/^sysMgmtdHost.*/sysMgmtdHost = 'beegfsmgmt'/g' /etc/beegfs/beegfs-admon.conf
    systemctl daemon-reload
    systemctl enable beegfs-mgmtd.service
    systemctl enable beegfs-admon.service
    systemctl start beegfs-mgmtd.service
    systemctl start beegfs-admon.service


## MetaData server

This server / these servers, servces are storing the metadata information. With moving this off from a storage node the over throughput performance can be much better.

    wget -O /etc/yum.repos.d/beegfs-rhel7.repo http://www.beegfs.com/release/latest-stable/dists/beegfs-rhel7.repo
    yum install beegfs-meta -y

On the metadata server we have to initialize the disks. A striped LVM will be perfect for the purpose.
    
    pvcreate /dev/sd[c-d]
    vgcreate vg00  /dev/sd[c-d]
    lvcreate --extents 100%FREE --stripes 2 --stripesize 4096 --name lv_metadata vg00
    mkfs.ext4 -i 2048 -I 512 -J size=400 -Odir_index,filetype /dev/mapper/vg00-lv_metadata
    tune2fs -o user_xattr /dev/mapper/vg00-lv_metadata
    echo “/dev/mapper/vg00-lv_metadata /data/beegfs/meta  ext4     noatime,nodiratime,nobarrier,nofail 0 2” >> /etc/fstab
    mkdir /data/beegfs/meta
    mount /data/beegfs/meta
    
We also have to configure where the metadata can be stored and where is the management service running. Imoportant to use a unique **NODEID**!
    
    /opt/beegfs/sbin/beegfs-setup-meta -p /data/beegfs/meta -s NODEID -m beegfsmgmt
There are also some parameters which are recommended to be tuned. These are available at the BeeGFS Wiki site as well.
To raise the allowed number of connections from a client change the following property.

    sed -i 's/^connMaxInternodeNum.*/connMaxInternodeNum = 800/g' /etc/beegfs/beegfs-meta.conf
To tune the max number of threads modify the following property. This allows larger number of (100-200) clients as well to connect.

    sed -i 's/^tuneNumWorkers.*/tuneNumWorkers = 128/g' /etc/beegfs/beegfs-meta.conf
Reload systemctl, enable, start the service
    
    systemctl daemon-reload
    systemctl enable beegfs-meta.service
    systemctl start beegfs-meta.service


## Storage server

Storage servers are responsible to store the file chunks in the cluster. It is worth to mention here, that the file chunks are stored distributed on all the storage servers by default (RAID 0 like operation). It is possible to implement a High Availabel solution with Buddy Mirroring.

    wget -O /etc/yum.repos.d/beegfs-rhel7.repo http://www.beegfs.com/release/latest-stable/dists/beegfs-rhel7.repo
    yum install beegfs-storage -y

On the storage server it is also sufficient to create a raid0 stripe with or LVM.
    
    pvcreate /dev/sd[c-f]
    vgcreate vg00  /dev/sd[c-f]
    lvcreate --extents 100%FREE --stripes 4 --stripesize 4096 --name lv_storage01 vg00
    mkfs -t xfs /dev/mapper/vg00-lv_storage01
    echo “/dev/mapper/vg00-lv_storage01 /data/beegfs/storage xfs rw,noatime,attr2,inode64,nobarrier,sunit=1024,swidth=4096,nofail 0 2” >> /etc/fstab
We have to configure where the files can be stored and where is the management service running to register the machine to the management server. It is important to cofigure unique **NODEID** and **TARGETID** values!

    /opt/beegfs/sbin/beegfs-setup-storage -p /data/beegfs/storage -s NODEID -i TARGETID -m beegfsmgmt

Some of the parameters also recommended to be tuned to have better performance with large number of clients.

    sed -i 's/^connMaxInternodeNum.*/connMaxInternodeNum = 800/g' /etc/beegfs/beegfs-storage.conf
    sed -i 's/^tuneNumWorkers.*/tuneNumWorkers = 128/g' /etc/beegfs/beegfs-storage.conf
    sed -i 's/^tuneFileReadAheadSize.*/tuneFileReadAheadSize = 32m/g' /etc/beegfs/beegfs-storage.conf
    sed -i 's/^tuneFileReadAheadTriggerSize.*/tuneFileReadAheadTriggerSize = 2m/g' /etc/beegfs/beegfs-storage.conf
    sed -i 's/^tuneFileReadSize.*/tuneFileReadSize = 256k/g' /etc/beegfs/beegfs-storage.conf
    sed -i 's/^tuneFileWriteSize.*/tuneFileWriteSize = 256k/g' /etc/beegfs/beegfs-storage.conf
    sed -i 's/^tuneWorkerBufSize.*/tuneWorkerBufSize = 16m/g' /etc/beegfs/beegfs-storage.conf
reload systemctl and enable, start the service

    systemctl daemon-reload
    systemctl enable beegfs-storage.service
    systemctl start beegfs-storage.service

## Client machine

It is possible to use NFS to mount BeeGFS shares, however in that case we completley loose the parallel file system benefits (like file coherency in less than a second). It is advised to use the BeeGFS client to mount the shares.

    wget -O /etc/yum.repos.d/beegfs-rhel7.repo http://www.beegfs.com/release/latest-stable/dists/beegfs-rhel7.repo
    yum install -y beegfs-client beegfs-helperd beegfs-utils 
    sed -i 's/^sysMgmtdHost.*/sysMgmtdHost = 'beegfsmgmt'/g' /etc/beegfs/beegfs-client.conf
    echo "/mnt/beegfs /etc/beegfs/beegfs-client.conf" > /etc/beegfs/beegfs-mounts.conf
    mkdir -p /mnt/beegfs

    systemctl daemon-reload
    systemctl enable beegfs-helperd.service
    systemctl enable beegfs-client.service

# Management commands

To manage the services use sysctl commands like

    systemctl start/stop/status beegfs-mgmtd
    systemctl start/stop/status beegfs-meta
    systemctl start/stop/status beegfs-storage
    systemctl start/stop/status beegfs-helperd
    systemctl start/stop/status beegfs-client
To manage BeeGFS from the management node use the following commands

## List nodes
```sh
[root@beegfsmgmt ~]# beegfs-ctl --listnodes --nodetype=storage
   beegfs-storage02 [ID: 1]
   beegfs-storage01 [ID: 2]
```        
```sh
[root@beegfsmgmt ~]# beegfs-ctl --listnodes --nodetype=meta
        beegfs-meta01 [ID: 1]
```

## Listing metadata nodes after adding a new metadata server
```sh
[root@beegfsmgmt ~]# beegfs-ctl --listtargets --nodetype=meta --state
TargetID     Reachability  Consistency   NodeID
========     ============  ===========   ======
       1           Online         Good        1
       2           Online         Good        2
```

## List storage target
```sh
[root@beegfsmgmt ~]# beegfs-ctl --listtargets --nodetype=storage --state
TargetID     Reachability  Consistency   NodeID
========     ============  ===========   ======
     101           Online         Good        1
     201           Online         Good        2
```
```sh
[root@beegfsmgmt ~]# beegfs-ctl --listtargets --nodetype=meta --state
TargetID     Reachability  Consistency   NodeID
========     ============  ===========   ======
       1           Online         Good        1
```
# Mirroring/Buddy Groups


With Mirroring/Buddy groups it is possible to create a Highly Available storage solution. Without this in a case of a storage, metadata node failure, the whole storage becomes unavailable, corrupted.

## Enabling metadata mirroring

Before executing the following command, clients should unmount the share
```sh 
[root@beegfsmgmt ~]# beegfs-ctl –mirrormd
```
After executing the command, the metadata service on all metadata nodes should be restarted.

## Crerating buddy group

Manual definition of mirros/buddy group works with the following command, with automatic definiton I personally had some issues.
```sh
[root@beegfsmgmt ~]# beegfs-ctl --addmirrorgroup --nodetype=storage --primary=101 --secondary=201 --groupid=100
Mirror buddy group successfully set: groupID 100 -> target IDs 101, 201
```

## Creating metadata mirror group after adding a new metadata server


```sh 
[root@beegfsmgmt ~]# beegfs-ctl --addmirrorgroup --nodetype=meta --primary=1 --secondary=2 --groupid=200
```

### Checking buddy group status

```sh
[root@beegfsmgmt ~]# beegfs-ctl --listmirrorgroups --nodetype=storage
     BuddyGroupID   PrimaryTargetID SecondaryTargetID
     ============   =============== =================
              100               101               201
```

```sh
[root@beegfsmgmt ~]# beegfs-ctl --listtargets --mirrorgroups
    MirrorGroupID MGMemberType TargetID   NodeID
    ============= ============ ========   ======
              100      primary      101        1
              100    secondary      201        2
```
          
### Displaying metadata mirrorgroup

```sh
[root@beegfsmgmt ~]# beegfs-ctl --listmirrorgroups --nodetype=meta
     BuddyGroupID     PrimaryNodeID   SecondaryNodeID
     ============     =============   ===============
              200                 1                 2
```

## Configuring Storage striping
By default the storage striping is RAID 0 which does not provides any sort of redundancy on the storage layer.
### Checking the actual status
```sh
[root@beegfsmgmt ~]# beegfs-ctl --getentryinfo /mnt/beegfs
Path:
Mount: /mnt/beegfs
EntryID: root
Metadata node: beegfs-meta01 [ID: 1]
Stripe pattern details:
+ Type: RAID0
+ Chunksize: 512K
+ Number of storage targets: desired: 4
```

### Configuring buddy mirroring

If this is configured before any new files/directories are created on the mount, then it will be applied on all new files. 
If it is configured after, then it has to be enforced manually. So only will apply for new directories and files by default.
```sh
[root@beegfsmgmt ~]# beegfs-ctl --setpattern --numtargets=2 --chunksize=1m --buddymirror /mnt/beegfs
```
This command sets the stripe pattern to a buddy mirror group as the stripe target. 
```sh
[root@beegfsmgmt ~]# beegfs-ctl --setpattern --numtargets=2 --chunksize=1m --buddymirror /mnt/beegfs
New chunksize: 1048576
New number of storage targets: 2

Path:
Mount: /mnt/beegfs
Checking buddy mirror settings
beegfs-ctl --getentryinfo /mnt/beegfs
Path:
Mount: /mnt/beegfs
EntryID: root
Metadata node: beegfs-meta01 [ID: 1]
Stripe pattern details:
+ Type: Buddy Mirror
+ Chunksize: 1M
+ Number of storage targets: desired: 2
```
By creating a new directory we can check if this gets applied or not.
```sh 
[root@beegfsmgmt ~]# beegfs-ctl --getentryinfo /mnt/beegfs/smallfiles/
Path: /smallfiles
Mount: /mnt/beegfs
EntryID: 0-59070A97-1
Metadata node: beegfs-meta01 [ID: 1]
Stripe pattern details:
+ Type: Buddy Mirror
+ Chunksize: 1M
+ Number of storage targets: desired: 2
```

### Checking Metadata mirroring

#### Checking status
If you the mirroring is not configured properly you will see the following
```sh
[root@beegfsmgmt ~]#  beegfs-ctl --resyncstats --nodetype=meta --nodeid=2
**Job state: Not started**
# of discovered dirs: 0
# of discovery errors: 0
# of synced dirs: 0
# of synced files: 0
# of dir sync errors: 0
# of file sync errors: 0
# of client sessions to sync: 0
# of synced client sessions: 0
session sync error: No
# of modification objects synced: 0
# of modification sync errors: 0
```
After successful metadata mirror configuration, and service restart the following should be seen.
```sh
[root@beegfsmgmt ~]#  beegfs-ctl --resyncstats --nodetype=meta --nodeid=2
**Job state: Running
Job start time: Mon May  1 12:25:45 2017
# of discovered dirs: 2
# of discovery errors: 0
# of synced dirs: 0
# of synced files: 0
# of dir sync errors: 0
# of file sync errors: 0
# of client sessions to sync: 0
# of synced client sessions: 0
session sync error: No
# of modification objects synced: 0
# of modification sync errors: 0
```
If everything goes well after a few minutes metadata should be synced
```sh
[root@beegfsmgmt ~]# beegfs-ctl --resyncstats --nodetype=meta --nodeid=2
**Job state: Completed successfully
Job start time: Mon May  1 12:25:45 2017
Job end time: Mon May  1 12:26:27 2017
# of discovered dirs: 2
# of discovery errors: 0
# of synced dirs: 2
# of synced files: 5
# of dir sync errors: 0
# of file sync errors: 0
# of client sessions to sync: 0
# of synced client sessions: 0
session sync error: No
# of modification objects synced: 0
# of modification sync errors: 0
Restarting metadata sync
```
It is possible to restart sync operation if necessary
```sh
[root@beegfsmgmt ~]# beegfs-ctl --startresync --nodetype=metadata --nodeid=2 --restart
```

