# DRBD 9.0.29-1
- The purpose of this page is to understand the brilliant replication software DRBD with source code.
  - Kernel module source code: https://github.com/LINBIT/drbd
  - The target verson is [9.0.29-1](https://github.com/LINBIT/drbd/releases/tag/drbd-9.0.29-1).
  - Source Code: https://linbit.com/linbit-software-download-page-for-linstor-and-drbd-linux-driver/#drbd

## Drivers
### drbd.ko
#### Initialize
1. module_init
1. [drbd_init](#drbd_init)

### drbd_transport_tcp.ko

## Struct
```
static const struct block_device_operations drbd_ops = {
	.owner		= THIS_MODULE,
	.submit_bio	= drbd_submit_bio,
	.open		= drbd_open,
	.release	= drbd_release,
};
```
- [drbd_submit_bio](#drbd_submit_bio)
- [drbd_open](#drbd_open)

## Function
### drbd_init
1. initialize_kref_debugging
1. register_blkdev
1. drbd_genl_register
   - We cannot find the function...
1. drbd_create_mempools
   - Allocate memory for the driver.
1. create_singlethread_workqueue
1. Initialize [do_retry](#do_retry) with INIT_WORK.

### do_retry
1. list_for_each_entry_safe
1. __drbd_make_request

### drbd_submit_bio
1. __drbd_make_request
