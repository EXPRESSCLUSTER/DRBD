# DRBD 9.0.28-1
- The purpose of this page is to understand the brilliant replication software DRBD with source code.
  - Kernel module source code: https://github.com/LINBIT/drbd
  - The target verson is [9.0.28-1](https://github.com/LINBIT/drbd/releases/tag/drbd-9.0.28-1).

## Flow
### Initialize
1. module_init
1. [drbd_init](#drbd_init)

### Synchronous I/O
- FIXME

### Asynchronous I/O
- FIXME

## Struct
```c
/* One global retry thread, if we need to push back some bio and have it
 * reinserted through our make request function.
 */
static struct retry_worker {
	struct workqueue_struct *wq;
	struct work_struct worker;

	spinlock_t lock;
	struct list_head writes;
} retry;
```
## Function
### drbd_init
1. register_blkdev
   - Register DRBD block device.
     - Major number is 147.
1. drbd_genl_register
   - Cannot find the source code.
1. [drbd_create_mempools](#drbd_create_mempools)
1. proc_create_single
1. create_singlethread_workqueue
   - Create a work queue.
1. Initialize [do_retry](#do_retry) with INIT_WORK.

### drbd_create_mempools
1. Allocate memory.

### drbd_adm_new_resource
- FIXME: Who calls this function?
1. [drbd_create_resource](#drbd_create_resource)

### drbd_create_resource
1. [drbd_thread_init](#drbd_thread_init)
   - Initialize the the threads.
     - [drbd_worker](#drbd_worker)
     - [drbd_receiver](#drbd_receiver)
     - [drbd_sender](#drbd_sender)
     - [drbd_ack_receiver](#drbd_ack_receiver)

### drbd_thread_init
1. Set functions and resources.

### drbd_thread_start
1. Check thread state.
   - NONE: [drbd_thread_setup](#drbd_thread_setup)

### drbd_thread_setup
1. Call the function initialized by [drbd_resource](#drbd_resource).


### drbd_worker

### drbd_receiver

### drbd_sender

### drbd_ack_receiver

### do_retry
1. [__drbd_make_request](#__drbd_make_request)

### __drbd_make_request
1. inc_ap_bio
1. [drbd_request_prepare](#drbd_request_prepare)
1. [drbd_send_and_submit](#drbd_send_and_submit)

### drbd_request_prepare
1. [drbd_req_new](#drbd_req_new)
1. [req_make_private_bio](#req_make_private_bio)
1. [drbd_queue_write](#drbd_queue_write)

### drbd_req_new
1. mempool_alloc
   - Allocate memory for a request.

### req_make_private_bio
1. bio_clone_fast
   - Clone bio.
1. Set [drbd_request_endio](#drbd_request_endio) to bi_end_io.

### drbd_queue_write
1. queue_work
1. wake_up

### drbd_send_and_submit
1. wake_all_senders
1. drbd_submit_req_private_bio
1. complete_master_bio

### drbd_request_endio
1. [__req_mod](#__req_mod)

### tl_mark_for_resend_by_connection
1. [__req_mod](#__req_mod)

### process_one_request
1. [__req_mod](#__req_mod)

### __req_mod
1. To do the following steps.
   - TO_BE_SENT
     - Send the data to a remote node.
