
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

## Install and configure DRBD

On both nodes, do as follows.

1. Prepare block device ( e.g. `/dev/sdb1` ) and mountpoint (`/mnt/drbd1`).
1. Install required package.

    ```sh
    sudo apt-get install drbd-utils
    ```

1. Configure DRBD configuration file `/etc/drbd.conf`.
   The two hosts in this example were named *node1* and *node2*

   NOTE: This configuration includes automatic recovery settings after Split Brain called `after-sb-[0-2]pri` and uses `discard-zero-changes`, `discard-secondary` policy.
   See man page of drbd.conf(5) for detail.

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
        after-sb-0pri discard-zero-changes;
        after-sb-1pri discard-secondary;
        after-sb-2pri disconnect;
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

1. Initialize DRBD metadata storage and enable the service.

   ```sh
   # Initialize the metadata storage and enable the resource.
   sudo drbdadm create-md ecx_md1

   # Enable and start the DRBD service.
   sudo systemctl enable drbd
   sudo systemctl start drbd
   ```

1. On Node1 initialize the DRBD resource and create a file system on it.

   ```sh
   # Initialize the DRBD resource.
   sudo drbdadm -- --overwrite-data-of-peer primary all

   # Create a file system.
   sudo mkfs.ext4 /dev/drbd1

   # Demote it secondary
   sudo drbdadm secondary ecx_md1
   ```

## Install and configure EXPRESSCLUSTER

1. Install EXPRESSCLUSTER and register licenses.
1. Configure the NP resolution resource.
1. Configure the failover group `failover-1`.
1. Configure the exec resource `exec-drbd`.

     - Start script:

       ```sh
       #! /bin/sh
       #***************************************
       #*              start.sh               *
       #***************************************

       echo "[D] drbdadm primary ecx_md1"
       drbdadm primary ecx_md1
       result=$?
       echo "exit status: $result"
       if [ $result != 0 ]; then
           exit $result
       fi
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

       echo "[D] umount /mnt/drbd1"
       umount /mnt/drbd1
       result=$?
       echo "exit status: $result"
       if [ $result != 0 ]; then
           exit $result
       fi
       echo "[D] drbdadm secondary ecx_md1"
       drbdadm secondary ecx_md1
       result=$?
       echo "exit status: $result"
       exit $result
       ```

1. Configure a custom monitor resource `genw-drbd`.

   - Monitor(common)
     - Monitor Timing: Active (Target resource: exec-drbd)
   - Monitor(special)
     - Log Output Path: `/opt/nec/clusterpro/log/genw-drbd.log`
     - Rotate Log: check
     - Monitor script:

       ```sh
       #! /bin/sh
       #***********************************************
       #*                   genw.sh                   *
       #***********************************************
       res=ecx_md1
       drbdadm role $res | grep '^Primary'
       if [ $? -ne 0 ]; then
           echo "[E] [$res] is Not Primary"
           exit 1
       fi
       ```

   - Recovery Action
     - Recovery Action: Execute only the final action
     - Recovery Target: LocalServer
     - Final Action: Stop the cluster service and reboot OS

1. Configure the disk monitor resource "diskw-drbd".
   - Monitor(common)
     - Monitor Timing: Active (Target resource: exec-drbd)
   - Monitor(special)
     - Method: WRITE(FILE)
     - Monitor Target: /mnt/drbd1/diskw
   - Recovery Action
     - Recovery Action: Restart the recovery target, and if there is no effect with restart, then failover
     - Recovery Target: failover-drbd

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

```bash
drbdadm connect [resource]
```

These commands are used after DRBD detects a split-brain to manually intervene and discard changes on one node while resynchronizing with the other. When a split-brain occurs, DRBD immediately disconnects, and one node will always be in a ‘StandAlone’ state. Recovery is then done manually using the commands mentioned above.

## Reference

- Ubuntu HA - DRBD: <https://discourse.ubuntu.com/t/ubuntu-ha-drbd/11314>
