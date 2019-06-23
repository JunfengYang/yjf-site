---
title: "Postgres Virtual File Descriptor(vfd)"
date: 2019-06-23T16:56:21+08:00
categories:
- postgres
- storage mamagement
- file
tags:
- postgres
- storage
- vfd
keywords:
- postgres
- virtual
- file
- descriptor
- storage
#thumbnailImage: //example.com/image.jpg
---

I'm reading the Postgres storage code recently. I'll pust some note onto this site.

This note is about the VFD(virtual file descriptor). 

Let's first look at the Postgres database physic structure. Where the relations(table, index, ...) files get stored on disk. [Reference: interdb - Chapther 1](http://www.interdb.jp/pg/pgsql01.html)
<!--more-->

## Relation(table, schema, index, view ...) path and name. common/relpath.c
Under the data directory, a relation's physical storage consists of one or more forks(names for the same relation).
For example a table oid "13059" under database "13213" contains files "13059"(main fork), "13059_fsm"(FSM fork), "13059_vm"(visibility fork). There may also contain an initialization fork.
```c
/*
 * Stuff for fork names.
 *
 * The physical storage of relation consists of one or more forks.
 * The main fork is always created, but in addition to that there can be
 * additional forks for storing various metadata. ForkNumber is used when
 * we need to refer to a specific fork in a relation.
 */
typedef enum ForkNumber
{
    InvalidForkNumber = -1,
    MAIN_FORKNUM = 0,
    FSM_FORKNUM,
    VISIBILITYMAP_FORKNUM,
    INIT_FORKNUM

    /*
     * NOTE: if you add a new fork, change MAX_FORKNUM and possibly
     * FORKNAMECHARS below, and update the forkNames array in
     * src/common/relpath.c
     */
} ForkNumber;
const char *const forkNames[] = {
    "main",                        /* MAIN_FORKNUM */
    "fsm",                        /* FSM_FORKNUM */
    "vm",                        /* VISIBILITYMAP_FORKNUM */
    "init"                        /* INIT_FORKNUM */
};
```

##### char * GetDatabasePath(Oid dbNode, Oid spcNode): construct path to a database directory

- If the ```spcNode == GLOBALTABLESPACE_OID```, return "global".
- If ``` spcNode == DEFAULTTABLESPACE_OID```, return "bash/(dbNode/dboid)".

All other tablespaces are accessed via symlinks. pg_tblspc/(spcNode/spcoid)/PG_(version)/(dbNode/dboid).

##### char * GetRelationPath(Oid dbNode, Oid spcNode, Oid relNode, int backendId, ForkNumber forkNumber): construct path to a relation's file

- If the ```spcNode == GLOBALTABLESPACE_OID```:
    - if ```forkNumber != MAIN_FORKNUM```:
        - return "global/reloid_forkname"
    - else return "global/reloid"
- If ``` spcNode == DEFAULTTABLESPACE_OID```
    - `return "base/dboid/[tbackendid_]reloid[\_forkname(if not ```MAIN_FORKNUM```)]"`
- else
    - `return "pg_tblspc/spcoid/PG_(verison)/dboid/[tbackendid_]reloid[\_forkname(if not ```MAIN_FORKNUM```)]"`


## VFD: storage/file/fd.c

### Reason for virtual file descriptors.
This code manages a cache of 'virtual' file descriptors (VFDs).
The server opens many file descriptors for a variety of reasons,
including base tables, scratch files (e.g., sort and hash spool
files), and random calls to C library routines like system(3); it
is quite easy to exceed system limits on the number of open files
a single process can have.  (This is around 256 on many modern
operating systems, but can be as low as 32 on others.)

### Key interfaces.
- ```PathNameOpenFile``` - used to open virtual files
    PathNameOpenFile is intended for files that are held open for a long time, like relation files. 
- ```OpenTemporaryFile``` - used to open virtual files
    automatically deleted when the File is closed, either explicitly or implicitly at end of transaction or process exit.

It is the caller's responsibility to close them, there is no automatic mechanism in fd.c for that.

- ```AllocateFile```, ```AllocateDir```, ```OpenPipeStream``` and ```OpenTransientFile``` are wrappers around ```fopen(3)```, ```opendir(3)```, ```popen(3)``` and ```open(2)```, respectively. They behave like the corresponding native functions, except that the handle is registered with the current subtransaction, and will be *automatically closed at abort*. These are intended mainly for *short operations* like reading a configuration file; there is a limit on the number of files that can be opened using these functions at any one time.

