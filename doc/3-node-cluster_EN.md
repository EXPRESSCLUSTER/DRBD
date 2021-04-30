3-node DRBD cluster
===

About this guide
---

This guide describes how to create a distributed storage cluster with DRBD 9.0.
For the detailed information of EXPRESSCLUSTER X, please refer to [this site](https://www.nec.com/en/global/prod/expresscluster/index.html).

Configuration
---

### Overview

In the configuration of this guide, the user data is replicated among Node1,
Node2 (the both nodes are located in Site1) and Node3 (that is located in Site2 which is far from Site1).
The data replication in Site1 (i.e. between Node1 and Node2) is performed in
the protocol C. The cross-site data replication is performed in the protocol A.

```
||==========[ Site1 ]==========||                        ||==[ Site2 ]==||
||                             ||                        ||             ||
|| [ Node1 ]         [ Node2 ] ||                        ||  [ Node3 ]  ||
     ^   ^             ^   ^                                   ^   ^
     |   |             |   |                                   |   |
     |   +=============+   +-----------------------------------+   |
     |   (Replication:           (Replication: Protocol A)         |
     |       Protocol C)                                           |
     |                                                             |
     +-------------------------------------------------------------+
                                 (Replication: Protocol A)
```

### Software versions

- Redhat Enterprise Linux 7.5
- EXPRESSCLUSTER X 3.3.5 / 4.0.0
- DRBD 9.0

### Cluster configuration

- Group resource
  - exec-drbd: Mounts and unmounts the DRBD device.
- Monitor resources
  - diskw-drbd: Monitors the DRBD device by using the WRITE(FILE) monitoring.
  - genw-drbd: Monitors the status of DRBD.
  - genw-drbd-service: Monitors the status of the DRBD service.

Install and configure DRBD
---

Operate the following procedures on each node of the cluster.

1. Add the RPM repository of DRBD.
   ```sh
   # rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
   # rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm

   # yum info *drbd* | grep Name
   Name        : drbd84-utils
   Name        : drbd84-utils-sysvinit
   Name        : drbd90-utils
   Name        : drbd90-utils-sysvinit
   Name        : kmod-drbd84
   Name        : kmod-drbd90
   ```
1. Install DRBD.
   ```sh
   # yum install drbd90-utils kmod-drbd90
   ```
1. Load the kernel module.
   ```sh
   # insmod $(rpm -ql kmod-drbd90 | grep drbd.ko)
   ```
1. Configure the DRBD resource.
   ```sh
   # vi /etc/drbd.d/test.res
   ```
   The following is an example configuration.
   Please replace (Node1), (Node2) or (Node3) with the hostname of each node.
   Also, Please replace (Address1), (Address2) and (Address3) with the IP
   address of each node.
   ```
   resource test {
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
        on (Node3) {
                node-id 2;
                address (Address3):7700;
        }
        connection {
                net {
                        protocol C;
                }
                host (Node1) port 7710;
                host (Node2) port 7711;
        }
        connection {
                net {
                        protocol A;
                }
                host (Node1) port 7720;
                host (Node3) port 7722;
        }
        connection {
                net {
                        protocol A;
                }
                host (Node2) port 7731;
                host (Node3) port 7732;
        }
   }
   ```
1. Initialize the metadata storage.
   ```sh
   # drbdadm create-md test
   ```
1. Enable and start the DRBD service.
   ```sh
   # systemctl enable drbd
   # systemctl start drbd
   ```

Verify DRBD operation
---

1. (On Node1) Initialize the DRBD resource.
   ```sh
   # drbdadm primary --force test
   ```
1. (On Node1) Create a file system.
   ```sh
   # mkfs.xfs -f /dev/drbd1
   ```
1. (On Node1) Check if read/write is available.
   ```sh
   # mount /dev/drbd1 /mnt/drbd1
   # hostname >> /mnt/drbd1/hostnames
   # cat /mnt/drbd1/hostnames
   (Node1)
   # umount /mnt/drbd1
   # drbdadm secondary test
   ```
1. (On Node2) Check if read/write is available.
   ```sh
   # mount /dev/drbd1 /mnt/drbd1
   # hostname >> /mnt/drbd1/hostnames
   # cat /mnt/drbd1/hostnames
   (Node1)
   (Node2)
   # umount /mnt/drbd1
   ```
   (Note: if auto-promote option is enabled, the DRBD resource will be
   automatically promoted to primary when you mount the DRBD device.
   Also, the resource will be automatically demoted to secondary when you
   unmount the device.)
1. (On Node3) Check if read/write is available.
   ```sh
   # mount /dev/drbd1 /mnt/drbd1
   # hostname >> /mnt/drbd1/hostnames
   # cat /mnt/drbd1/hostnames
   (Node1)
   (Node2)
   (Node3)
   # umount /mnt/drbd1
   ```

Install and configure EXPRESSCLUSTER
---

1. Install EXPRESSCLUSTER and register lincenses.
1. Configure the server groups.
   - Site1: Node1, Node2
   - Site2: Node3
1. Configure the failover group "failover-drbd".
   - Basic settings
     - User server group settings: enabled
   - Server groups that can run the group
     - 1: Site1
     - 2: Site2
   - Attribute
     - Auto failover
       - Prioritize failover policy in the server group
         - Enable only manual failover among the server groups: enabled
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
       drbdadm secondary test   # just for confirmation
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

       role=`drbdadm role test`
       echo "role: $role"
       if [ $role != "Primary" ]; then
           exit 1
       fi

       drbdadm cstate test | grep Connected
       cstate=$?
       if [ $cstate -ne 0 ]; then
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

Reference
---

- The DRBD9 User's Guide: https://docs.linbit.com/docs/users-guide-9.0/
