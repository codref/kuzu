# Storage Module

The storage module is the core persistence layer of Kuzu, responsible for managing all data storage, retrieval, and transaction processing. It provides a columnar disk-based storage system optimized for graph data and analytical workloads.

## Overview

This module implements the storage engine that handles:
- Physical data layout and persistence
- Buffer and memory management
- Transaction isolation and consistency
- Data compression and encoding
- Indexing structures
- Write-ahead logging for durability

## Key Components

### StorageManager
The central coordinator for all storage operations. Manages tables, indexes, and coordinates with the buffer manager and WAL. Handles database initialization, checkpoint operations, and recovery procedures.

### Checkpointer
Manages the checkpointing process to persist in-memory changes to disk. Ensures consistency between the main data file, shadow pages, and the write-ahead log. Coordinates serialization of catalog metadata and table data.

### BufferManager (buffer_manager/)
Handles memory allocation and page eviction policies. Manages the buffer pool that caches disk pages in memory. Implements an optimistic concurrency control mechanism with page states.

### Table (table/)
Implements the physical storage structures for node and relationship tables. Uses a columnar format with node groups for efficient scanning and compression. Manages column chunks, dictionary encoding, and versioning.

### Index (index/)
Provides hash-based indexing for primary keys and efficient lookups. Supports both persistent on-disk indexes and in-memory indexes during bulk loading operations.

### WAL (wal/)
Implements write-ahead logging for durability and crash recovery. Records all modifications before they are applied to the main database file. Supports replaying logs during database recovery.

### Compression (compression/)
Implements various compression techniques including bitpacking, dictionary encoding, and floating-point compression. Reduces storage footprint while maintaining query performance.

### LocalStorage (local_storage/)
Maintains transaction-local changes that are not yet committed. Provides isolation between concurrent transactions by storing uncommitted insertions, updates, and deletions separately from the persistent storage.

### Stats (stats/)
Collects and maintains statistics about tables and columns, including cardinality estimates and distinct value counts using HyperLogLog. Used by the query optimizer for cost estimation.

### Predicate (predicate/)
Implements column predicates and zone map filtering for query optimization. Enables early pruning of data chunks based on min/max statistics without reading the actual data.

## Core Files

- `storage_manager.cpp`: Main storage coordinator
- `checkpointer.cpp`: Checkpoint and recovery logic
- `disk_array.cpp`: Disk-based array structures
- `file_handle.cpp`: File management and I/O operations
- `page_manager.cpp`: Page allocation and management
- `shadow_file.cpp`: Shadow paging for atomic updates
- `overflow_file.cpp`: Storage for variable-length data
- `undo_buffer.cpp`: Undo log for transaction rollback
- `free_space_manager.cpp`: Tracks free space in pages

## Design Principles

- Columnar storage for analytical query performance
- Vectorized processing of data chunks
- Multi-version concurrency control (MVCC) for transaction isolation
- Shadow paging for atomic checkpoints
- Compression-aware query execution
- Efficient handling of variable-length data
