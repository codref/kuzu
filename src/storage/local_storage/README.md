# Local Storage

The local storage module manages transaction-local data that has not yet been committed to persistent storage. It provides isolation between concurrent transactions by maintaining separate storage for uncommitted changes.

## Overview

This module implements the transaction isolation layer by storing all insertions, updates, and deletions in memory until commit time. Local storage is created per transaction and discarded on rollback or merged into persistent storage on commit.

## Key Components

### LocalStorage
The main coordinator for transaction-local changes. Maintains separate local tables for each table being modified in the transaction. Manages optimistic allocators for temporary page allocation.

Key responsibilities:
- Create and manage local tables for modified tables
- Track optimistic page allocations
- Coordinate commit by flushing local changes to persistent storage
- Coordinate rollback by discarding all local changes

Operations:
- `getOrCreateLocalTable()`: Get or create local table for a given table
- `commit()`: Flush all local changes to persistent storage
- `rollback()`: Discard all local changes

### LocalTable
Base class for table-specific local storage. Provides the interface for storing uncommitted data that is specific to each table type (node or relationship tables).

### LocalNodeTable
Manages local storage for node tables. Stores newly inserted nodes and their property values. Tracks which nodes have been added in the current transaction.

Key features:
- In-memory storage of new node properties
- Efficient batch insertion of multiple nodes
- Integration with table statistics for cardinality updates

### LocalRelTable
Manages local storage for relationship tables. Stores newly inserted relationships and their property values. Maintains CSR (Compressed Sparse Row) adjacency structures for efficient access.

Key features:
- In-memory storage of new relationship properties
- CSR structure for source/destination node adjacency
- Support for relationship property updates

### LocalHashIndex
Maintains a local hash index for tracking insertions and deletions to persistent indexes. Keeps sets of locally inserted and deleted keys to provide transactional semantics.

Key features:
- Track locally inserted keys
- Track locally deleted keys
- Merge with persistent index on commit

## Files

- `local_storage.cpp`: Main coordinator for transaction-local state
- `local_node_table.cpp`: Node table local storage implementation
- `local_rel_table.cpp`: Relationship table local storage implementation

## Transaction Isolation

Local storage provides read-committed isolation:

1. **Reads**: See committed data plus local uncommitted changes
2. **Writes**: Stored in local storage, invisible to other transactions
3. **Commit**: Local changes become visible atomically
4. **Rollback**: Local changes are discarded without affecting persistent data

## Memory Management

Local storage uses optimistic allocators for temporary pages:
- Pages allocated from the buffer pool optimistically
- Assumed to be available without contention
- Merged into free space manager on commit
- Returned to buffer pool on rollback

## Scan Integration

Table scans check both committed and local data:
1. First scan committed data from persistent storage
2. Then scan uncommitted data from local storage
3. Results merged to provide consistent view
