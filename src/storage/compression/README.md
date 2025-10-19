# Compression

The compression module provides data encoding and compression techniques to reduce storage footprint while maintaining efficient query processing. It implements various compression schemes optimized for different data types and patterns.

## Overview

This module contains compression algorithms that work at the column chunk level. Compression is applied during data ingestion and checkpoint operations, with decompression happening transparently during query execution. The system chooses compression methods based on data characteristics.

## Compression Techniques

### Bitpacking
Compresses integers by packing them into the minimum number of bits required. Analyzes the range of values in a chunk and packs multiple values into each storage word.

Features:
- Efficient for integers with small dynamic range
- Supports signed and unsigned integers
- Special handling for 128-bit integers
- SIMD-optimized compression and decompression

### Float Compression
Implements the ALP (Adaptive Lossless floating-Point) compression algorithm for floating-point values. Achieves high compression ratios for scientific and analytical data.

Features:
- Lossless compression for floats and doubles
- Adaptive encoding based on data distribution
- Efficient decompression for vectorized processing
- Handles special values (NaN, infinity) correctly

### Dictionary Encoding
Implicitly supported through the dictionary chunk structures in the table module. Maps repeated values to small integer codes.

## Files

- `compression.cpp`: Core compression infrastructure and utilities
- `bitpacking_utils.cpp`: Bitpacking helper functions and SIMD operations
- `bitpacking_int128.cpp`: Specialized bitpacking for 128-bit integers
- `float_compression.cpp`: ALP floating-point compression implementation

## Usage Pattern

Compression is applied automatically during:
1. Data loading and bulk insertion
2. Checkpoint operations that flush in-memory data
3. Updates that trigger column chunk recompression

Decompression happens on-demand during:
1. Table scans reading column chunks
2. Index lookups requiring data access
3. Predicate evaluation on compressed data

## Performance Considerations

- Compression trades CPU for I/O and storage
- Chunk-level granularity balances compression ratio and access overhead
- Vectorized decompression maintains query performance
- Statistics tracked during compression enable zone map filtering
- Compression decisions can be influenced by data distribution analysis
