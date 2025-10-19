# Buffer Manager

The buffer manager subsystem handles memory allocation, buffer pool management, and page eviction for the storage engine. It provides an abstraction layer between physical disk storage and in-memory data processing.

## Overview

This module manages the fixed-size buffer pool that caches disk pages in memory. It implements sophisticated eviction policies and concurrency control mechanisms to maximize cache hit rates while ensuring data consistency.

## Key Components

### BufferManager
The main buffer pool manager that handles page pinning, unpinning, and eviction. Implements an optimistic concurrency control scheme using atomic page states. Manages multiple file handles and coordinates with the memory manager for allocation limits.

Key responsibilities:
- Pin and unpin pages for read/write access
- Evict pages when memory pressure occurs
- Track page states (unlocked, locked, marked, evicted)
- Handle page faults by reading from disk
- Coordinate shadow page updates during checkpoints

### MemoryManager
Manages the overall memory budget for the system. Tracks memory usage across buffer pool, intermediate query results, and other allocations. Enforces memory limits and triggers spilling when necessary.

Key responsibilities:
- Allocate and track memory blocks
- Enforce memory usage limits
- Coordinate with spiller for disk-based overflow
- Provide memory allocators for different components

### Spiller
Handles spilling of data to disk when memory limits are exceeded. Works with the memory manager to write temporary data to disk files and read it back when needed. Used during large sorts, aggregations, and hash joins.

Key responsibilities:
- Write spilled data to temporary files
- Track spilled data locations
- Read spilled data back into memory
- Clean up temporary files

### VMRegion
Represents a virtual memory region backed by file pages. Provides a memory-mapped view of disk data with lazy loading. Enables efficient access to large data structures that don't fit in memory.

Key responsibilities:
- Map file regions to virtual memory
- Handle page faults transparently
- Support both read and write access
- Manage region lifecycle

## Files

- `buffer_manager.cpp`: Core buffer pool implementation with eviction queue
- `memory_manager.cpp`: Memory allocation tracking and limits
- `spiller.cpp`: Disk-based overflow handling
- `vm_region.cpp`: Virtual memory region management

## Eviction Policy

The buffer manager uses a second-chance CLOCK algorithm for page eviction:
1. Pages are enqueued to the eviction queue when first accessed
2. When memory pressure occurs, the eviction cursor scans for candidates
3. Recently accessed pages get a second chance before eviction
4. Page states are managed atomically to avoid race conditions
5. Dirty pages are flushed before eviction

## Concurrency Control

Page access uses optimistic concurrency control:
- Pages have atomic state tracking (unlocked, locked, marked, evicted)
- Version numbers prevent ABA problems during concurrent access
- Lock-free operations for common read paths
- Fine-grained locking only when necessary
