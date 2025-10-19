# Write-Ahead Log (WAL)

The WAL module implements write-ahead logging for durability and crash recovery. It ensures that all database modifications are logged before being applied to persistent storage, enabling the database to recover to a consistent state after a crash.

## Overview

This module provides transaction durability through logging all modifications to a sequential log file. The WAL follows the standard write-ahead logging protocol where log records must be flushed to disk before the corresponding data pages are modified.

## Key Components

### WAL
The main write-ahead log coordinator. Manages the global WAL file that records all committed transactions. Provides methods for logging records, checkpointing, and recovery.

Key responsibilities:
- Write transaction commit records
- Write checkpoint records
- Ensure proper log ordering and durability
- Support WAL replay during recovery
- Coordinate with local WAL for transaction logs

Operations:
- `logCommittedWAL()`: Write a transaction's log records to the shared WAL
- `logAndFlushCheckpoint()`: Record a checkpoint in the WAL
- `clear()`: Clear the WAL buffer and truncate the file
- `reset()`: Remove the WAL file entirely

### LocalWAL
Per-transaction write-ahead log that accumulates log records during transaction execution. Buffered in memory until transaction commit, then flushed to the shared WAL atomically.

Key responsibilities:
- Buffer log records for a single transaction
- Track transaction modifications
- Provide log records to shared WAL on commit
- Support transaction rollback by discarding logs

### WALRecord
Represents individual log entries describing database modifications. Different record types capture different operations like insertions, updates, deletions, and structural changes.

Record types:
- Commit records: Mark successful transaction completion
- Checkpoint records: Mark database checkpoint completion
- Table modification records: Track data changes
- Catalog modification records: Track schema changes

### WALReplayer
Handles recovery by replaying WAL records after a crash. Reads the WAL file sequentially and applies logged operations to restore the database to a consistent state.

Key responsibilities:
- Read WAL file sequentially
- Validate log record checksums
- Apply operations in log order
- Handle partial transactions (rollback)
- Reconstruct in-memory state after recovery

Operations:
- `replay()`: Apply all committed transactions from the WAL
- Validate log integrity with checksums
- Skip uncommitted transactions

### ChecksumReader/Writer
Handle checksummed I/O for the WAL file. Compute and validate checksums to detect log corruption and ensure durability guarantees.

Key features:
- CRC32 checksums for each log record
- Validation during recovery
- Detection of torn writes
- Configurable checksum enabling

## Files

- `wal.cpp`: Main WAL coordinator
- `local_wal.cpp`: Per-transaction WAL buffering
- `wal_record.cpp`: WAL record structures
- `wal_replayer.cpp`: Recovery and replay logic
- `checksum_reader.cpp`: Checksummed reading
- `checksum_writer.cpp`: Checksummed writing

## Write-Ahead Logging Protocol

The WAL follows these principles:

1. **Log Before Data**: All modifications must be logged before data pages are written
2. **Sequential Logging**: Log records written sequentially for efficient I/O
3. **Force on Commit**: Transaction commit record must be flushed to disk before commit returns
4. **Atomic Commits**: All log records for a transaction written atomically

## Recovery Process

Database recovery involves:

1. **Log Scanning**: Read WAL file from beginning
2. **Record Validation**: Verify checksums for each record
3. **Transaction Analysis**: Identify committed vs uncommitted transactions
4. **Redo Phase**: Apply all operations from committed transactions
5. **Undo Phase**: Roll back any incomplete transactions
6. **Cleanup**: Truncate or archive the WAL after successful recovery

## Checkpointing

Checkpointing reduces recovery time by:

1. **Flush**: Write all dirty pages to disk
2. **Log Checkpoint**: Write checkpoint record to WAL
3. **Truncate**: Old log records before checkpoint can be discarded
4. **Metadata**: Persist catalog and storage metadata

After a checkpoint, recovery only needs to replay WAL records after the checkpoint.

## Transaction Commit Flow

When a transaction commits:

1. Transaction generates log records in LocalWAL
2. LocalWAL records are written to shared WAL
3. WAL buffer is flushed and synced to disk
4. Commit returns to application
5. Modified pages written to disk asynchronously
6. Local storage merged into persistent storage

## Crash Recovery

If database crashes:

1. WAL file survives on disk
2. On restart, WAL is replayed
3. Committed transactions are reapplied
4. Uncommitted transactions are discarded
5. Database restored to last committed state

## Log Truncation

The WAL is truncated when:

1. Checkpoint completes successfully
2. All log records before checkpoint are no longer needed
3. WAL file is reset or truncated
4. Archived logs can be kept for point-in-time recovery
