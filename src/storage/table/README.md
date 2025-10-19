# Table

The table module implements the physical storage layout for node and relationship tables. It provides a columnar storage format optimized for analytical queries and graph traversals.

## Overview

This module contains the core table storage structures that organize data into node groups and column chunks. Tables use a hybrid row-column format where data is partitioned into node groups, and within each group, data is stored in compressed column chunks.

## Key Components

### Table
Base class for all table types. Provides common interfaces for scanning, insertion, update, and deletion operations. Manages the collection of node groups that comprise the table.

### NodeTable
Stores node data including node properties and primary keys. Organizes nodes into node groups with columnar storage for each property. Supports efficient property access and primary key lookups.

Key features:
- Columnar storage within node groups
- Primary key index for fast lookups
- Support for variable-length properties
- Multi-version concurrency control

### RelTable
Stores relationship data including relationship properties and adjacency information. Uses CSR (Compressed Sparse Row) format to efficiently represent graph structure.

Key features:
- CSR adjacency lists for source and destination
- Columnar property storage
- Efficient neighborhood traversal
- Support for directed relationships

### NodeGroup
Represents a horizontal partition of a table containing a batch of rows. Each node group stores data in columnar format with one column chunk per property.

Key characteristics:
- Fixed maximum size (typically 262,144 rows)
- Columnar layout within group
- Independent compression per column
- Version information for MVCC

Types:
- `ChunkedNodeGroup`: Standard node group with column chunks
- `CSRNodeGroup`: Specialized for CSR adjacency data in rel tables
- `InMemChunkedNodeGroup`: Transient node group during bulk loading

### Column
Represents a single column of data across all node groups. Provides unified access to column data regardless of which node group contains it.

Key features:
- Abstraction over multiple column chunks
- Handles different data types uniformly
- Supports null values
- Integrates with compression

Types:
- `Column`: Base column implementation
- `DictionaryColumn`: Column with dictionary encoding
- `StringColumn`: Specialized for variable-length strings
- `ListColumn`: Nested list data
- `StructColumn`: Nested struct data
- `NullColumn`: Optimized for all-null columns

### ColumnChunk
The basic unit of columnar storage containing data for one column in one node group. Each chunk stores data in compressed format with metadata for statistics and zone maps.

Key features:
- Compressed storage with various encoding schemes
- Min/max statistics for zone map filtering
- Null bitmap for efficient null checking
- Page-based storage for buffer management

Components:
- `ColumnChunkData`: Actual data storage with compression
- `ColumnChunkMetadata`: Statistics and structural information
- `ColumnChunkStats`: Min/max values for zone maps

### ColumnReader/Writer
Handle serialization and deserialization of column data. Support various data types and compression schemes. Coordinate with the buffer manager for page I/O.

## Files

- `table.cpp`: Base table interface
- `node_table.cpp`: Node table implementation
- `rel_table.cpp`: Relationship table implementation
- `node_group.cpp`: Node group base class
- `chunked_node_group.cpp`: Standard columnar node group
- `csr_node_group.cpp`: CSR adjacency node group
- `column.cpp`: Column abstraction
- `column_chunk.cpp`: Column chunk storage
- `column_chunk_data.cpp`: Compressed data storage
- `column_chunk_metadata.cpp`: Chunk metadata and statistics
- `column_reader_writer.cpp`: Serialization logic
- `version_info.cpp`: MVCC version tracking

## Storage Layout

```
Table
├── NodeGroup 0 (rows 0-262143)
│   ├── Column 0 Chunk (compressed)
│   ├── Column 1 Chunk (compressed)
│   └── ...
├── NodeGroup 1 (rows 262144-524287)
│   ├── Column 0 Chunk (compressed)
│   └── ...
└── ...
```

Each column chunk contains:
- Compressed data pages
- Null bitmap
- Min/max statistics
- Metadata (compression type, page count, etc.)

## CSR Format (Relationship Tables)

Relationship tables use CSR to efficiently store adjacency:

```
Adjacency stored as:
- Offset array: Cumulative neighbor count for each source node
- Neighbor array: Destination node IDs in adjacency order
- Property arrays: Relationship properties aligned with neighbor array
```

This enables:
- O(1) access to neighbor list for a source node
- Efficient iteration over all neighbors
- Cache-friendly sequential access patterns

## Multi-Version Concurrency Control (MVCC)

Tables maintain version information for MVCC:

- `VersionInfo`: Tracks insert and delete transaction IDs per row
- `VersionRecordHandler`: Manages version records
- `UpdateInfo`: Stores before-images for updates

This provides:
- Read-committed isolation
- Non-blocking reads
- Efficient rollback through version chains

## Scan Operations

Table scanning supports:
- Sequential scan of all rows
- Predicate pushdown with zone map filtering
- Column projection to read only needed columns
- Integration with local storage for uncommitted data
- MVCC visibility checks

## Update Operations

Updates are handled through:
1. Check visibility of row being updated
2. Store before-image in version record
3. Apply update to column chunk
4. Track version information for rollback
