# Part 1 Storage engines

## Introduction

Throughput - the number of transactions the database is able to process per minute

### Client/server model

- Transport (cluster and client communication)
- Query processor (parser and optimizer)
- Executor engine (remote and local execution)
- Storage engine:
  - Transactions
  - Locks
  - Access (storage structures)
  - Buffer
  - Recovery

In-memory stores use write-ahead logs and backup copies.

Disk-based dbs use specialized storage structures, optimized for disk access.

How the data is stored on disk: row- or column-wise.
Choose according to yout access patterns.

Column-oriented dbs != wide column stores (BigTable, HBase).

In wide column stores data is represented as a multidimensional map, columns are grouped into column families (usually storing data of the same type), and inside each column family, data is stored row-wise.

DBs use specific format to store data

### Goals

- Storage efficiency
- Access efficiency
- Update efficiency

Indexes are used to locate a record without full scan.

A db usually separates data (primary) files and index files.

Files are partitioned into pages.

Data files organisation:

- Index-organized tables
  - Stores data in the index itself
  - This lets range scan to be a simple sequantial scan
- Heap-organized tables
  - No particular order, mostly write order
  - Require additional index structures to map keys to data location
- Hash-organized tables
  - Stored in buckets by key hash
  - No particular order inside of the bucket, mostly write order

Index files are organized as special structures that map keys to locations in data files where the records corresponding to the keys are stored.

If the order of data records folloaws the search key order, index is called clustered.

An index on primary file is called the primary index. All others are called secondary.
Secondary indexes can point to the key or directly to the record.
Secondary indexes may hold multiple entries for a single search value.

Things to consider in stotage engines:

- Buffering  
Whether or not the storage structure collects a certain amount of data in memory before putting it on disk.
- Immutability  
Whether or not storage can modify data files. Immutable structures may be append-only (write modifications at the end of the file), copy-on-write (write updated records to the new location), i.e.
- Ordering  
Whether or not the data records are stored in the key order in the pages on disk. Alternative is to store records in the order of insertion.

## B-Tree Basics

Most of the mutable storages update records in-place.

For now we assume that at most one record corresponds for each key (this is not the case for some storage engines).

B-Tree is one of the most popular storage structures. Intorduced back in 1971.

### Binary Search Tree (BTS)

We need to rotate the tree after each operation in order to keep it balanced, otherwise the worst-case complexity goes up to O(N).
Maintaining BTS on disk faces several problems:

- Locality  
Close nodes may be stored on different disk pages. This can be improved by modifying the tree layout and using paged binary trees.
- Tree height  
Low fanout leeds to up to O(log N) amount of seeks to locate the searched element.

### Disk-Based Structures

Many algorithms have been designed to work excel on hard disk drives, where seek operation is an expansive operation and reading/writing consecutive bytes after that is relativily cheap.

Solid state drives do not have moving parts. The strurcture is following:

 Die -> Plane -> Block -> Page -> Array -> String -> Memory cell

 The smallest read/write block is page. Before writing page needs to be emptied, but we can only erase multiple pages at a time, this is called **erase block**.

 Flash translation layer is responsible for mapping page ids to their physical locations and tracking empty/writter/discarded pages. It is also responsible for garbage collection.

 Most operation systems have a block device abstractiion layer.

 In SSD random I/O does not have that much overhead compared to HDD.

 On disk, most of the time we manage the data layout manually. Creating long dependency chains in on-disk structures greatly increases code and structure complexity.

### B-Trees

B-Trees are beuilt upon the foundation of balances trees, but have higher fanout and lower height.

B-Trees are sorted: keys inside the B-Tree are stored in order. B-Trees are good for both point and range queries.

Since B-Trees are a page organization technique (i.e., they are used to organize and navigate fixed-size pages), we often use terms node and page interchangeably.

The relation between the node capacity and the number of keys it actually holds is called occupancy.

Balancing operations (namely, splits and merges) are triggered when the nodes are full or nearly empty.

B-Tree lookup complexity can be viewed from two standpoints: the number of block transfers (log M, M is the total number of items in B-Tree) and the number of comparisons done during the lookup (log M).

To insert the value into a B-Tree, we first have to locate the target leaf and find the insertion point. After the leaf is located, the key and value are appended to it. Updates in B-Trees work by locating a target leaf node using a lookup algorithm and associating a new value with an existing key.
If the target node doesn’t have enough room available, we say that the node has overflowed and has to be split in two to fit the new data.
The node is split if the following conditions hold:

- For leaf nodes: if the node can hold up to N key-value pairs, and inserting one more key-value pair brings it over its maximum capacity N.
- For nonleaf nodes: if the node can hold up to N + 1 pointers, and inserting one more pointer brings it over its maximum capacity N + 1.

Splits are done by allocating the new node, transferring half the elements from the splitting node to it, and adding its first key and pointer to the parent node. In this case, we say that the key is promoted. The index at which the split is performed is called the split point. All elements after the split point (including split point in the case of nonleaf node split) are transferred to the newly created sibling node, and the rest of the elements remain in the splitting node.

If the parent node is full, it is split as well. This may repeat up to the root. When the root is split, a new root is allocated that holds the split key. This increased height by one.

Node split algorithm:

