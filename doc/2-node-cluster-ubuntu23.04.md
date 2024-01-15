
# EXPRESSCLUSTER X Data Mirroring setup on Ubuntu 23.04 howto

As of today (Dec 2023) EXPRESSCLUSTER X is impossible to configure its Data Mirroring cluster on Ubuntu 23.04, because its kernel module (Data Mirroring driver) still not released in spite of the user land modules can run on Ubuntu 23.04.
This guide describes how to setup Data Mirroring cluster on Ubuntu 23.04 by EXPRESSCLUSTER X with *DRBD*.  

---

## Configuration

In the Data Mirroring configuration, the data is replicated among Node1, Node2.
The data replication is performed in the protocol "C" (synchronous replication) of DRBD. the protocol "A" can be chosen when asynchronous replication is required.

```txt
 [ Node1 ]                   [ Node2 ]
     ^                         ^
     |                         |
     +=========================+
      (Replication: Protocol C)
```

## Software versions

- Ubuntu 23.04.3
  - drbd: 8.4.11
  - drbd-utils: 9.15.0-1build2
- EXPRESSCLUSTER X 5.1.2-1

## Cluster configuration

- NP setting is mandatory required
- Group resource
  - exec-drbd: Mounts and unmounts the DRBD device.
- Monitor resources
  - diskw-drbd: Monitors the DRBD device by using the WRITE(FILE) monitoring.
  - genw-drbd: Monitors the status of DRBD.
  - genw-drbd-connection: Monitors the status of the DRBD connection.
  - genw-drbd-service: Monitors the status of the DRBD service.

## Install and configure DRBD

On both nodes which to be the cluster, do as follows.

1. Prepare block device ( e.g. */dev/sdb1* )
1. Install software-properties-common.

    ```sh
    sudo apt-get install drbd-utils
    ```

1. Create the DRBD configuration file.

    ```sh
    sudo vi /etc/drbd.conf
    ```

   The two hosts in this exmplle will be called *node1* and *node2*

    ```conf
    global { usage-count no; }
    common { syncer { rate 1G; } }
    resource ecx_md1 {
      protocol C;
      startup {
        wfc-timeout 15;
        degr-wfc-timeout 60;
      }
      net {
        cram-hmac-alg sha1;
        shared-secret "secret";
      }
      on node1 {
        device /dev/drbd1;
        disk /dev/sdb1;
        address 192.168.0.1:7700;
        meta-disk internal;
      }
      on node2 {
        device /dev/drbd1;
        disk /dev/sdb1;
        address 192.168.0.2:7700;
        meta-disk internal;
      }
    }
    ```

1. Initialize the metadata storage and enable the resource.

   ```sh
   sudo drbdadm create-md ecx_md1
   ```

1. Enable and start the DRBD service.

   ```sh
   sudo systemctl enable drbd
   sudo systemctl start drbd
   ```

## Verify DRBD operation

1. (On Node1) Initialize the DRBD resource.

   ```sh
   sudo drbdadm -- --overwrite-data-of-peer primary all
   ```

1. (On Node1) Create a file system.

   ```sh
   sudo mkfs.ext4 /dev/drbd1
   ```

1. (On Node1) Check if read/write is possible.

   ```sh
   # mkdir /mnt/drbd1
   # mount /dev/drbd1 /mnt/drbd1
   # hostname >> /mnt/drbd1/hostnames
   # cat /mnt/drbd1/hostnames
   node1
   # umount /mnt/drbd1
   ```

1. (On Node2) Check if read/write is possible.

   ```sh
   # mkdir /mnt/drbd1
   # mount /dev/drbd1 /mnt/drbd1
   # hostname >> /mnt/drbd1/hostnames
   # cat /mnt/drbd1/hostnames
   node1
   node2
   # umount /mnt/drbd1
   ```

## Install and configure EXPRESSCLUSTER

1. Install EXPRESSCLUSTER and register licenses.
1. Configure the NP resolution resource.
1. Configure the failover group "failover-drbd".
1. Configure the exec resource "exec-drbd".

     - Start script:

       ```sh
       #! /bin/sh
       #***************************************
       #*              start.sh               *
       #***************************************

       echo "exec-drbd: start"
       drbdadm primary ecx_md1
       result=$?
       echo "exit status: $result"
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
       drbdadm secondary ecx_md1
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
       result=`systemctl status drbd`
       status=`echo "$result" | grep Active: | awk '{print $2}'`
       if [ $status != "active" -a $status != "activating" ]; then
           echo "exit status: $result"
           exit $result
       fi
       exit 0
       ```

   - Recovery Action
     - Recovery action: Custom setting
     - Recovery Target: LocalServer
     - Recovery Script Execution Count: 1
     - Recovery script:

       ```sh
       #! /bin/sh
       #***********************************************
       #*           preaction.sh                      *
       #***********************************************

       systemctl restart drbd
       result=$?
       echo "exit status: $result"
       exit $result
       ```

     - Final Action: Stop the cluster service and reboot OS
1. Apply the cluster configuration.

## Maintenance

In case that "systemctl enable drbd" is showing error status, drbd mirroring is being interrupted for any reason.
Depending on the result of "drbdadm cstate (drbd resource name)", take the following actions.

- Status: **Connnecting**
  - The server is unable to communicate with another server.
  - After communication is restoerd, differential copy will run automatically.
- Status: **StandAlone** or **WFConnection**
  - Regardless of the success or failure of the communication with another server, there is inconsistency in the data between both servers.
  - You need to specify manually which server has the most recent data.
  1. Move the failover group to see which one has the most recent data.
  1. Make the failover group running on the server that has the most recent data. Take the node as the active server.
  1. Execute following commands on **standby** server.

      ```sh
      # drbdadm disconnect <drbd resource name>
      # drbdadm secondary <drbd resource name>
      # drbdadm connect --discard-my-data <drbd resource name>
      ```

  1. Execute following commands on **active** server.

      ```sh
      # drbdadm disconnect <drbd resource name>
      # drbdadm connect <drbd resource name>
      ```

  1. Full copy is executed from active to standby server.

## How to recover from a split-brain situation

On the node that will discard its changes (the one that will be sacrificed), run the following commands:

```bash
drbdadm disconnect [resource]
drbdadm secondary [resource]
drbdadm -- --discard-my-data connect [resource]
```

These commands will discard the data changes on that node and start synchronization with the other node.

On the surviving node (the one that will keep its data), execute the following command to start synchronization:

drbdadm connect [resource]

These commands are used after DRBD detects a split-brain to manually intervene and discard changes on one node while resynchronizing with the other. When a split-brain occurs, DRBD immediately disconnects, and one node will always be in a ‘StandAlone’ state. Recovery is then done manually using the commands mentioned above.

## Reference

- Ubuntu HA - DRBD: <https://discourse.ubuntu.com/t/ubuntu-ha-drbd/11314>
