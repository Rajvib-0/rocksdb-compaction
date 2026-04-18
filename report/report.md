# RocksDB-Compaction

**202518048 — Rajvi**

---

![Logo]()

## Table of Contents
- [System Overview](#system-overview)
  - [What is RocksDB ?](#what-is-rocksdb-)
  - [What is Compaction ?](#what-is-compaction-)
  - [Why Compaction Exists ? (Motivation)](#why-compaction-exists--motivation)
- [Execution Path - Complete Trace](#execution-path---complete-trace)
- [Compaction Algorithm](#compaction-algorithm)
- [Design Decisions](#design-decisions)
- [Concept Mapping](#concept-mapping)
- [Experiments](#experiments)
- [Failure Analysis](#failure-analysis)
- [References](#references)

---

## System Overview

### What is RocksDB ?
RocksDB is an embedded key-value **storage engine** based on the Log-Structured Merge (LSM) tree design developed by Facebook, based on Google’s LevelDB. It is designed for fast writes and is widely used in production systems such as CockroachDB, MyRocks, and many other stateful services.  

It is optimized for high write throughput by converting random writes into **sequential** disk operations. Unlike traditional databases, it runs as a library inside applications. The tradeoff is that additional work (compaction) is required to maintain read performance and storage efficiency.  

![Flow](flow.png)

### What is Compaction ?
- **Definition:**  
  Compaction is a background process in RocksDB that merges multiple SST files into new sorted files at the next level. It reads data from existing files, combines them using a merge process, and writes optimized output files. This operation runs in the background without blocking writes.

- **What it does internally:**  
  During compaction, duplicate keys are resolved by keeping only the latest version, and deletion markers (tombstones) are removed when no longer needed. This ensures that data remains clean, sorted, and efficient for lookup operations across levels.

### Why Compaction Exists ? (Motivation)
- **Reduce read amplification**  
  Without compaction, many SST files (especially in L0) must be checked for every read, increasing latency. Compaction merges files and reduces the number of lookups needed.  

- **Remove stale and deleted data**  
  Multiple versions of the same key and deletion markers accumulate over time. Compaction removes outdated entries and frees disk space.  

- **Maintain structured storage**  
  As new data is continuously written, the system becomes fragmented. Compaction reorganizes data into sorted, non-overlapping levels, keeping the storage efficient and scalable.

---

## Execution Path - Complete Trace

---

## Compaction Algorithm

---

## Design Decisions

---

## Concept Mapping

---

## Experiments

---

## Failure Analysis

---

## References
