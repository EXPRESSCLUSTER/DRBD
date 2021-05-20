# Create 3-node Cluster with Ubuntu

## Index
- [Software](#software)
- [Overview](#overview)
- [Prerequisite](#prerequisite)
- [Installing DRBD](#installing-drbd)
- [Configuring DRBD](#configuring-drbd)
- [Verifying Replication](#verifying-replication)

## Software
- Ubuntu 20.04.2 LTS (5.4.0-72-generic)
- DRBD 9
  - drbd-dkms: 9.0.29-1
  - drbd-utils: 9.17.0-1
- EXPRESSCLUSTER X 4.3 (4.3.0-1)

## Overview
- Create 1:2 replication cluster with DRBD and EXPRESSCLUSTER X.
- DRBD provides three types of protocol. On this configuration, Protocol C (synchronous mirroring) is used between Node1 and Node2. Protocol A (Asynchronous mirroring) is used between site1 and site2. Regarding to protocal, refer to the following documentation.
  - https://linbit.com/drbd-user-guide/drbd-guide-9_0-en/#s-replication-protocols
- When Node1 is the active server, Node1 replicates the data to Node2 with synchronous mode and replicates the same data to Node3 with asynchronous mode.
  ```
  Replication Protocol
  A: Asynchronous
  C: Synchronous

  +--- Site1 -----+
  | +-------+     |
  | | Node1 +---------+
  | |       +---+ |   |
  | +-------+   | |   |
  |             C |   |
  | +-------+   | |   |
  | | Node2 <---+ |   |
  | |       |     |   |
  | +-------+     |   |
  +---------------+   |
                      A
  +--- Site2 -----+   |
  | +-------+     |   |
  | | Node3 <---------+
  | |       |     |
  | +-------+     |
  +---------------+
  ```
- When Node2 is the active server, Node2 replicates the data to Node1 with synchronous mode and replicates the same data to Node3 with asynchronous mode.
  ```
  Replication Protocol
  A: Asynchronous
  C: Synchronous

  +--- Site1 -----+
  | +-------+     |
  | | Node1 |     |
  | |       <---+ |
  | +-------+   | |
  |             C |
  | +-------+   | |
  | | Node2 +---+ |
  | |       +---------+
  | +-------+     |   |
  +---------------+   |
                      A
  +--- Site2 -----+   |
  | +-------+     |   |
  | | Node3 <---------+
  | |       |     |
  | +-------+     |
  +---------------+
  ```
## Prerequisite
- Do the following steps on **each node**.
  1. Add a block device for replication (e.g. /dev/vda).
  1. Create a partition (e.g. /dev/vda1).
     - Don't create a file system.

## Installing DRBD
- Do the following steps on **each node**.
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
  1. 
     ```sh
     sudo dpkg -l |grep drbd
     ii  drbd-dkms                            9.0.29-1ppa1~focal1               all          RAID 1 over TCP/IP for Linux module source
     ii  drbd-utils                           9.17.0-1ppa1~focal1               amd64        RAID 1 over TCP/IP for Linux (user utilities)
     ```
## Configuring DRBD
- Do the following steps on **each node**.
  1. Move to /etc/drbd.d and create [resource name].res file (e.g. r0.res) as below.
     ```
     resource r0 {
         device /dev/drbd0;
         disk /dev/vda1;
         meta-disk internal;
     
         on node1 {
            node-id 0;
            address 192.168.122.1:7700;
         }
         on node2 {
            node-id 1;
            address 192.168.122.2:7700;
         }
         on node3 {
            node-id 2;
            address 192.168.122.3:7700;
         }
         connection {
             net {
                 protocol C;
             }
             host node1 port 7710;
             host node2 port 7711;
         }
         connection {
             net {
                 protocol A;
             }
             host node1 port 7720;
             host node3 port 7722;
         }
         connection {
             net {
                 protocol A;
             }
             host node2 port 7731;
             host node3 port 7732;
         }
     }
     ```
  1. Initialize DRBD's metadata.
     ```sh
     sudo drbdadm create-md r0
     ```
  1. Enable DRBD service.
     ```sh
     sudo systemctl enable drbd
     sudo systemctl start drbd
     ```
## Verifying Replication
1. On node1, do the following steps.
   ```sh
   sudo drbdadm primary --force r0
   sudo mkfs -t ext4 /dev/drbd0
   sudo mkdir /mnt/drbd0
   sudo mount /dev/drbd0 /mnt/drbd0
   sudo hostname >> /mnt/drbd0/hostnames
   sudo cat /mnt/drbd0/hostnames
   node1
   sudo umount /mnt/drbd0
   sudo drbdadm secondary r0
   ```
1. On node2, do the following steps.
   ```sh
   sudo mkdir /mnt/drbd0
   sudo mount /dev/drbd0 /mnt/drbd0
   sudo hostname >> /mnt/drbd0/hostnamesrun the
   sudo cat /mnt/drbd0/hostnames
   sudo drbdadm status r0
   node1
   node2
   sudo umount /mnt/drbd0
   ```
1. On node3, do the following steps.
   ```sh
   sudo mkdir /mnt/drbd0
   sudo mount /dev/drbd0 /mnt/drbd0
   sudo hostname >> /mnt/drbd0/hostnames
   sudo cat /mnt/drbd0/hostnames
   node1
   node2
   node3
   sudo umount /mnt/drbd0
   ```
1. On node1, run the following command to set primary node for node1.
   ```sh
   $ sudo drbdadm status r0
   r0 role:Primary
     disk:UpToDate
     node2 role:Secondary
       peer-disk:UpToDate
     node3 role:Secondary
       peer-disk:UpToDate   
   ```
## Installing EXPRESSCLUSTER
1. Install EXPRESSCLUSTER.
1. Register licenses.
1. Reboot all servers.

## Creating a Cluster
- FIXME

## Link
- LINBIT
  - https://linbit.com
- User's Guide
  - https://linbit.com/drbd-user-guide/drbd-guide-9_0-en/
  - Installing
    - https://linbit.com/drbd-user-guide/drbd-guide-9_0-en/#s-install-pkgs-ubuntu_linux
  - Configuring
    - https://linbit.com/drbd-user-guide/drbd-guide-9_0-en/#ch-configure