# Stats

The stats module collects and maintains statistical information about tables and columns. These statistics are used by the query optimizer for cardinality estimation and join order selection.

## Overview

This module implements data structures and algorithms for tracking table cardinality and column characteristics. Statistics are updated incrementally during data modifications and persisted during checkpoint operations.

## Key Components

### TableStats
Maintains statistics for an entire table including total row count and per-column statistics. Provides methods to update statistics incrementally and merge statistics from multiple sources.

Key metrics:
- Table cardinality (estimated row count)
- Column-level statistics for each column

Operations:
- `incrementCardinality()`: Increase table row count estimate
- `merge()`: Combine statistics from another TableStats object
- `update()`: Update statistics based on new data in value vectors
- `getNumDistinctValues()`: Get distinct value estimate for a column

### ColumnStats
Maintains statistics for individual columns including distinct value estimates. Uses HyperLogLog for efficient cardinality estimation with low memory overhead.

Key metrics:
- Number of distinct values (approximate)
- Null count
- Data type information

Operations:
- `update()`: Process new values to update statistics
- `merge()`: Combine with another ColumnStats object
- `getNumDistinctValues()`: Get estimated distinct count

### HyperLogLog
Implements the HyperLogLog algorithm for approximate distinct counting. Provides memory-efficient cardinality estimation with configurable precision.

Features:
- Constant memory usage regardless of cardinality
- Approximate counts with low error rate
- Efficient merging of multiple HyperLogLog sketches
- Support for various data types through hashing

Operations:
- `add()`: Insert a value into the sketch
- `merge()`: Combine with another HyperLogLog sketch
- `getCount()`: Estimate the number of distinct values

## Files

- `table_stats.cpp`: Table-level statistics management
- `column_stats.cpp`: Column-level statistics tracking
- `hyperloglog.cpp`: HyperLogLog approximate distinct counting

## Statistics Collection

Statistics are updated during:

1. **Bulk Loading**: Initial statistics computed during data import
2. **Insertions**: Incrementally updated when new rows are added
3. **Updates**: Refreshed when existing values are modified
4. **Deletions**: Adjusted when rows are removed (cardinality only)

## HyperLogLog Algorithm

The HyperLogLog algorithm works by:

1. **Hashing**: Each value is hashed to a uniform bit string
2. **Register Selection**: Leading bits select one of many registers
3. **Leading Zeros**: Trailing bits are examined for leading zeros
4. **Maximum Tracking**: Each register tracks the maximum leading zeros seen
5. **Estimation**: Cardinality estimated from harmonic mean of register values

This provides:
- Memory usage: O(m) where m is number of registers
- Error rate: ~1.04/sqrt(m)
- Typical configuration: 16KB memory, ~0.8% error

## Usage by Optimizer

The query optimizer uses statistics for:

1. **Cardinality Estimation**: Estimate result size of operations
2. **Join Order Selection**: Choose optimal join order based on selectivity
3. **Index Selection**: Decide when to use indexes vs table scans
4. **Memory Budgeting**: Allocate memory for hash tables and sorts

## Persistence

Statistics are serialized during checkpoint:
- Stored alongside catalog metadata
- Loaded during database initialization
- Updated incrementally during transaction commit
- Merged during concurrent updates