- ```BasicOpenFile``` is just a thin wrapper around open() that can release file descriptors in use by the virtual file descriptors if necessary.

- ```OpenTransientFile/CloseTransient``` File for an unbuffered file descriptor

- VFD structure

```c
typedef struct vfd
{
    int            fd;                /* current FD, or VFD_CLOSED if none */
    unsigned short fdstate;        /* bitflags for VFD's state */
    ResourceOwner resowner;        /* owner, for automatic cleanup */
    File        nextFree;        /* link to next free VFD, if in freelist */
    File        lruMoreRecently;    /* doubly linked recency-of-use list */
    File        lruLessRecently;
    off_t        seekPos;        /* current logical file position, or -1 */
    off_t        fileSize;        /* current size of file (0 if not temporary) */
    char       *fileName;        /* name of file, or NULL for unused VFD */
    /* NB: fileName is malloc'd, and must be free'd when closing the VFD */
    int            fileFlags;        /* open(2) flags for (re)opening the file */
    int            fileMode;        /* mode to pass to open(2) */
} Vfd;

/*
 * Virtual File Descriptor array pointer and size.  This grows as
 * needed.  'File' values are indexes into this array.
 * Note that VfdCache[0] is not a usable VFD, just a list header.
 */
static Vfd *VfdCache;
static Size SizeVfdCache = 0;

/*
 * Number of file descriptors known to be in use by VFD entries.
 */
static int    nfile = 0;

typedef int File; /* It indicate the VFD index in VfdCache. */
```

### Key Private Routines for VFD implementation.
The ```VfdCache``` is an array of VDF. In the cache, there's an LRU ring and a free list.
The Least Recently Used ring is a doubly linked list(constructs by ```vfd.lruMoreRecently``` and ```vfd.lruLessRecently```) that begins and ends on element zero. Element zero is special -- it doesn't represent a file and its "fd" field always == VFD_CLOSED. Element zero is just an anchor that shows us the beginning/end of the ring. Only VFD elements that are currently really open (have an FD assigned) are in the LRU ring. Elements that are "virtually" open can be recognized by having a non-null fileName field.
Free list is a list contains free VDF, constructs by the ```vfd.nextFree```.
##### ```InitFileAccess``` initialize this module during backend startup
For initialization, only init one VFD and set to VfdCache[0], 
it's head of the VFD cache for both LRU and free list. Set ```VfdCache->fd = VFD_CLOSED```.
Not ```SizeVfdCache = 1```
Also call
```c
/* register proc-exit hook to ensure temp files are dropped at exit */
    on_proc_exit(AtProcExit_Files, 0);
```
to register proc-exit hook to ensure temp files are dropped at exit.(on_proc_exit is a IPC module)
Function ```AtProcExit_Files``` is the callback to clean temple files.
It a wrapper of ```CleanupTempFiles(true)```, it closes temporary files and deletes their underlying files.

##### ```AllocateVfd``` grab a free (or new) file record (from VfdCache).
After ```InitFileAccess```, the ```SizeVfdCache``` must bigger than 0 now.
If the free list is empty ```VfdCache[0].nextFree == 0```.
    - Double ```SizeVfdCache```, if it's first time to ```AllocateVfd```. Set ```SizeVfdCache=32```. 
    - realloc the ```VfdCache``` array to siese ```SizeVfdCache```. Note the first element is header.
    - Initialize the new entries in ```VfdCache``` and link them into the free list.
Get ```File file``` from free list head(VfdCache[0].nextFree). The File is actually the id of the VDF in ```VfdCache``` array.
Remove the File from free list. ```VfdCache[0].nextFree = VfdCache[file].nextFree```.
Return the allocated vfd file.

##### ```FreeVfd``` free a file record.
The input is the VFD File identitor. We can get the actual VFD by it(```Vfd *vfdP = &VfdCache[file]```).
We should clean the VFD and add it to head of free list.
```c
vfdP->nextFree = VfdCache[0].nextFree;
VfdCache[0].nextFree = file;
```

