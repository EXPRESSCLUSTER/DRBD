
# EXPRESSCLUSTER X Data Mirroring setup on CentOS 8 howto

As of today (Feb 2020) EXPRESSCLUSTER X is impossible to configure its Data Mirroring cluster on CentOS 8, because its kernel module (Data Mirroring driver) still not released in spite of the user land modules can run on CentOS 8.
This guide describes how to setup Data Mirroring cluster on CentOS 8 by EXPRESSCLUSTER X with using *DRBD*.  
May refer to [this site](https://www.nec.com/expresscluster) for  details of ECX itself.

---

### Configuration

In the Data Mirroring configuration, the data is replicated among Node1,
Node2.
The data replication is performed in the protocol "C" (synchronous replication) of DRBD. the protocol "A" can be chosen when asynchronous replication is required.

```
 [ Node1 ]                   [ Node2 ]
     ^                         ^
     |                         |
     +=========================+
      (Replication: Protocol C)
```

### Software versions

- CentOS 8.1 ( DRBD 9.0 )
- EXPRESSCLUSTER X 4.1.1-1

### Cluster configuration

- Group resource
  - exec-drbd: Mounts and unmounts the DRBD device.
- Monitor resources
  - diskw-drbd: Monitors the DRBD device by using the WRITE(FILE) monitoring.
  - genw-drbd: Monitors the status of DRBD.
  - genw-drbd-service: Monitors the status of the DRBD service.

### Install and configure DRBD

On both nodes which to be the cluster, do as follows.

1. Prepare block device ( e.g. */dev/sdb1* )

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
   # vi /etc/drbd.d/ecx_md1.res
   ```
   The following is an example configuration.  
   Replace (Node1) and (Node2) with the hostname of each node.  
   Replace (Address1) and (Address2) with the IP address of each node

   ```
   resource ecx_md1 {
        device /dev/drbd1;
        disk /dev/sdb1;
        meta-disk internal;

        on (Node1) {
                node-id 0;
                address (Address1):7700;
        }
        on (Node2) {
                node-id 1;
                address (Address2):7700;
        }
        connection {
                net {
                        protocol C;
                }
                host (Node1) port 7710;
                host (Node2) port 7711;
        }
   }
   ```

1. Initialize the metadata storage and enable the resource. 
   ```sh
   # drbdadm create-md ecx_md1
   # drbdadm up ecx_md1
   ```


1. Enable the DRBD service.
   ```sh
   # systemctl enable drbd
   ```

### Verify DRBD operation

1. (On Node1) Initialize the DRBD resource.
   ```sh
   # drbdadm primary --force ecx_md1
   ```

1. (On Node1) Create a file system.
   ```sh
   # mkfs.xfs -f /dev/drbd1
   ```
1. (On Node1) Check if read/write is possible.
   ```sh
   # mount /dev/drbd1 /mnt/drbd1
   # hostname >> /mnt/drbd1/hostnames
   # cat /mnt/drbd1/hostnames
   (Node1)
   # umount /mnt/drbd1
   ```

1. (On Node2) Check if read/write is possible.
   ```sh
   # mount /dev/drbd1 /mnt/drbd1
   # hostname >> /mnt/drbd1/hostnames
   # cat /mnt/drbd1/hostnames
   (Node1)
   (Node2)
   # umount /mnt/drbd1
   ```
   **Note** : Expecting *auto-promote* option is enabled by default. DRBD resource is automatically promoted to primary on mount and demoted to secondary on umount.

### Install and configure EXPRESSCLUSTER

1. Install EXPRESSCLUSTER and register lincenses.
1. Configure the failover group "failover-drbd".
1. Configure the exec resource "exec-drbd".
   - Details
     - Start script:
       ```sh
       #! /bin/sh
       #***************************************
       #*              start.sh               *
       #***************************************

       echo "exec-drbd: start"
       mount /dev/drbd1 /mnt/drbd1
       result=$?
       echo "exit status: $result"
       exit $result
       ```
     - Stop script:
       ```sh
       #! /bin/sh
       #***************************************
       #*               stop.sh               *
       #***************************************

       echo "exec-drbd: stop"
       umount /mnt/drbd1
       result=$?
       echo "exit status: $result"
       exit $result
       ```
1. Configure the disk monitor resource "diskw-drbd".
   - Monitor(common)
     - Monitor timing: Active (Target resource: exec-drbd)
   - Monitor(special)
     - Method: WRITE(FILE)
     - Monitor target: /mnt/drbd1/diskw
   - Recovery action
     - Recovery action: Restart the recovery target, and if there is no effect with restart, the failover
     - Recovery target: failover-drbd
1. Configure the custom monitor resource "genw-drbd".
   - Monitor(common)
     - Monitor timing: Active (Target resource: exec-drbd)
   - Monitor(special)
     - Monitor script:
       ```sh
       #! /bin/sh
       #***********************************************
       #*                   genw.sh                   *
       #***********************************************

       echo "genw-drbd: monitor"

       role=`drbdadm role ecx_md1`
       echo "role: $role"
       if [ $role != "Primary" ]; then
           exit 1
       fi

       drbdadm cstate ecx_md1 | grep Connected
       if [ $? -ne 0 ]; then
           echo "Not connected"
           exit 1
       fi

       df /mnt/drbd1 | grep /mnt/drbd1
       mstate=$?
       echo "mount status: $mstate"
       exit $mstate
       ```
   - Recovery action
     - Recovery action: Restart the recovery target, and if there is no effect with restart, the failover
     - Recovery target: failover-drbd
1. Configure the custom monitor resource "genw-drbd-service".
   - Monitor(common)
     - Monitor timing: Always
   - Monitor(special)
     - Monitor script:
       ```sh
       #! /bin/sh
       #***********************************************
       #*                   genw.sh                   *
       #***********************************************

       echo "genw-drbd-service: monitor"
       systemctl status drbd
       result=$?
       echo "exit status: $result"
       exit $result
       ```
   - Recovery action
     - Recovery action: Custom setting
     - Recovery target: LocalServer
     - Recovery script execution count: 1
     - Recovery script:
       ```sh
       #! /bin/sh
       #***********************************************
       #*	          preaction.sh                 *
       #***********************************************

       systemctl restart drbd
       result=$?
       echo "exit status: $result"
       exit $result
       ```
     - Final action: Stop cluster service and reboot OS

### Reference

- The DRBD9 User's Guide: https://docs.linbit.com/docs/users-guide-9.0/