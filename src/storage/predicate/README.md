# Predicate

The predicate module implements column predicates and zone map filtering for query optimization. It enables early pruning of data chunks without reading actual data by evaluating predicates against min/max statistics.

## Overview

This module provides predicate pushdown capabilities that allow the query engine to skip reading entire chunks of data when predicates can be evaluated against chunk-level statistics. This significantly reduces I/O for selective queries.

## Key Components

### ColumnPredicate
Base class for predicates that can be evaluated against column chunk statistics. Defines the interface for checking whether a predicate can potentially match values in a chunk based on min/max bounds.

Predicate types:
- Equality predicates (=)
- Comparison predicates (<, <=, >, >=)
- Range predicates (BETWEEN)
- Null checks (IS NULL, IS NOT NULL)

### ConstantPredicate
Represents predicates comparing a column to a constant value. Used for conditions like `age > 25` or `name = 'Alice'`. Can be evaluated against zone maps to determine if a chunk might contain matching rows.

Operations:
- `checkZoneMap()`: Returns ALWAYS_TRUE, ALWAYS_FALSE, or MIGHT_CONTAIN
- Comparison with min/max statistics to prune chunks

### NullPredicate
Represents null checks on columns (IS NULL / IS NOT NULL). Evaluated against null count statistics in chunk metadata.

Operations:
- `checkZoneMap()`: Returns ALWAYS_TRUE if all values are null, ALWAYS_FALSE if no nulls exist, or MIGHT_CONTAIN otherwise

### ColumnPredicateSet
Container for multiple predicates on different columns. Combines multiple predicates using AND semantics. Allows conjunctive predicates to be evaluated together against chunk statistics.

Operations:
- `addPredicate()`: Add a new predicate to the set
- `checkZoneMap()`: Evaluate all predicates against chunk statistics
- Returns ALWAYS_FALSE if any predicate fails, MIGHT_CONTAIN if all might match

## Files

- `column_predicate.cpp`: Base predicate infrastructure and predicate sets
- `constant_predicate.cpp`: Constant comparison predicates
- `null_predicate.cpp`: Null check predicates

## Zone Map Filtering

Zone maps store min/max statistics for each column chunk:

1. **Statistics Collection**: During chunk compression, min/max values are tracked
2. **Predicate Conversion**: Query predicates are converted to ColumnPredicate objects
3. **Chunk Pruning**: For each chunk, predicates are evaluated against statistics
4. **Skip or Scan**: Chunks returning ALWAYS_FALSE are skipped entirely

## Check Results

Predicate evaluation against zone maps returns:
- `ALWAYS_TRUE`: All values in chunk satisfy the predicate (rare)
- `ALWAYS_FALSE`: No values in chunk can satisfy the predicate (skip chunk)
- `MIGHT_CONTAIN`: Chunk might contain matching values (must scan)

## Performance Benefits

Zone map filtering provides:
- Reduced I/O by skipping irrelevant chunks
- Reduced decompression overhead
- Reduced CPU for predicate evaluation
- Especially effective for range queries and selective predicates

## Integration

Predicates are pushed down through the query plan:
1. Binder converts filter expressions to predicates
2. Optimizer pushes predicates to table scans
3. Table scan evaluates predicates on chunk metadata
4. Only promising chunks are read and decompressed
