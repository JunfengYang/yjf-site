---
title: "Postgres Dynamic Shared Memory(dsm)"
date: 2019-06-20T16:35:16+08:00
categories:
- postgres
- greenplum
tags:
- pg
- gp
- postgres
- greenplum
keywords:
- pg
- dsm
- dynamic
- shared
- memory
#thumbnailImage: //example.com/image.jpg
---

# Dynamic Shared Memory And Shared Memory Queue

These days I just read the postgres' dynamic shared memory and shared message queue.
This is my study note of the postgres source code.
The note covers "dsm.c", "shm_toc.c" and "shm_mq.c".

<!--more-->

## Structure
```c
static dsm_handle dsm_control_handle;
```
The dsm_control_handle is an unit32 value.
Postgres use it to create a global dynamic shared memory file/segment. And create share memory on it.

```

                      (global dsm space/segment)
dsm_control_handle --> ------------------------------------
                      |HEADER|item|item|item|item|item|item|--> dsm_control_item
                       ---------|----|----|----|----|----|-
                      ^         |    ...  ...  ...  |    ...
                      |         |                   v (dsm space)
dsm_control_header *dsm_control |                    -----
                   -------------|                   | ... |
                   |                                 -----
                   v (dsm space)
                    --------------------------------------------------------------------
                   |toc| toc_entry | toc_entry | ... | free space | ... | CHUNK | CHUNK |
                    ---------|-----------|----------------------------------------------
                             |           |                              ^       ^   |
                             |           |------------------------------|       |   |
                             |--------------------------------------------------|   |
toc means table of content, it seperate a space to table of chunks.                 |
                                                                                    |
Shared memory message queue is residence dynamic shared memory.                     |
For example we put a queue in a toc chunk.                                          |
                                                   queue                            v
                                                    --------------------------------
                                                   |HEADER|      data ring          |
                                         shm_mq ->  --------------------------------
```

## Dynamic Shared Memory
```c
/* Shared-memory state for a dynamic shared memory segment. */
typedef struct dsm_control_item
{
  dsm_handle  handle;
  uint32    refcnt;     /* 2+ = active, 1 = moribund, 0 = gone */
  void     *impl_private_pm_handle; /* only needed on Windows */
  bool    pinned;
} dsm_control_item;

/* Layout of the dynamic shared memory control segment. */
typedef struct dsm_control_header
{
  uint32    magic;
  uint32    nitems;
  uint32    maxitems;
  dsm_control_item item[FLEXIBLE_ARRAY_MEMBER];
} dsm_control_header;

/* Backend-local state for a dynamic shared memory segment. */
struct dsm_segment
{
  dlist_node  node;     /* List link in dsm_segment_list. */
  ResourceOwner resowner;   /* Resource owner. */
  dsm_handle  handle;     /* Segment name. */
  uint32    control_slot; /* Slot in control segment. */
  void     *impl_private; /* Implementation-specific private data. */
  void     *mapped_address; /* Mapping address, or NULL if unmapped. */
  Size    mapped_size;  /* Size of our mapping. */
  slist_head  on_detach;    /* On-detach callbacks. */
};
```

### dsm startup.
Since postmaster don't use dsm, the startup of dsm will execute when a backend get started.
For the whole database cluster(Not greenplum cluster).
The shared memory control segment/space get created, which is also a dsm.
The unique random ```handle``` value that identify the lower layer shared memory file/segment
will be generated. Then, the file get created. After init ```dsm_control_header *dsm_control```,
the startup is done.

### Create new dsm.
First it creates a local ```dsm_segment``` as segment descriptor.
Add it to local ```dsm_segment_list``` to track current created ot attached dsm segments
for current process.
Generate random ```handle``` that identify the segment. Then create dsm base on it.
If dsm gets created, record it into shared memory control segment's free ```dsm_control_item - dsm_control->item[free_slot]```.
Set the item's ```refcnt``` to 2. Increase the ```dsm_control->nitems```.
Record the current ```free_slot``` into local segment descriptor ```seg->control_slot```.

### Attach to a dsm
Use the ```handle``` that identify the segment, first iterate the local ```dsm_segment_list```
to check whether current proc already created or attached to that dsm segment.
Error out if exists in the list.
If not exists in the list,  creates a local ```dsm_segment``` as segment descriptor. 
Add it to local ```dsm_segment_list```. 
Iterate dsm items from shared memory control segment ```dsm_control->item```.
Find the required ```handle``` and increment item's refcnt. 
Set ```seg->control_slot``` to current item index.
Finally, attach to the shared memory segment.

### Resize an existing dsm
If mapped size is equal to new size, do nothing.
If mapped size > new size, shrink to new size.
If mapped size < new size, fill up to new size.
Not all OS support resize. And resize will lose data for last two case.

After resize, other process attached to the dsm need remap on existing shared memory segment.

### Detach from dsm
If there are detach callbacks registeded on local segment descriptor ```seg->on_detach```,
call the callback before detach.
Then execute detach on dsm. Update shared memory control segment item attributes like ```refcnt```.
If current ```refcnt``` is 1, destory the dsm. Then clean the shared memory control segment item.

### Register dsm detach callback.
Once successfully created or attached to a dsm, we can register detach callback in local
segment descriptor. ```seg->on_detach```. It's a list chain every detach callback.

