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

# Tests
## Performce
### Smallfiles test 1
Generating 16GBs of files with 8 threads, in a single metadata node dual storage node configuration on a RAID0 storage share with mirroring.
```sh
[root@beegfs-cli01 smallfile-master]# python smallfile_cli.py --operation create --threads 8 --file-size 1024 --files 2048 --top /mnt/beegfs/smallfiles

140.674193 sec elapsed time
108.605563 files/sec
108.605563 IOPS
108.605563 MB/sec
```
Storage utilization.
```sh
[root@beegfs-cli01 smallfile-master]# df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda2        30G  1.7G   28G   6% /
devtmpfs        3.4G     0  3.4G   0% /dev
tmpfs           3.5G     0  3.5G   0% /dev/shm
tmpfs           3.5G   17M  3.4G   1% /run
tmpfs           3.5G     0  3.5G   0% /sys/fs/cgroup
/dev/sda1       497M   62M  436M  13% /boot
/dev/sdb1        99G   61M   94G   1% /mnt/resource
tmpfs           697M     0  697M   0% /run/user/1000
beegfs_nodev    8.0T   33G  8.0T   1% /mnt/beegfs
```
From the tests it seems that in Buddy Mirroring the storage space is halved, files are using twice as much space from the available storage. What completley makes sense as we are having a RAID1 like cluster.

### Smallfiles test 2
Generating 16GBs of files with 8 threads, in a dual metadata node dual storage node configuration
```sh
[root@beegfs-cli01 smallfile-master]# python smallfile_cli.py --operation create --threads 8 --file-size 1024 --files 2048 --top /mnt/beegfs/smallfiles

122.164110 sec elapsed time
134.114676 files/sec
134.114676 IOPS
134.114676 MB/sec
```
Read test:
```sh
176.852571 sec elapsed time
90.595234 files/sec
90.595234 IOPS
90.595234 MB/sec
```
Append test
```sh
116.225757 sec elapsed time
139.211827 files/sec
139.211827 IOPS
139.211827 MB/sec
```
## High Availability test
I have tested HA failures on a Storage and Metadata node level as well.

