
# EXPRESSCLUSTER X Data Mirroring setup on Ubuntu 20.04 howto

As of today (Feb 2020) EXPRESSCLUSTER X is impossible to configure its Data Mirroring cluster on Ubuntu 20.04, because its kernel module (Data Mirroring driver) still not released in spite of the user land modules can run on Ubuntu 20.04.
This guide describes how to setup Data Mirroring cluster on Ubuntu 20.04 by EXPRESSCLUSTER X with using *DRBD*.  
May refer to [this site](https://www.nec.com/expresscluster) for details of ECX itself.

---

### Configuration

In the Data Mirroring configuration, the data is replicated among Node1, Node2.
The data replication is performed in the protocol "C" (synchronous replication) of DRBD. the protocol "A" can be chosen when asynchronous replication is required.

```
 [ Node1 ]                   [ Node2 ]
     ^                         ^
     |                         |
     +=========================+
      (Replication: Protocol C)
```

### Software versions

- Ubuntu 20.04.1
- DRBD 9
  - drbd-dkms: 9.0.29-1
  - drbd-utils: 9.17.0-1
- EXPRESSCLUSTER X 4.3.0-1

### Cluster configuration

- NP setting is mandatory
- Group resource
  - exec-drbd: Mounts and unmounts the DRBD device.
- Monitor resources
  - diskw-drbd: Monitors the DRBD device by using the WRITE(FILE) monitoring.
  - genw-drbd: Monitors the status of DRBD.
  - genw-drbd-connection: Monitors the status of the DRBD connection.
  - genw-drbd-service: Monitors the status of the DRBD service.

### Install and configure DRBD

On both nodes which to be the cluster, do as follows.

1. Prepare block device ( e.g. */dev/sdb1* )
1. Install software-properties-common.
    ```sh
    sudo apt-get install software-properties-common
    ```
1. Add LINBIT repository.
    ```sh
    sudo add-apt-repository ppa:linbit/linbit-drbd9-stack
    ```
    - If you have a proxy server, run the following command.
      ```sh
      sudo https_proxy=<your proxy server address> add-apt-repository ppa:linbit/linbit-drbd9-stack
      ```
1. Update the repository.
    ```sh
    sudo apt update
    ```     
1. Install DRBD.
    ```sh
    sudo apt install drbd-utils drbd-dkms
    ```
1. Confirm that the packages was installed.
    ```sh
    sudo dpkg -l |grep drbd
    ii  drbd-dkms                            9.0.29-1ppa1~focal1               all          RAID 1 over TCP/IP for Linux module source
    ii  drbd-utils                           9.17.0-1ppa1~focal1               amd64        RAID 1 over TCP/IP for Linux (user utilities)
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


1. Enable and start the DRBD service.
   ```sh
   # systemctl enable drbd
   # systemctl start drbd
   ```
   if the enable command doesn't work, please refer to [this section](#Workaround-for-systemctl-enable-drbd-error).

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
   # mkdir /mnt/drbd1
   # mount /dev/drbd1 /mnt/drbd1
   # hostname >> /mnt/drbd1/hostnames
   # cat /mnt/drbd1/hostnames
   (Node1)
   # umount /mnt/drbd1
   ```

1. (On Node2) Check if read/write is possible.
   ```sh
   # mkdir /mnt/drbd1
   # mount /dev/drbd1 /mnt/drbd1
   # hostname >> /mnt/drbd1/hostnames
   # cat /mnt/drbd1/hostnames
   (Node1)
   (Node2)
   # umount /mnt/drbd1
   ```
   **Note** : Expecting *auto-promote* option is enabled by default. DRBD resource is automatically promoted to primary on mount and demoted to secondary on umount.

### Install and configure EXPRESSCLUSTER

1. Install EXPRESSCLUSTER and register licenses.
1. Configure the NP resolution resource.
   - This is mandatory setting.
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
     - Monitor Timing: Active (Target resource: exec-drbd)
   - Monitor(special)
     - Method: WRITE(FILE)
     - Monitor Target: /mnt/drbd1/diskw
   - Recovery Action
     - Recovery Action: Restart the recovery target, and if there is no effect with restart, then failover
     - Recovery Target: failover-drbd
1. Configure the custom monitor resource "genw-drbd".
   - Monitor(common)
     - Monitor Timing: Active (Target resource: exec-drbd)
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

       df /mnt/drbd1 | grep /mnt/drbd1
       mstate=$?
       echo "mount status: $mstate"
       exit $mstate
       ```
   - Recovery Action
     - Recovery Action: Restart the recovery target, and if there is no effect with restart, then failover
     - Recovery Target: failover-drbd
1. Configure the custom monitor resource "genw-drbd-connection".
   - Monitor(common)
     - Monitor Timing: Active (Target resource: exec-drbd)
   - Monitor(special)
     - Monitor script:
       ```sh
       #! /bin/sh
       #***********************************************
       #*                   genw.sh                   *
       #***********************************************

       drbdadm cstate ecx_md1 | grep Connected
       if [ $? -ne 0 ]; then
           echo "Not connected"
           exit 1
       fi
       ```
   - Recovery Action
     - Recovery Action: Execute only the final action
     - Recovery Target: LocalServer
     - Final Action: No operation
1. Configure the custom monitor resource "genw-drbd-service".
   - Monitor(common)
     - Monitor Timing: Always
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
   - Recovery Action
     - Recovery action: Custom setting
     - Recovery Target: LocalServer
     - Recovery Script Execution Count: 1
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
     - Final Action: Stop the cluster service and reboot OS
1. Apply the cluster configuration.

### Maintenance information

In case that "systemctl enable drbd" is showing error status, drbd mirroring is being interrupted for any reason.
Depending on the result of "drbdadm cstate (drbd resource name)", take the following actions.

- Status: **Connnecting**
   - The server is unable to communicate with another server.
   - After communication is restoerd, differential copy will run automatically.
- Status: **StandAlone** or **WFConnection**
   - Regardless of the success or failure of the communication with another server, there is inconsistency in the data between both servers.
   - You need to specify manually which one has the most recent data.
   1. Move a failover group to see which one has the most recent data.
   1. Keep a failover group running on the server that has the most recent data.
   1. Execute following commands on ECX **secondary** server.
      ```sh
      # drbdadm disconnect <drbd resource name>
      # drbdadm secondary <drbd resource name>
      # drbdadm connect --discard-my-data <drbd resource name>
      ```
   1. Execute following commands on ECX **secondary** server.
      ```sh
      # drbdadm disconnect <drbd resource name>
      # drbdadm connect <drbd resource name>
      ```
   1. Full copy is executed from primary to secondary.z

### [Workaround for "systemctl enable drbd" error](#Workaround-for-systemctl-enable-drbd-error)

Depending DRBD version, you may face this error after "systemctl enable drbd".
  ```sh
  root@ubuntu2004-1:~# uname -rv
  5.8.0-1035-aws #37~20.04.1-Ubuntu SMP Tue Jun 1 09:54:15 UTC 2021
  root@ubuntu2004-1:~# cat /sys/module/drbd/version
  9.0.29-1
  root@ubuntu2004-1:~# systemctl enable drbd
  Synchronizing state of drbd.service with SysV service script with /lib/systemd/systemd-sysv-install.
  Executing: /lib/systemd/systemd-sysv-install enable drbd
  update-rc.d: error: drbd Default-Start contains no runlevels, aborting.
  root@ubuntu2004-1:~# systemctl is-enabled drbd
  disabled
  ```

In that case, please follow the workaround below.
1. Edit /etc/init.d/drbd
   - Before:
      ```sh
      # Default-Start:
      ```
   - After:
      ```sh
      # Default-Start:  2 3 4 5
      ```
1. Execute following commands
    ```sh
    # cd /etc/rc2.d
    # ln -s ../init.d/drbd S01drbd
    # cd ../rc3.d
    # ln -s ../init.d/drbd S01drbd
    # cd ../rc4.d
    # ln -s ../init.d/drbd S01drbd
    # cd ../rc5.d/
    # ln -s ../init.d/drbd S01drbd
    # update-rc.d drbd enable 2 3 4 5
    ```
1. Execute "systemctl enable drbd"

### Reference

- The DRBD9 User's Guide: https://docs.linbit.com/docs/users-guide-9.0/