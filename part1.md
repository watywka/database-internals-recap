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