### Storage node failure test
Downloaded a CentOS image
```sh
871d931b7d71cf5ecededb03c76583de  CentOS-7-x86_64-DVD-1611.iso
```
Then stopped one of the storage nodes. (Notice that the available space went down to 4 TB from 8 TB.
```sh
[root@beegfs-cli01 beegfs]# df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda2        30G  1.7G   28G   6% /
devtmpfs        3.4G     0  3.4G   0% /dev
tmpfs           3.5G     0  3.5G   0% /dev/shm
tmpfs           3.5G   17M  3.4G   1% /run
tmpfs           3.5G     0  3.5G   0% /sys/fs/cgroup
/dev/sda1       497M   62M  436M  13% /boot
/dev/sdb1        99G   61M   94G   1% /mnt/resource
tmpfs           697M     0  697M   0% /run/user/1000
beegfs_nodev    4.0T   21G  4.0T   1% /mnt/beegfs
```
It takes about 1-2 minutes to identify the node failure
```sh
[root@beegfsmgmt ~]# beegfs-ctl --listtargets --nodetype=storage --state
TargetID     Reachability  Consistency   NodeID
========     ============  ===========   ======
     101 Probably-offline         Good        1
     201           Online         Good        2
```
```sh
[root@beegfsmgmt ~]# beegfs-ctl --listtargets --nodetype=storage --state
TargetID     Reachability  Consistency   NodeID
========     ============  ===========   ======
     101          Offline         Good        1
     201           Online         Good        2
```
When checking the file md5sum gives back the hash on a single node configuration.
```sh
[root@beegfs-cli01 beegfs]# md5sum CentOS-7-x86_64-DVD-1611.iso
871d931b7d71cf5ecededb03c76583de  CentOS-7-x86_64-DVD-1611.iso
```
#### Synchronization of data after failure

Downloading a new file and deleting one
```sh
[root@beegfs-cli01 beegfs]# wget http://mirrors.vooservers.com/centos/6.9/isos/x86_64/CentOS-6.9-x86_64-minimal.iso
--2017-05-01 11:06:13--  http://mirrors.vooservers.com/centos/6.9/isos/x86_64/CentOS-6.9-x86_64-minimal.iso
Resolving mirrors.vooservers.com (mirrors.vooservers.com)... 194.0.252.34, 2a00:1c10:3:634:0:1000:0:1
Connecting to mirrors.vooservers.com (mirrors.vooservers.com)|194.0.252.34|:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 427819008 (408M) [application/octet-stream]
Saving to: ‘CentOS-6.9-x86_64-minimal.iso’

100%[===============================================================================================>] 427,819,008 11.2MB/s   in 38s

2017-05-01 11:06:51 (10.7 MB/s) - ‘CentOS-6.9-x86_64-minimal.iso’ saved [427819008/427819008]

[root@beegfs-cli01 beegfs]# rm -rf CentOS-7-x86_64-DVD-1611.iso

[root@beegfs-cli01 beegfs]# df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda2        30G  1.7G   28G   6% /
devtmpfs        3.4G     0  3.4G   0% /dev
tmpfs           3.5G     0  3.5G   0% /dev/shm
tmpfs           3.5G   17M  3.4G   1% /run
tmpfs           3.5G     0  3.5G   0% /sys/fs/cgroup
/dev/sda1       497M   62M  436M  13% /boot
/dev/sdb1        99G   61M   94G   1% /mnt/resource
tmpfs           697M     0  697M   0% /run/user/1000
beegfs_nodev    4.0T  441M  4.0T   1% /mnt/beegfs

[root@beegfs-cli01 beegfs]# md5sum CentOS-6.9-x86_64-minimal.iso
af4a1640c0c6f348c6c41f1ea9e192a2  CentOS-6.9-x86_64-minimal.iso
```
After the secondary node joins the cluster resynchronization happens
```sh
[root@beegfs-cli01 beegfs]# df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda2        30G  1.7G   28G   6% /
devtmpfs        3.4G     0  3.4G   0% /dev
tmpfs           3.5G     0  3.5G   0% /dev/shm
tmpfs           3.5G   17M  3.4G   1% /run
tmpfs           3.5G     0  3.5G   0% /sys/fs/cgroup
/dev/sda1       497M   62M  436M  13% /boot
/dev/sdb1        99G   61M   94G   1% /mnt/resource
tmpfs           697M     0  697M   0% /run/user/1000
beegfs_nodev    8.0T  882M  8.0T   1% /mnt/beegfs 
```
```sh
[root@beegfsmgmt ~]# beegfs-ctl --resyncstats --nodetype=storage --mirrorgroupid=100
Job state: Completed successfully
Job start time: Mon May  1 11:09:27 2017
Job end time: Mon May  1 11:09:40 2017
# of discovered dirs: 2
# of discovered files: 2
# of dir sync candidates: 177
# of file sync candidates: 178
# of synced dirs: 177
# of synced files: 2
# of dir sync errors: 0
# of file sync errors: 0
```
### Metadata node failure test after mirroring

After configuring mirroring and down loading a new file we are testing the metadata mirroring in this step.
#### First we check the node state
```sh
[root@beegfsmgmt ~]# beegfs-ctl --listtargets --nodetype=metadata --state
TargetID     Reachability  Consistency   NodeID
========     ============  ===========   ======
       1           Online         Good        1
       2           Online         Good        2
We also check the status of the mirroring group and process
```
```sh 
[root@beegfsmgmt ~]# beegfs-ctl --listmirrorgroups --nodetype=meta
     BuddyGroupID     PrimaryNodeID   SecondaryNodeID
     ============     =============   ===============
              200                 1                 2
```
```sh
[root@beegfsmgmt ~]# beegfs-ctl --resyncstats --nodetype=meta --nodeid=2
Job state: Completed successfully
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
```
Then we get a file
```sh
[root@beegfs-cli01 beegfs]# wget http://mirrors.vooservers.com/centos/6.9/isos/x86_64/CentOS-6.9-x86_64-minimal.iso
--2017-05-01 12:31:17--  http://mirrors.vooservers.com/centos/6.9/isos/x86_64/CentOS-6.9-x86_64-minimal.iso
Resolving mirrors.vooservers.com (mirrors.vooservers.com)... 194.0.252.34, 2a00:1c10:3:634:0:1000:0:1
Connecting to mirrors.vooservers.com (mirrors.vooservers.com)|194.0.252.34|:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 427819008 (408M) [application/octet-stream]
Saving to: ‘CentOS-6.9-x86_64-minimal.iso’

100%[===============================================================================================>] 427,819,008 11.2MB/s   in 38s

2017-05-01 12:31:54 (10.8 MB/s) - ‘CentOS-6.9-x86_64-minimal.iso’ saved [427819008/427819008]
```
```sh
[root@beegfs-cli01 beegfs]# md5sum CentOS-6.9-x86_64-minimal.iso
af4a1640c0c6f348c6c41f1ea9e192a2  CentOS-6.9-x86_64-minimal.iso
```
After this we stop the primary metadata node.
```sh
[root@beegfsmgmt ~]# beegfs-ctl --listtargets --nodetype=metadata --state
TargetID     Reachability  Consistency   NodeID
========     ============  ===========   ======
       1 Probably-offline         Good        1
       2           Online         Good        2
```
After the node goes down from the storage (takes about 1-2 minutes)
```sh
[root@beegfsmgmt ~]# beegfs-ctl --listtargets --nodetype=metadata --state
TargetID     Reachability  Consistency   NodeID
========     ============  ===========   ======
       1          Offline Needs-resync        1
       2           Online         Good        2
```
The sync process will also be stopped
```sh
[root@beegfsmgmt ~]# beegfs-ctl --resyncstats --nodetype=meta --nodeid=1
Job state: Not started
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
```sh
[root@beegfsmgmt ~]# beegfs-ctl --resyncstats --nodetype=meta --nodeid=2
Job state: Not started
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
Only files and directories which were created after the sync has been introduced will be available, others not!
```sh
[root@beegfs-cli01 beegfs]# ll
ls: cannot access smallfiles: Communication error on send
total 417792
-rw-r--r--. 1 root root 427819008 Mar 28 18:31 CentOS-6.9-x86_64-minimal.iso
d?????????? ? ?    ?            ?            ? smallfiles
```
```sh
[root@beegfs-cli01 beegfs]# cd smallfiles
-bash: cd: smallfiles: Communication error on send
[root@beegfs-cli01 beegfs]# md5sum CentOS-6.9-x86_64-minimal.iso
af4a1640c0c6f348c6c41f1ea9e192a2  CentOS-6.9-x86_64-minimal.iso
```
After downloading a new file, what goes without error
```sh
[root@beegfs-cli01 beegfs]# wget http://mirrors.clouvider.net/CentOS/6.8/isos/x86_64/CentOS-6.8-x86_64-minimal.iso
--2017-05-01 12:44:21--  http://mirrors.clouvider.net/CentOS/6.8/isos/x86_64/CentOS-6.8-x86_64-minimal.iso
Resolving mirrors.clouvider.net (mirrors.clouvider.net)... 185.42.223.15, 2a04:92c1:1:2::6
Connecting to mirrors.clouvider.net (mirrors.clouvider.net)|185.42.223.15|:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 468713472 (447M) [application/octet-stream]
Saving to: ‘CentOS-6.8-x86_64-minimal.iso’

100%[===============================================================================================>] 468,713,472 17.6MB/s   in 16s

2017-05-01 12:44:37 (27.4 MB/s) - ‘CentOS-6.8-x86_64-minimal.iso’ saved [468713472/468713472]
```
We also restart the primary node
After the primary node comes up we will see the resync happening in no time
```sh
[root@beegfsmgmt ~]# beegfs-ctl --resyncstats --nodetype=meta --nodeid=1
Job state: Completed successfully
Job start time: Mon May  1 12:58:19 2017
Job end time: Mon May  1 12:59:04 2017
# of discovered dirs: 2
# of discovery errors: 0
# of synced dirs: 2
# of synced files: 9
# of dir sync errors: 0
# of file sync errors: 0
# of client sessions to sync: 1
# of synced client sessions: 1
session sync error: No
# of modification objects synced: 0
# of modification sync errors: 0
```
The metadata status will be also good
```sh
[root@beegfsmgmt ~]# beegfs-ctl --listtargets --nodetype=metadata --state
TargetID     Reachability  Consistency   NodeID
========     ============  ===========   ======
       1           Online         Good        1
       2           Online         Good        2
```
Chedcking if we bring down the secondary node we still can access the new file
```sh
[root@beegfsmgmt ~]# beegfs-ctl --listtargets --nodetype=metadata --state
TargetID     Reachability  Consistency   NodeID
========     ============  ===========   ======
       1           Online         Good        1
       2 Probably-offline         Good        2
```
```sh
[root@beegfsmgmt ~]# beegfs-ctl --listtargets --nodetype=metadata --state
TargetID     Reachability  Consistency   NodeID
========     ============  ===========   ======
       1           Online         Good        1
       2          Offline         Good        2
```
```sh
[root@beegfs-cli01 beegfs]# ll
total 875520
-rw-r--r--. 1 root root 468713472 May 23  2016 CentOS-6.8-x86_64-minimal.iso
-rw-r--r--. 1 root root 427819008 Mar 28 18:31 CentOS-6.9-x86_64-minimal.iso
drwxr-xr-x. 2 root root         0 May  1 11:43 smallfiles
[root@beegfs-cli01 beegfs]# md5sum CentOS-6.8-x86_64-minimal.iso
0ca12fe5f28c2ceed4f4084b41ff8a0b  CentOS-6.8-x86_64-minimal.iso
[root@beegfs-cli01 beegfs]# md5sum CentOS-6.9-x86_64-minimal.iso
af4a1640c0c6f348c6c41f1ea9e192a2  CentOS-6.9-x86_64-minimal.iso
```
After bringing up the secondary node you will see that it is doing resync operation
```sh
[root@beegfsmgmt ~]# beegfs-ctl --listtargets --nodetype=metadata --state
TargetID     Reachability  Consistency   NodeID
========     ============  ===========   ======
       1           Online         Good        1
       2           Online    resyncing        2
```
```sh
[root@beegfsmgmt ~]# beegfs-ctl --listtargets --nodetype=metadata --state
TargetID     Reachability  Consistency   NodeID
========     ============  ===========   ======
       1           Online         Good        1
       2           Online         Good        2
```