##### ````Insert```` put a file at the front of the LRU ring.
The input is the VFD File identitor. We can get the actual VFD by it(```Vfd *vfdP = &VfdCache[file]```).
And we insert current file VFD into LRU as the most recently used item.
We know the LRU is a ring through ```lruLessRecently``` and ```lruMoreRecently``` which is also ```File``` type indicates the index in ```VfdCache```.
If the Vfd item's ```VfdCache[file].lruMoreRecently = 0``` and ```VfdCache[0].lruLessRecently == file```. means the ```VfdCache[file]``` is the most recentlly used VFD.
```c
vfdP->lruMoreRecently = 0;
vfdP->lruLessRecently = VfdCache[0].lruLessRecently;
VfdCache[0].lruLessRecently = file;
VfdCache[vfdP->lruLessRecently].lruMoreRecently = file;
```

##### ```Delete``` delete a file from the Lru ring.
The input is the VFD File identitor. We can get the actual VFD by it(```Vfd *vfdP = &VfdCache[file]```).
Remove the file VFD from LRU ring.
```c
VfdCache[vfdP->lruLessRecently].lruMoreRecently = vfdP->lruMoreRecently;
VfdCache[vfdP->lruMoreRecently].lruLessRecently = vfdP->lruLessRecently;
```
Alyhrough the VFD get remove from LRU, but we may nore free it because we may reuse it later.

##### ```LruDelete``` remove a file from the LRU ring and close its FD.
We can get the actual VFD by it(```Vfd *vfdP = &VfdCache[file]```).
Make sure the seek position is valid. ```FilePosIsUnknown(vfdP->seekPos)```
Then call ```close(vfdP->fd)``` to close the real FD. ```--nfile``` reduce real used FD count.
And then all ```Delete``` to remove vfd from LRU.

##### ```ReleaseLruFile``` Release an fd by closing the last entry in the Lru ring.
if real used FD count > 0.
    Get the less recently used VFD(```VfdCache[0].lruMoreRecently```) and call ```LruDelete```.
    return true.
else return false.

##### ```ReleaseLruFiles``` Release fd(s) until we're under the max_safe_fds(max fs allowed in the pg) limit
If opened fs >= max_safe_fds, run ```ReleaseLruFile()```.

##### ```LruInsert``` put a file at the front of the LRU ring and open it.
We can get the actual VFD by it(```Vfd *vfdP = &VfdCache[file]```).
If the file not opened yet, run ```ReleaseLruFiles``` to make sure opened fd count is safe.
Then execute ```BasicOpenFile``` to open it. BasicOpenFile, same as open(2) except can free other FDs by ```ReleaseLruFile``` if needed.
if open success, ```++nfile```, else return -1.
Then check to seek position is valid, if not ```--nfile``` and return -1.
If everything is ok, call ```Insert(file)``` to insert it into LRU head.

#### More private routines for VFD implemenation.
##### ```FileAccess``` open the file(File) if not opened, add it at the frount of LRU ring.
If the file not opened, call ```LruInsert```.
If the file opened, and the VDF is not head of LRU ring, call ```Delete(file); Insert(file);```


#### VDF public interface
##### Main operations
- PathNameOpenFile(FileName fileName, int fileFlags, int fileMode): open a file in an arbitrary directory by file name.
    1. It'll can ```AllocateVfd``` to retrieve a ```File file```, get it from ```vfdP = &VfdCache[file]```.
    2. Call ```ReleaseLruFiles``` to make sure current fd number is safe.
    3. Call ```BasicOpenFile(fileName, fileFlags, fileMode)``` to open a real fd.
    4. Call ```Insert``` to add it to head of LRU ring.
    5. Set VFD arributes.

- OpenTemporaryFile(bool interXact): Open a temporary file that will disappear when we close it.
    It actually record the ```File file``` in current resource owner normally, it'll get deleted if not used anymore(transaction end). If we want the temp file to outlive the current transaction, we don't record it.
    In either case, the file is removed when the File is explicitly closed.

- FileClose(File file): close a file when done with it.
    1. Get VFD ```vfdP = &VfdCache[file]```, if the file opened, close it by ```close(cfdP->fd)```, ```--nfile``` and mark it closed. Then call ```Delete(file)``` to remove file from LRU.
    2. If the file is temporary ```FD_TEMPORARY```, delete it by ```unlink(vdfP->fukeName)```
    3. If it has resource owner, remove it from it's owner.
    4. call ```FreeVfd(file)``` to return vfd to free list.

- FilePrefetch?

- FileRead(File file, char \*buffer, int amount, uint32 wait_event_info)
    1. Call ```FileAccess``` to a file to make the file as the head of LRU ring.
    2. Call ```read(vfdP->fd, buffer, amount)``` to read data into buffer. if nor success, it'll retry.

