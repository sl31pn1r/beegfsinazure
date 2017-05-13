# beegfsinazure
BeeGFS parallel file system in Azure

Introduction
=========
BeeGFS is a parallel filesystem solution for HPC cluster workloads running on premise or in cloud environments.

With Storage-, Metadata servers, Buddy Mirroring and a management service solution it provides vast ranges of configuration options, to accomodate almost any design requirement, with easily scaleout options.

This article is a How-To article with templates to deploy a High-Available BeeGFS cluster in Azure. 
The core components are basic Azure resources with properties easy to modify. The article also contains some test results.

The solution was implemented in the Azure West Europe region, on standard services, like Virtual Machines, Storage accounts.
The high level design looks like the following: https://ibb.co/nRg55k

Within a single region, I have deployed a Single resource Group with a VNET, within that VNET with a Subnet. If required the storage servers can be separeted to another subnet from the clients. In a production environment I would advise to have them separated.
I have deployed a single management server, an Availability Set with two metadata- and another Availability set two storage nodes, along with two clients. In an production HPC cluster solution I aso would recomment to put the clients to an Availability set, for HA purposes.

All the nodes were Standard D2_v2 with all of its limitations on IOPS, network throughput etc.
The operating system was CentOS 7.3 for the storage environment.
For the start a single metadata node with two 1 TB disks was deployed, later I have introduced a second metadata node with the same setup.
There were two storage nodes with 4 standard 1TB disk / node.
I have used 2 client nodes for test CentOS 6.8 and CentOS 7.3

Installation & Configuration
========
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

MGMT server
--------
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


MetaData server
--------
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


Storage server
--------
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

Client machine
--------
It is possible to use NFS to mount BeeGFS shares, however in that case we completley loose the parallel file system benefits (like file coherency in less than a second). It is advised to use the BeeGFS client to mount the shares.

    wget -O /etc/yum.repos.d/beegfs-rhel7.repo http://www.beegfs.com/release/latest-stable/dists/beegfs-rhel7.repo
    yum install -y beegfs-client beegfs-helperd beegfs-utils 
    sed -i 's/^sysMgmtdHost.*/sysMgmtdHost = 'beegfsmgmt'/g' /etc/beegfs/beegfs-client.conf
    echo "/mnt/beegfs /etc/beegfs/beegfs-client.conf" > /etc/beegfs/beegfs-mounts.conf
    mkdir -p /mnt/beegfs

    systemctl daemon-reload
    systemctl enable beegfs-helperd.service
    systemctl enable beegfs-client.service

Management commands
========

To manage the services use sysctl commands like

    systemctl start/stop/status beegfs-mgmtd
    systemctl start/stop/status beegfs-meta
    systemctl start/stop/status beegfs-storage
    systemctl start/stop/status beegfs-helperd
    systemctl start/stop/status beegfs-client
To manage BeeGFS from the management node use the following commands

**List nodes**
```sh
[root@beegfsmgmt ~]# beegfs-ctl --listnodes --nodetype=storage
   beegfs-storage02 [ID: 1]
   beegfs-storage01 [ID: 2]
```        
```sh
[root@beegfsmgmt ~]# beegfs-ctl --listnodes --nodetype=meta
        beegfs-meta01 [ID: 1]
```

**Listing metadata nodes after adding a new metadata server**
```sh
[root@beegfsmgmt ~]# beegfs-ctl --listtargets --nodetype=meta --state
TargetID     Reachability  Consistency   NodeID
========     ============  ===========   ======
       1           Online         Good        1
       2           Online         Good        2
```

**List storage target**
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
