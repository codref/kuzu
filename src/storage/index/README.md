# Index

The index module implements hash-based indexing structures for efficient key lookups and primary key enforcement. It supports both persistent on-disk indexes and transient in-memory indexes used during bulk data loading.

## Overview

This module provides hash index implementations that map keys to row offsets or node IDs. The indexes support concurrent access, transactional updates, and efficient bulk insertion patterns. Hash indexes are primarily used for primary key constraints and fast point lookups.

## Key Components

### HashIndex
The main persistent hash index implementation that stores index data on disk. Supports lookups, insertions, and deletions with transactional semantics. Uses separate local storage for uncommitted changes to provide isolation between transactions.

Key features:
- Persistent storage of index slots
- Local insertions and deletions for uncommitted changes
- Lock-free lookups with optimistic concurrency
- Efficient probing with linear and quadratic strategies
- Support for variable-length keys via overflow storage

Operations:
- `lookup()`: Find value for a given key, checking local deletions/insertions first
- `insert()`: Add a new key-value pair, tracking in local storage
- `delete()`: Mark a key as deleted in local storage
- `checkpoint()`: Merge local changes into persistent storage

### InMemHashIndex
An in-memory hash index used during bulk loading operations. Provides fast insertion without the overhead of maintaining persistent structures. Converted to a persistent HashIndex after bulk loading completes.

Key features:
- Fast bulk insertion without disk I/O
- No transaction overhead during loading
- Efficient memory layout for sequential inserts
- Conversion to persistent format after loading

### HashIndexHeader
Manages metadata for the hash index including table size, number of entries, and structural parameters. Tracks separate headers for read and write transactions to support MVCC.

### HashIndexSlot
Represents individual slots in the hash table. Each slot contains a key-value pair and metadata for collision resolution.

## Files

- `hash_index.cpp`: Main persistent hash index implementation
- `in_mem_hash_index.cpp`: In-memory bulk loading index
- `index.cpp`: Base index interface and factory

## Hash Table Structure

The hash index uses open addressing with the following characteristics:

1. **Slot Structure**: Each slot stores the key (or key offset for variable-length keys) and the associated value
2. **Collision Resolution**: Uses probing to find the next available slot
3. **Resizing**: Dynamically grows the hash table when load factor exceeds threshold
4. **Overflow Handling**: Variable-length keys stored in separate overflow file

## Transaction Support

Indexes maintain both persistent and local storage:

- **Persistent Storage**: Committed index data on disk
- **Local Deletions**: Set of keys marked as deleted in current transaction
- **Local Insertions**: New key-value pairs added in current transaction

On lookup, the index:
1. Checks local deletions - return not found if key is deleted
2. Checks local insertions - return value if key is locally inserted
3. Checks persistent storage - return value if key exists in committed data

This provides read-committed isolation without blocking concurrent reads.

## Performance Characteristics

- O(1) average-case lookup time
- Lock-free reads via optimistic concurrency
- Bulk loading optimized with in-memory index
- Minimal checkpoint overhead via incremental merging