- FileWrite(File file, char \*buffer, int amount, uint32 wait_event_info)
    1. Call ```FileAccess``` to a file to make the file as the head of LRU ring.
    2. If it's a temporary file, make sure the size does not exceed ```temp_file_limit```
    3. Call ```write(vfdP->fd, buffer, amount)``` to write buffer.
    4. Calculate ```temporary_files_size``` for temporary file.
    5. if not succeed, may retry.
 
- FileSync(File file, uint32 wait_event_info): synchronize a file's in-core state with that on disk

- FileSeek(File file, off_t offset, int whence): seek read offset.

- FileTruncate(File file, off_t offset, uint32 wait_event_info): truncate file size.
    1. ```FileAccess(file)```
    2. call ```ftruncate``` and update file size in vfd.

- FileWriteback: advise OS that the described dirty data should be flushed
- FilePathName: get the file path name by return ```VfdCache[file].fileName```

### FD operations Implementations
They behave like the corresponding OS native functions, except that the handle is registered with the current subtransaction, and will be *automatically closed at abort*.
```c
/* To specify the file type allocated by these wrapper fnctions */
typedef enum
{
    AllocateDescFile,
    AllocateDescPipe,
    AllocateDescDir,
    AllocateDescRawFD
} AllocateDescKind;

/* Each file allocated by these wrapper function use AllocateDesc to describe itself. */
typedef struct
{
    AllocateDescKind kind;
    SubTransactionId create_subid;
    union
    {
        FILE       *file;
        DIR           *dir;
        int            fd;
    }            desc;
} AllocateDesc;

static int    numAllocatedDescs = 0;    /* Number of fds opened by these wraper functions */
static int    maxAllocatedDescs = 0;  /* Max fds than can be opened by these wraper functions */
static AllocateDesc *allocatedDescs = NULL; /* Array of the fd' AllocateDescs opened by these wraper functions */
```

##### reserveAllocatedDesc(void): Make room for another allocatedDescs[] array entry if needed and possible.
If ```numAllocatedDescs < maxAllocatedDescs```, still have room for new ```AllocateDescs```, return.
If ```allocatedDescs``` is null, set ```maxAllocatedDescs = FD_MINFREE / 2``` and malloc this size of ```AllocateDesc```.
if ```allocatedDescs != NULL``` and no room for new ```AllocateDescs```, set ```maxAllocatedDescs = max_safe_fds / 2``` and realloc allocatedDescs to new ```maxAllocatedDescs``` size.

##### AtEOSubXact_Files(bool isCommit, SubTransactionId mySubid, SubTransactionId parentSubid): Take care of subtransaction commit/abort.
For each element in ```allocatedDescs```, if is ```create_subid == mySubid```, for commit, we reassign the files that were opened to the parent subtransaction.  For abort, we close temp files that the subtransaction may have opened.

#### FD operations interface
##### Operations that allow use of regular stdio --- USE WITH CAUTION
- AllocateFile: Routines that want to use stdio (ie, FILE\*) should use AllocateFile rather than plain fopen()
    1. ```reserveAllocatedDesc``` to make sure ```AllocateDescs``` still have room for new opened file.
    2. ```ReleaseLruFiles``` to close excess fds.
    3. ```fopen``` to open the file and if success, set ```AllocateDesc``` attributes for current ```allocatedDescs[numAllocatedDescs]```.

```c
desc->kind = AllocateDescFile;
desc->desc.file = file;
desc->create_subid = GetCurrentSubTransactionId();
```

- FreeFile: Close a file returned by AllocateFile.

##### Operations that allow use of pipe streams (popen/pclose)
- OpenPipeStream: Routines that want to initiate a pipe stream should use OpenPipeStream rather than plain popen().
    Same routine with ```AllocateFile``` but use ```popen```.

- ClosePipeStream: Close a pipe stream returned by OpenPipeStream

##### Operations to allow use of the <dirent.h> library routines
- AllocateDir: Routines that want to use <dirent.h> (ie, DIR\*) should use AllocateDir 
rather than plain ```opendir()```.
- ReadDir
- ReadDirExtended
- FreeDir: Close a directory opened with AllocateDir.

##### Operations to allow use of a plain kernel FD, with automatic cleanup
- OpenTransientFile: Like AllocateFile, but returns an unbuffered fd like ```open(2)```
- CloseTransientFile: Close a file returned by OpenTransientFile.