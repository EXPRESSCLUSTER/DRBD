
# 3 nodes cascading replication by Stacking

This document describes how to configure the cascading storage replication with DRBD 9.0.

## Configuration overview

In the configuration of this guide, the data is replicated among Node1, Node2 and Node3. Node1, 2 are located in Site1. Node3 is located in Site2 which is far from Site1.
The data replication in Site1 (i.e. between Node1 and Node2) is performed in **synchronous** (protocol C). The cross-site data replication is performed in **asynchronous** (protocol A).

```
============[ Site1 ]=============         ==[ Site2 ]==

+-- Node1 -------+   +-- Node2 --+         +-- Node3 --+
|                |   |           |         |           |
|  Application   |   |           |         |           |
|  writes        |   |           |         |           |
|  (file system) |   |           |         |           |
|    |           |   |           |         |           |
|    v           |   |           |         |           |
|  drbd10 ---------C---> drbd10  |         |           |  Upper layer (md)
|    |           |   |     |     |         |           |
|    |           |   |     v     |         |           |
|    |           |   |   drbd0 -------A------> drbd0   |  Lower layer (mdL)
|    |           |   |     |     |         |     |     |
|    v           |   |     v     |         |     v     |
|  Storage       |   |   Storage |         |   Storage |
+----------------+   +-----------+         +-----------+
```

On Node1, a write operation goes to the block device *drbd10* then the storage. *drbd10* also sends it to *drbd10 on Node2*.  
On Node2, the write operation goes from *drbd10* to *drbd0* then the storage. *drbd0* also sends it to *drbd0 on Node3*.  
On Node3, the write operation goes from *drbd0* to the storage.  
Thus, the write operation on Node1 is replicated for all nodes.

All the write operation on Node1 wait for the completion of the send operation on Node 2, and it causes **performance degrade** of the operation.
If the communication between Node2 and Node3 is **suspended**, the write operation which is replicated from Node1 to Node2 are recorded as *DIRTY WRITE for Node3* on the bitmap in *drbd0* of Node2 .
The remaining *DIRTY WRITE*s are replicated to Node3 on **resume** of the communication. **Node2 can be thought as write cache of Node3.**  
*cron* is utilized to controll connect and disconnect of the communication and enables periodic resyncing.

## Software versions

- DRBD 9.0 in CentOS 8.1
- EXPRESSCLUSTER X 4.1.1

## Setup procedure

#### Install and configure DRBD

On all nodes, do as follows.

1. Prepare block device ( */dev/sdb1* )

1. Install DRBD.
   ```sh
   # yum -y install elrepo-release
   # yum -y install drbd90-utils kmod-drbd90
   ```

1. Load the kernel module.
   ```sh
   # insmod $(rpm -ql kmod-drbd90 | grep drbd.ko)
   ```

1. Create the DRBD resource file.
   ```sh
   # vi /etc/drbd.d/md.res
   ```

   ```
   # Lower layer replication
   resource mdL {
           protocol A;
           device /dev/drbd0;
           disk /dev/sdb1;
           meta-disk internal;
   
           on sv12 {
                   address 192.168.137.12:7700;
           }
           on sv13 {
                   address 192.168.137.13:7700;
           }
   }
   
   # Upper layer replication stacked on mdL
   resource md {
           protocol C;
           net {
                   verify-alg md5;
           }
           stacked-on-top-of md {
                   device /dev/drbd10;
                   address 192.168.137.12:7711;
           }
           on sv11 {
                   device /dev/drbd10;
                   disk    /dev/sdb1;
                   address 192.168.137.11:7711;
                   meta-disk internal;
           }
   }
   ```

   following is the parameters on this example.

   |      | host name | IP address    |
   |--    |--         |--             |
   |Node1 | sv11      | 192.168.137.11|
   |Node2 | sv12      | 192.168.137.12|
   |Node3 | sv13      | 192.168.137.13|

