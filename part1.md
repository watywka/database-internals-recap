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