1. Allocate a new node.
2. Copy half the elements from the splitting node to the new one.
3. Place the new element into the corresponding node.
4. At the parent of the split node, add a separator key and a pointer to the new node.

Deletions are done by first locating the target leaf. When the leaf is located, the key and the value associated with it are removed.
If neighboring nodes have too few values (i.e., their occupancy falls under a threshold), the sibling nodes are merged. This situation is called underflow.
Underflow scenarios:

- if two adjacent nodes have a common parent and
their contents fit into a single node, their contents should be merged (concatenated);
- if their contents do not fit into a single node, keys are redistributed between them to
restore balance.
More precisely, two nodes are merged if the following conditions hold:

- For leaf nodes: if a node can hold up to N key-value pairs, and a combined number of key-value pairs in two neighboring nodes is less than or equal to N.
- For nonleaf nodes: if a node can hold up to N + 1 pointers, and a combined number of pointers in two neighboring nodes is less than or equal to N + 1.

Just as with splits, merges can propagate all the way to the root level.

Node merge algorithm:

1. Copy all elements from the right node to the left one.
2. Remove the right node pointer from the parent (or demote it in the case of a non‐
leaf merge).
3. Remove the right node.

## File Formats

### Binary encoding  

To store data on disk efficiently, it needs to be encoded using a format that is compact
and easy to serialize and deserialize.

Keys and values have a type, such as integer, date, or string, and can be represented (serialized to and deserialized from) in their raw binary forms. Most numeric data types are represented as fixed-size values. When working with multibyte numeric values, it is important to use the same byte-order (endianness) for both encoding and decoding.

Strings and variable-size data can be serialized as a numberr, representing the length of the array or string, followed by size bytes: the actual data.

Booleans have only two values, using an entire byte for its representation is wasteful, and developers often batch boolean values together in groups of eight, each boolean occupying just one bit.

### General principles

Usually, you start designing a file format by deciding how the addressing is going to be done: whether the file is going to be split into same-sized pages, which are represented by a single block or multiple contiguous blocks. Most in-place update storage structures use pages of the same size, since it significantly simplifies read and write access. Append-only storage structures often write data page-wise, too: records are appended one after the other and, as soon as the page fills up in memory, it is flushed on disk.

The file usually starts with a fixed-size header and may end with a fixed-size trailer, which hold auxiliary information that should be accessed quickly or is required for decoding the rest of the file. The rest of the file is split into pages.

### Page structure

Database systems store data records in data and index files. These files are partitioned into fixed-size units called pages, which often have a size of multiple filesystem blocks. Page sizes usually range from 4 to 16 Kb.

### Slotted pages

When storing variable-size records, the main problem is free space management: reclaiming the space occupied by removed records. If we attempt to put a record of size n into the space previously occupied by the record of size m, unless m == n or we can find another record that has a size exactly m – n, this space will remain unused. Similarly, a segment of size m cannot be used to store a record of size k if k is larger than m, so it will be inserted without reclaiming the unused space.

Space reclamation can be done by simply rewriting the page and moving the records around, but we need to preserve record offsets, since out-of-page pointers might be using these offsets. It is desirable to do that while minimizing space waste, too.

What we need:

- Store variable-size records with minimum overhead
- Reclaim space occupied by the removed records
- Reference records in the page without regard to their exact location

To efficiently store variable-size records such as string, BLOBs, etc., we can use slotted page or slotted directory.

We organize each page into a collection of slots. A slotted-page has a fixed-size header, pointers region and cells region.

Advantages:

- Minimal overhead, only pointer array.
- Space reclamation can be done by defragmating the page
- Dynamic layout, outside the page slots are referenced by ids.

Two types of cells:

- Key
  - Key size
  - Id of the child page the cell is pointing to
  - Key bytes
- Key-value
  - Key size
  - Value size
  - Key bytes
  - Data record bytes

To organize cells into slotted pages we append them to the right size of the page and keep cell offset in the left side of the page.
While cells are stored in the insertion order, offsets are key ordered.
Removed cells are marked as deleted and added to the availability list. It stores offsets of freed segments and their sizes. Before record insertion we check the list for a segment it may fit.

Fit strategies:

- First fit
- Best fit

If no fit is found we check the total number of free space, if it is greater than the record size, we defragment the page, otherwise an overflow page is created.

Files on disk may get damaged or corrupted by software bugs and hardware failures. To identify these problems preemptively and avoid propagating corrupt data to other subsystems or even nodes, we can use checksums and cyclic redundancy checks (CRCs).

Checksums provide the weakest form of guarantee and aren’t able to detect corruption in multiple bits. They’re usually computed by using XOR with parity checks or summation.

CRCs can help detect burst errors (e.g., when multiple consecutive bits got corrupted) and their implementations usually use lookup tables and polynomial division. Multibit errors are crucial to detect, since a significant percentage of failures in communication networks and storage devices manifest this way.

Before writing the data on disk, we compute its checksum and write it together with the data. When reading it back, we compute the checksum again and compare it with the written one. If there’s a checksum mismatch, we know that corruption has occur red and we should not use the data that was read.

Since computing a checksum over the whole file is often impractical, page checksums are usually computed on pages and placed in the page header. This way, checksums can be more robust (since they are performed on a small subset of the data), and the whole file doesn’t have to be discarded if corruption is contained in a single page.
