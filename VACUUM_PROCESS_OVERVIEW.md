# Overview of PostgreSQL VACUUM

This document provides a beginner-friendly introduction to how the VACUUM process is organized inside PostgreSQL.  The goal is not to replace the official documentation, but to help new contributors navigate the source tree and understand where the main pieces live.

## Entry point

The SQL command `VACUUM` is parsed and eventually handled by `ExecVacuum` in `src/backend/commands/vacuum.c`.

```c
void
ExecVacuum(ParseState *pstate, VacuumStmt *vacstmt, bool isTopLevel)
```
The function parses command options and prepares a `VacuumParams` structure before calling the generic `vacuum()` routine【F:src/backend/commands/vacuum.c†L159-L221】【F:src/backend/commands/vacuum.c†L380-L462】.

## Common vacuum logic

Function `vacuum()` orchestrates vacuuming one or more relations. It can run ordinary VACUUM, ANALYZE or both. The function acquires locks, determines whether to start separate transactions for each table and calls `vacuum_rel()` for each relation【F:src/backend/commands/vacuum.c†L540-L704】.

## Heap vacuum implementation

For heap tables the heavy lifting is implemented in `src/backend/access/heap/vacuumlazy.c`.  A large comment at the top of the file describes the three major phases of vacuum:

1. **Phase I** – scan table pages, prune and freeze tuples, and collect dead tuple IDs.
2. **Phase II** – remove dead entries from indexes.
3. **Phase III** – revisit pages to mark dead tuples as unused.

These phases are summarized in the source comments starting at line 1 of that file【F:src/backend/access/heap/vacuumlazy.c†L1-L35】 and lines describing the scanning process continue further【F:src/backend/access/heap/vacuumlazy.c†L38-L79】.

Function `heap_vacuum_rel()` sets up a `LVRelState` structure and then calls `lazy_scan_heap()` which performs most of the work【F:src/backend/access/heap/vacuumlazy.c†L604-L707】【F:src/backend/access/heap/vacuumlazy.c†L1130-L1175】.

### lazy_scan_heap

`lazy_scan_heap()` iterates over the table, pruning tuples and collecting dead tuple identifiers.  When the internal storage for dead TIDs fills up, it vacuums indexes and removes heap tuples in batches.  The function also updates the visibility map and may truncate the relation at the end【F:src/backend/access/heap/vacuumlazy.c†L1164-L1225】.

## Parallel vacuum

PostgreSQL can perform index cleanup in parallel.  The helper functions for this live in `src/backend/commands/vacuumparallel.c`.  The file header explains how `ParallelVacuumState` is set up and used during index bulk deletion and cleanup【F:src/backend/commands/vacuumparallel.c†L1-L26】.

## Autovacuum

The background autovacuum daemon automatically launches vacuum workers.  `src/backend/postmaster/autovacuum.c` contains the launcher and worker logic.  Its introductory comment describes how the launcher schedules worker processes to connect to databases and run vacuum tasks【F:src/backend/postmaster/autovacuum.c†L1-L20】.

## Further reading

The official PostgreSQL documentation provides detailed information about VACUUM in the user manual.  See `doc/src/sgml/maintenance.sgml` for the chapter discussing routine database maintenance.

