# About the Portworx Graph Driver for Docker
PX-Graph is a graph driver for Docker and designed to provide a purpose-built container image management file system. It is specifically designed to address the following issues:

1. Efficient page cache usage: Current graph drivers that depend on device mapper abuse the page cache by loading multiple copies of the same image layers in memory. This takes away host memory from running applications.
2. Efficient i-node usage: Current graph drivers exhaust the number of inodes available, thereby causing the underlying filesystems to run out of space.
3. Efficient cloning: Current graph drivers such as overlay implement a copy-on-write approach, which consumes CPU and takes time during container image management operations
4. Efficient and correct garbage collection and space management: Current graph drivers routinely end up with orphaned layers and cause the operator to resort to resetting Docker (usually by deleting `/var/lib/docker`).

PX-Graph is built on a new filesystem designed by Portworx, specifically for managing Linux container images. This filesystem is called LCFS, which stands for `Layer Cloning File System`.

This file system is provided to Docker by way of the FUSE low level API. 

# Installing the Portworx Graph driver for Docker
PX-Graph is available as a v2 plugin and requires Docker version 1.13 or higher. It is available on the public Docker hub and is installed using `docker plugin install portworx/px-graph`.

> Currently the v2 interface is not generally available. There is an [oustanding issue with the v2 interface](https://github.com/docker/docker/issues/28948). Therefore, Portworx provides an alternate way of installing the PX-Graph plugin.  
> Follow [these instructions](https://github.com/portworx/px-graph/tree/master/INSTALL.md) to install Px-Graph


# Overview

Portworx Graph Driver is a custom-built file system to meet the needs of a saving, starting and managing Linux container images.  In the context of Docker, it is similar to AUFS and Overlay graph drivers in terms of its use cases.  Unlike AUFS and Overlay, this new graph driver is a native file system, which means it does not operate on top of another file system, but operates directly on top of block devices. Therefore, this file system does not have the inefficiencies of merged file systems or device-mapper based systems.

This new file system is a user-level file system written in C and integrated into Linux and MacOS via Fuse.  Therefore it does not require any kernel modifications, making it a portable file system. It is a POSIX-compliant file system. Given the ephemeral (temporary) nature of data stored in a graph driver, it is implemented without having some of the complexities of a general-purpose file system (for example, journaling). Most file systems in use today are optimized towards persistent data, provide ACID properties for system calls and aim to work well with random read-write workloads.  This file system is written with container image handling as a specific workload.  These operations involve:

1. Container image creation
2. Container image cloning and launching of instances
3. Container image memory consumption
4. Number of inodes reported to the kernel by way of multiple copies of the same image (or layers) running
5. Container image data management - Actions like deletion, forced image removal and local system resource usage based on multiple container images being present

A lot of these techniques are implemented using image specific snapshotting techniques that are optimized for page cache consumption, image snap creation time and inode count.  Similar to other graph drivers, this graph driver also creates layers for images and read-write layers on top of those for containers.  Each image will have a base layer, in which files of the image are populated initially. Additional layers are created on top of the base layer and for each additional layer in the image being extracted.  Each layer shares data from the previous layer.  If a layer modifies any data, that data is visible from the layers on top of that layer, but not from the layers below it. Also if a container modifies data in an image it is loaded from, that modified data is not visible from any other derived layers on top of that image layer.

A layer in an image is a read-only snapshot sitting on top of the previous layer in the image.  Therefore, these derived layers share common data between each other.  A layer is immutable after its contents are completely populated.  When any existing data inherited from a previous layer is modified while populating data to a new layer, a branch-on-write (BOW) operation is performed in increments of 4KB blocks.  New data will be written to a newly allocated location on the back-end block storage device, and old data will no longer be accessible from the layer that modified the data (or any other layer created on top of that subsequently).  Similarly, any files deleted from a layer are not visible from that layer or on any of the layers on top of that layer.

When a read-write layer is created while starting a new container (two such layers for every container), a read-write snapshot is created on top of the image layer and mounted from the container. The container can see all the data from the image layers below the read-write layer and can create new data or modify any existing data as needed. When any data is modified, the existing data is not modified in the image layer... instead a private copy with new data is made available for the read-write layer.

Traditional file systems need to provide consistency for every system call, but for a graph driver, that is required only when a layer is created, deleted or persisted. This graph driver hosts the Docker database, with information about various images and containers.  It ensures that the database is consistent with the images and therefore the image data can be read correctly regardless of restarts or crashes.  This design eliminates the need to externally monitor or garbage inspect `/var/lib/docker`.

Snapshots are implemented without using any reference counts and thus allow an unlimited number of layers. The time to create a snapshot are independent of the size of the file system (devices), size of the data set, or the number of layers present in the file system. Snapshots are deleted in the background and processing time depends on the amount of data actually created/modified in the snapshot. Thus creation and deletion of layers can be done instantaneously.

A layer becomes read-only after new layers are populated on top of it, and a new layer will not conflict with any modification operations in progress on the parent layer.  In a graph driver, creating a new layer or removing a layer does not have to stop any in-progress operations and thus creating or removing images and containers can proceed without any noticeable impact on any other running containers.

Operations within a layer are independent of the total number of layers present in the file system.  LCFS treats each snapshot, regardless of the number of snapshots, as a sibling of the original layer.  This is well suited to the container image use case.

Layers in a graph are deleted in reverse of the order of creation. The newest layer is deleted first, and then the one created just before it, etc. A layer in the middle of chain or the base layer cannot be deleted when there is a newer layer on top of it. This simplifies the overall snapshot design since deleting a snapshot in the beginning or middle of the chain requires a more complex implementation. For example, each layer can easily track space allocated for storing data created/modified by the layer and this space can be freed without worrying about other layers sharing the data.

Also layers are not for rolling back, thus it does not have to incur some of the complexities of snapshots in a traditional file system. There is also no need to provide any block level differences between any two layers.

## Layout

When a new device is formatted as a new graph driver file system, a superblock with file system specific information is placed at the beginning of the device. This information identiifies this device as having a valid file system on it when it is mounted again in the future. If a device with no valid superblock is mounted as this file system, it is formatted before mounting.

Similarly, each of the layers created in the file system has a private superblock for locating data that belongs exclusively to that layer. Each layer in the file system has a unique index. This index stays the same for the lifetime of the layer.

In addition to the layers created for storing images and containers, a global file system layer keeps data that is not part of any other layer. This layer always has an index of 0. It cannot be deleted.

Superblocks of layers taken on a top of a common layer are linked together. Superblocks of the common layer point to one of these top layer superblocks. Thus superblocks of all layers on top of a layer can be reached from the superblock of that layer.

Available space is tracked using a list of free extents. There will be a single such extent immediately after the file system is formatted. The superblock of layer 0 tracks the blocks where this list is stored. Similarly, all other layers keep track of extents allocated to those layers. Those blocks are also reachable from the superblock of those layers.

4KB is the smallest unit of space allocation or size of I/O to the device, called file system block size. For files larger than 4KB, multiple such blocks can be allocated in a single operation. Every layer shares the whole device, and space can be allocated for any layer from anywhere in the underlying device.

Each file created in any layer has an inode to track information specific to that file such as stat info, dirty data not flushed to disk, etc. Each inode has a unique identifier in the file system called its “inode number.” Files deleted in a layer do not have to maintain any whiteouts as in some union file systems, as their references from the directories are removed in that layer. Inode numbers are not reused even after a file is deleted.

All UNIX file types are supported. For symbolic links, the target name is stored in the same block where inode is written. For directories, separate blocks are allocated for storing directory entries and those blocks are linked in a chain from the inode. For regular files, additional blocks are allocated for storing data and linked from the inode. When a file becomes fragmented, i.e., when an entire file cannot be stored contiguously on disk, additional blocks are allocated to track file page offsets and corresponding disk locations where data is stored, in extent format. Such blocks, called “emap blocks,” are linked from the inode as well. If the file has extended attributes, those are stored in additional blocks and linked from the inode as well. Currently, directories, emap blocks, and extended attribute blocks keep entries as a sequential lists, and they should be switched to use better data structures like B-trees, etc., in future.

All inodes in a layer can be reached from the superblock of the layer. Every inode block is tracked in blocks linked from the superblock. Inodes are not stored in any particular order on disk. Inodes have their number within the inode.

All metadata (superblocks, inodes, directories, emap, extended attributes, etc.) are always cached in memory (although this may change in the future). They are read from disk when file system is mounted and written out when file system is unmounted.

The root directory of the file system has inode number 2 and cannot be removed. Anything created under the tmp directory in root directory is considered temporary.

### Layer root directory

There is another directory under which roots of all layers are placed and called “layer root directory.” This directory cannot be removed once created. This directory is for internal use and creating files in this directory is not allowed.

### File handles

File handles are formed by combining the layer index and the inode number of the file. This is a 64-bit number and is returned to FUSE when files are opened or created. This file handle can be used to locate the file in subsequent operations such as read, readdir, write, truncate, flush, release, etc. The file handle for a shared file, when accessed from different layers, will differ because the layer index part of the file handle is different. This may turn out to be a problem when the same file is read from different layers because multiple copies of data may end up in the kernel page cache. In order to alleviate that problem, pages of a shared file in the kernel page cache are invalidated on its last close (this should be done when a file is closed in kernel, but FUSE does not have any knobs for doing this as of today). Also the direct-mount option cannot be used since that would prevent mmap. Ideally, FUSE should provide an option to bypass the page cache for a file if the file is not mmapped.

### Locking

Each layer has a read-write lock, which is taken in shared mode while reading or writing to the layer (all file operations). This lock is taken in exclusive mode while unmounting the root layer or while deleting any other layer.

Each inode has a read-write lock. Operations that can be run in shared mode (read, readdir, getattr, etc.), take the lock in shared mode, while other operations which modify the inode hold it in exclusive mode. This lock is not taken once a layer is frozen (meaning, a new layer is created on top of that layer and no more changes are allowed in the layer).

### Layers

New layers are added after locking the parent layer in shared mode, if there is a parent layer. The newly created layer will be linked to the parent layer. All the layers taken on a parent layer are linked together as well.

A layer with no parent layer forms a base layer. The base layer for any layer can be reached by traversing the parent layers starting from that layer. All layers with the same base layer form a “tree of layers”.

A layer is removed after locking it in exclusive mode. That makes sure all operations on the layer are drained. Also a shared lock on the base layer is held during that operation.

The root layer is locked in shared mode while creating/deleting layers. The root layer is locked exclusively while unmounting the file system.

### Space management/reclamation

Each layer allocates space in chunks of a few blocks and then files within that layer consume space from those chunks. This eliminates many of the complexities associated with space management faced by file systems that are not designed to
support layers efficiently.

The global pool does not have to be locked down for allocations happening concurrently across different layers in the file system. Another advantage being space allocated in layers will not be fragmented.

Every layer keeps track of space allocated within that layer and all that space can be returned to the global pool when the layer is deleted. Any unused space in reserved chunks returned (this happens as part of sync and unmount as well).

As for shared space between layers, a layer will free space in the global pool only if the space was originally allocated in the layer, not if the space was inherited from a previous layer.

There should be a minimum size for the device to be formatted/mounted as a file system. Operations like writes, file creations and creating new layers are failed when file system free space goes below a certain threshold.

## File operations

All file operations take the shared lock on the layer containing the files that they want to operate on. They can then proceed after taking on the locks of the files involved in those operations in the appropriate mode. For reading shared data, no lock on any inode is needed.

Unlike some other graph drivers, atomic rename is supported. Certain operations like hardlink, rename, etc., are not supported across layers of the file system (if attempted manually).

Access and creation times are not tracked.

### `writes`

Writes are returned immediately after copying new data to inode page table.  Zero blocks written to files are detected. If all data written to a file is zeros, then nothing is written to disk and such files do not consume any disk space. If pages of a file with non-zero data are overwritten with zeroes, then corresponding blocks are freed from the file.

Sparse files are supported and files do not consume space on disk for sparse regions.

Writes that are not page aligned do not trigger an immediate read/modify/write update but are deferred until the application reads the page again or when the page is written to disk. If later writes have filled in the rest of the pages, reading of the page from disk can be completely avoided as the whole page can be written down.

### `fsync`

Fsync is disabled on all files and layers are made persistent when needed. Syncing dirty pages are usually triggered on last close of a file, with the exception of files in global file system.

### `rmdir`

rmdir in global file system layer (layer 0) may succeed even when the directory is not empty. This helps the Docker daemon to delete unneeded directories without iterating over the directory sub-tree.

### `xattrs`

Many UNIX commands unnecessarily query or try to remove extended attributes even when the file does not have any extended attributes. When a file system does not have any files with extended attributes, these operations fail without even looking up and locking the inode. Ideally, the kernel should avoid making these calls when the inode does not have extended attributes (that info could be cached part of previous stat calls).

### `ioctls`

There is support for a few ioctls for operations like creating/removing/loading/unloading layers. Currently, ioctls are supported only on the layer root directory.

## Copying up (BOW, branch-on-write)

When a shared file is modified in a layer, its metadata is copied up completely which includes the inode, whole directory or whole map depending on the type of file, all the extended attributes, etc. Shared metadata may still be shared in cache, but separate copies are made on disk if dirty inode is flushed to disk.

Initially after a copy up of an inode, user data of the file is shared between the newly copied up inode and the original inode and copy-up of user data happens in units of 4KB whenever user data gets modified in the layer. While copying up a page, if a whole page is modified, then old data does not have to be read-in. New data is written to a new location and old data block is untouched.

In practice, most applications truncate the whole file and write new data to the file, thus the pages of files are not copied up.

User data blocks and certain metadata blocks (emap blocks, directory blocks, extended attributes, etc.) are never overwritten in place.

## Caching

As of now, all metadata (inodes, directories, emap, extended attributes, etc.), stay in memory until the layer is unmounted or the layer or file is deleted. There is no upper limit on how many of these can be cached. The metadata is cached, without the page-aligned padding. Almost all metadata is tracked using sequential lists in cache with the exception of directories bigger than a certain size, which use a hash table for tracking file names. The snapshot root directory uses a hash table always, irrespective of the number of layers present.

Each layer maintains a hash table for its inodes using a hash generated from the inode number. This hash table is private to that layer.

When a lookup happens on a file that is not present in a layer’s inode cache, the inode for that file is looked up by traversing the parent layer chain until the inode s found or the base layer is reached, in which case the operation fails with ENOENT. If the operation does not require a private copy of the inode in the layer [for example, operations which simply reading data like getattr(), read(), readdir(), etc.], then the inode from the parent layer is used without making a copy of the inode in the cache. If the operation involves a modification, then the inode is copied up and a new instance of the inode is instantiated in the inode cache of the layer.

Each regular file inode maintains an array for dirty pages of size 4KB indexed by the page number, for recently written or modified pages. If the file is bigger than a certain size and not a temporary file, then a hash table is used instead of the array. These pages are written out when the file is closed in read-only layers, when a file accumulates too many dirty pages, when a layer accumulates too many files with dirty pages, or when the file system is unmounted or persisted. Each regular file inode also maintains a list of extents to track the file's emap if the file is fragmented on disk. When blocks of zeroes are written to a file, they do not create separate copies of the zeros in cache.

Blocks can be cached in chunks of size 4KB, called “pages in block cache.” Pages are cached until the layer is unmounted or the layer is deleted. This block cache has an upper limit for entries.  Pages are recycled when the cache hits this limit. The block cache is shared by all the layers in a layer tree, as data could be shared between layers in the tree. The block cache maintains a hash table using a hash generated from the block number. Pages from the cache are purged under memory pressure and/or when layers are idle for a certain time period.

As the user data is shared, multiple layers sharing the same data will use the same page in the block cache, all looking up the data using its block number. Thus there will not be multiple copies of the same data in page cache. Pages cached in this private block cache are mostly shared data between layers. Data that is not shared between layers is still cached in the kernel page cache.

## Data placement

Space for files is not allocated when data is written to the file, but later when dirty data is flushed to disk. Since the size of the file is known at the time of space allocation, all the blocks needed for the file can be allocated as single extent if the file system is not fragmented. With the read-only layers created while populating images, files are written once and never modified and this scheme of deferred allocation helps keep the files contiguous on disk. Also temporary files may never get written to disk (large temporary files are created for image tar files).

This also helps when applications are writing to a file randomly or if writes are not page aligned. Also if writes received on a file are all zeroes, those are not written to disk and therefore do not consume any space on disk.

Such a scheme also helps writing out small files after coalescing many of them together. Similarly, metadata is also placed contiguously and written out together.

As every attempt is made to place files contiguously on disk, that benefits in consuming less memory (less metadata), less disk space, and less overhead.

## I/O coalescing

When space for a file is allocated contiguously as part of flush, the dirty pages of the file can be flushed in large chunks, reducing the number of I/Os issued to the device. Similarly, space for small files is allocated contiguously and the pages are written out in large chunks.  Metadata blocks such as inode blocks, directory blocks etc, are are allocated contiguously on disk and written out in chunks.

## Crash Consistency (TODO)

If the graph driver is not shutdown normally, the Docker database and layers in the graph driver need to be consistent. Each layer needs to be consistent as well. As the graph driver manages both Docker database and images/containers, these are kept in a consistent state by checkpointing. Thus this file system does not have the complexity of journaling schemes typically used in file systems to provide crash consistency.

## Layer Diff (for Docker build/commit)

Finding differences between any two layers is simply finding differences in the sets of inodes present in layers between the old layer and new layer (inclusive). As of now, this work is pending and the graph driver is using the default NaiveDiffDriver.

## Stats

When enabled at build time, all file operations and ioctl requests are counted and times taken for each of them are tracked for each layer separately. These stats can be queried using a command. Currently, they are also displayed at the time that a layer is unmounted. Stats for a layer can be cleared before running applications to trace actual operations during any time period.

Memory usage on a per-layer basis is tracked and reported as well. Similarly, a count of files of different types in every layer is maintained. The count of I/Os issued by each layer is also tracked.

# Contributing

The specification and code is licensed under the Apache 2.0 license found in 
the `LICENSE` file of this repository.  

See the [Style Guide](STYLEGUIDE.md).


### Sign your work

The sign-off is a simple line at the end of the explanation for the
patch, which certifies that you wrote it or otherwise have the right to
pass it on as an open-source patch.  The rules are pretty simple: if you
can certify the below (from
[developercertificate.org](http://developercertificate.org/)):

```
Developer Certificate of Origin
Version 1.1

Copyright (C) 2004, 2006 The Linux Foundation and its contributors.
660 York Street, Suite 102,
San Francisco, CA 94110 USA

Everyone is permitted to copy and distribute verbatim copies of this
license document, but changing it is not allowed.


Developer's Certificate of Origin 1.1

By making a contribution to this project, I certify that:

(a) The contribution was created in whole or in part by me and I
    have the right to submit it under the open source license
    indicated in the file; or

(b) The contribution is based upon previous work that, to the best
    of my knowledge, is covered under an appropriate open source
    license and I have the right under that license to submit that
    work with modifications, whether created in whole or in part
    by me, under the same open source license (unless I am
    permitted to submit under a different license), as indicated
    in the file; or

(c) The contribution was provided directly to me by some other
    person who certified (a), (b) or (c) and I have not modified
    it.

(d) I understand and agree that this project and the contribution
    are public and that a record of the contribution (including all
    personal information I submit with it, including my sign-off) is
    maintained indefinitely and may be redistributed consistent with
    this project or the open source license(s) involved.
```

then you just add a line to every git commit message:

    Signed-off-by: Joe Smith <joe@gmail.com>

using your real name (sorry, no pseudonyms or anonymous contributions.)

You can add the sign off when creating the git commit via `git commit -s`.