1. Initialize the metadata storage.

   At first Node2 and 3
   ```sh
   # drbdadm create-md mdL
   ```

   Then Node2
   ```sh
   # drbdadm create-md --stacked md
   ```

   Then Node1
   ```sh
   # drbdadm create-md md
   ```

<!--
1. Enable and start the DRBD service.
   ```sh
   # systemctl enable drbd
   # systemctl start drbd
   ```
-->

#### Initial replication

1. On Node2 and 3,
   ```sh
   # drbdadm up mdL
   ```

1. On Node2, Initialize the DRBD resource *md*.
   ```sh
   # drbdadm primary --force mdL
   ```
   Confirm the completion of the initialization.
   ```sh
   # drbdadm status
   ```

1. On Node2,
   ```sh
   # drbdadm up --stacked mds
   ```

1. On Node1,
   ```sh
   # drbdadm up mds
   # drbdadm primary --force mds
   ```
   Confirm the completion of the initialization.
   ```sh
   # drbdadm status
   ```

1. On Node1, create a file system.
   ```sh
   # mkfs.ext4 /dev/drbd10
   ```

1. On Node3, suspend the replication between Node2 and 3.
   ```sh
   # drbdadm disconnect mdL
   ```
   ---
   **NOTE**: Resuming the replication requires that *mdL* on Node2 is **Primary** and **UpToDate** status. Issue the following command on Node2 to confirm it.
   ```sh
   # drbdadm status mdL
   mdL role:Primary
     disk:UpToDate
     sv13 connection:Connecting
   ```

   Then issue the following command on Node3 for resuming.
   ```sh
   # drbdadm connect mdL
   ```
   ---

## Failover and Failback operations

<!--
sv11	umount /mnt
sv12	drbdadm down --stacked mds
sv13	mount /dev/drbd1 /mnt/
	hostname >> /mnt/hostnames
	umount /mnt
sv12	drbdadm up --stacked mds
	drbdadm primary --stacked mds

sv11	drbdadm discon > con > verify	NG!
	drbdadm verify > discon > con 	NG!
	reboot > drbdadm up > (verify)	OK!!!
	mount /dev/drvd10 /mnt
-->

#### Failover from Node1 to Node3
1. On Node1, check if read/write is possible.
   ```sh
   # mount /dev/drbd10 /mnt
   :
   # # Emulating business operation
   # hostname >> /mnt/hostnames
   # cat /mnt/hostnames
   sv11
   :
   # umount /mnt
   ```
1. On Node2, stop the stacked device.
   ```sh
   # drbdadm down --stacked md
   ```

1. On node3, check if read/write is possible consystently.
   ```sh
   # mount /dev/drbd1 /mnt
   # cat /mnt/hostnames
   sv11
   # hostname >> /mnt/hostnames
   # cat /mnt/hostnames
   sv11
   sv13
   ```

#### Failback from Node3 to Node1

1. On node3, unmount the device.
   ```
   # umount /mnt
   ```

1. On Node2, start the stacked device.
   ```
   # drbdadm up --stacked md
   # drbdadm primary --stacked md
   ```

1. On Node1, recover consistency.
   ```
   # reboot
   ```
   After completion of the reboot, start the device (resource md).
   Then the differences between Node1 and Node2 are synchronized.
   ```
   # drbdadm up md
   ```

1. On Node2, unmount the device (resoruce md).
   ```
   # umount /mnt
   ```

1. On Node1, mount the device (resource md).
   ```
   # mount /dev/drbd10 /mnt
   ```

1. Optionally, verify the consistency.
   ```
   # drbdadm verify md
   # tail -f /var/log/messages
   ```

---
**NOTE**: The downtime required for failback cannot be guaranteed to be sufficiently short, as the length of time required to transfer the difference from Node2 to Node1 is indefinite.

---

## Reference

- The DRBD9 User's Guide: https://docs.linbit.com/docs/users-guide-9.0/