## Shared Memory Segment Table of Contents
Provide a simple way to divide a chunk of shared
memory (probably dynamic shared memory allocated via dsm_create) into
a number of regions and keep track of the addresses of those regions or
key data structures within those regions.
```c
typedef struct shm_toc_entry
{
  uint64    key;      /* Arbitrary identifier */
  Size    offset;     /* Offset, in bytes, from TOC start */
} shm_toc_entry;

struct shm_toc
{
  uint64    toc_magic;    /* Magic number identifying this TOC */
  slock_t   toc_mutex;    /* Spinlock for mutual exclusion */
  Size    toc_total_bytes;  /* Bytes managed by this TOC */
  Size    toc_allocated_bytes;  /* Bytes allocated of those managed */
  uint32    toc_nentry;   /* Number of entries in TOC */
  shm_toc_entry toc_entry[FLEXIBLE_ARRAY_MEMBER];
};
```

### Crearte shared memory table of content
Init ```shm_toc``` structure in region of dsm. Set a magic number in the structure for attach validation.

### Attach shared memory table of content
Transfer the address region of dsm into ```shm_toc``` structure. 
Validate the expected magic number is equal to the one in ```shm_toc``` structure.

### Allocate space from table of content
Check whether remain free space is enough for the required size.
If yes, allocate space from the end of ```shm_toc``` structure.

### Insert table of content entry into shm_toc to record allocate space info.
Fill up the ```shm_toc->toc_entry[free_slot]``` with the allocated space affress offset and key.

### Lookup for a chunk from table of content
Iterate all ```shm_toc->toc_entry``` and get entry that match request key. Return the chunk address.

## Shared Memory Message Queue
Single-reader, single-writer shared memory message queue.

```c
// This structure represents the actual queue, stored in shared memory.
struct shm_mq
{
  slock_t   mq_mutex;
  PGPROC     *mq_receiver;
  PGPROC     *mq_sender;
  uint64    mq_bytes_read;
  uint64    mq_bytes_written;
  Size    mq_ring_size;
  bool    mq_detached;
  uint8   mq_ring_offset;
  char    mq_ring[FLEXIBLE_ARRAY_MEMBER];
};

// This structure is a backend-private(local) handle for access to a queue.
struct shm_mq_handle
{
  shm_mq     *mqh_queue;
  dsm_segment *mqh_segment;
  BackgroundWorkerHandle *mqh_handle;
  char     *mqh_buffer;
  Size    mqh_buflen;
  Size    mqh_consume_pending;
  Size    mqh_partial_bytes;
  Size    mqh_expected_bytes;
  bool    mqh_length_word_complete;
  bool    mqh_counterparty_attached;
  MemoryContext mqh_context;
};
```

### Create share memory message queue
Init ```shm_mq``` structure base on the given space. Set the ring size.

### Set share memory message queue receiver
Set the receiver proc for the queue. If the sender exists, notify the sender. 

### Set share memory message queue sender
Set the reader proc for the queue. if the receiver exists, notify the receiver.

### Attach to a share memory message queue
Attach is actually init a local ```shm_mq_handle``` structure record some local 
attribute for the shared queue.
Also register a detach callback which update ```shm_mq->mq_detached``` to true.
To let other process attach to the queue know that this proc detached.

Set ```shm_mq_handle->mqh_handle``` if the other process attach to this queue is a bgworker.
Because we don't want to wait forever if the bgworker fails to start. 
So we use the ```BackgroundWorkerHandle mqh_handle``` to track bgworker status.

### Send data to queue.
For data sending, it'll seperate into two phase.

1. write length into the buffer first.
2. write the data which length is equal the 1st step length.

For each write, calculate the freespace of the queue's ring.

- If there's no availabe space to write.
    - If the receiver not exists, wait for a receiver. Or return ```SHM_MQ_WOULD_BLOCK``` for no block write.
    - If the receiver detached, return ```SHM_MQ_DETACHED```, no wait.
    - If receiver exists, notify receiver to read more data from the queue.
        - If notify not success, return ```SHM_MQ_DETACHED```, means receiver detached. 
        - Remember it may write partial data here.
        - If success, wait, so receiver can read out data. Once receiver read some data, it'll wake up current sender. to continue write.
- If there still enough space for write, write to the ring, wrap if needed cause the queue is a ring.
```shm_mq->mq_bytes_written``` is used to track current write bytes.

### Read data from queue
For data reading, it read one msg at a time. It also seperate into two phase.

1. read length of the msg.
2. read the msg data which length is from 1st read.

- At the begining, it check whether sender exists. If not exists, it can not read.
So, wait the sender process. For no wait, it'll return SHM_MQ_WOULD_BLOCK or 
SHM_MQ_DETACHED if sender detached.
- If data get wraped in the ring, local process use local ```shm_mq_handle->mqh_buffer``` 
to assemble final msg.
- ```shm_mq->mq_bytes_read``` is used to track current read bytes.
- If there's no more data to read, wait for sender's notify so that the queue have data to read.


## More resources.
- [pgsql: Introduce dynamic shared memory areas.](https://www.postgresql.org/message-id/E1cCrkl-0000JT-16%40gemulon.postgresql.org)
- [shared memory message queues](https://www.postgresql.org/message-id/CA+TgmobUe28JR3zRUDH7s0jkCcdxsw6dP4sLw57x9NnMf01wgg@mail.gmail.com)