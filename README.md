# RocksDB Compaction — Reverse Engineering & Analysis

**Systems Engineering Project**  
Reverse engineering the internal workings of RocksDB's compaction process.

This repository documents a deep dive into **RocksDB's LSM-tree compaction mechanism**. It includes a complete 6-stage demonstration, detailed code-level analysis, experimental insights, and a comprehensive report.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Project Goals](#project-goals)
- [Compaction Stages](#compaction-stages)
- [Complete Demo Script](#complete-demo-script)
- [Report & Findings](#report--findings)
- [Key Design Decisions](#key-design-decisions)
- [Experiments](#experiments)
- [How to Run](#how-to-run)
- [Repository Structure](#repository-structure)
- [References](#references)
- [License](#license)

---

## Overview

RocksDB uses a Log-Structured Merge-tree (LSM-tree) that relies heavily on **compaction** to maintain performance by reducing read amplification, removing tombstones, and controlling space amplification.

This project reverse-engineers the entire compaction pipeline — from trigger to cleanup — by analyzing source code, logs, MANIFEST files, and running controlled experiments.

**Inspired by** previous projects: **Faculty Finder** and **Context Seeker**, where efficient key-value storage and fast lookups are critical.

---

## Project Goals

- Understand the **6-stage compaction execution path** in detail.
- Analyze important design decisions and their trade-offs.
- Study key parameters: `kMinOverlappingRatio`, `max_bytes_for_level_base`, `target_file_size_base`, `compaction_pri`, etc.
- Identify common failure modes (write stalls, compaction storms, high write amplification).
- Provide practical tuning recommendations.

---

## Compaction Stages

The compaction process in RocksDB consists of **6 major stages**:

1. **Trigger** — When compaction is needed (L0 file count or level size ratio)
2. **Dispatch** — Scheduling compaction on background threads
3. **Picking** — Choosing which files to compact (score-based + file picking policy)
4. **Merge** — Actual merging of SST files (merge loop)
5. **Install** — Atomically installing new version via MANIFEST
6. **Cleanup** — Deleting old SST files

---

## Complete Demo Script



## Report & Findings

Detailed technical report covering:

- In-depth code walkthrough
- Design decisions with trade-offs
- Failure analysis (write stalls, compaction storms, space amplification, etc.)
- Key tuning parameters and their impact

**Full Report**: [`report/report.md`](report/report.md)

---

## Key Design Decisions

- Background compaction using dedicated threads
- Score-based level prioritization
- Immutable SST files
- Write Stalls as backpressure mechanism
- File picking policies (`kMinOverlappingRatio` vs others)

---

## Experiments

- Read amplification before vs after compaction
- Stale version cleanup
- Tombstone removal effectiveness
- Impact of compaction on write stalls

Results and flows are available in the [`report/`](report/) and [`experiments/`](experiments/) folders.

---

## How to Run

### Prerequisites
```bash
sudo apt-get install -y rocksdb-tools ldb
```


rocksdb-compaction/
├── demo/                  # Complete demo scripts
├── report/                # Detailed technical report
├── experiments/           # Experiment scripts and results
├── docs/                  # Additional documentation
├── slides/                # Presentation slides
├── README.md
└── LICENSE 

## References

### Official RocksDB Documentation

- [RocksDB Compaction](https://github.com/facebook/rocksdb/wiki/Compaction) — Official wiki explaining compaction styles, algorithms, and options.
- [Leveled Compaction](https://github.com/facebook/rocksdb/wiki/Leveled-Compaction) — Detailed explanation of leveled compaction, score calculation, and target sizes.
- [Universal Compaction](https://github.com/facebook/rocksdb/wiki/Universal-Compaction) — Documentation for Universal (Tiered) compaction style.
- [RocksDB Tuning Guide](https://github.com/facebook/rocksdb/wiki/RocksDB-Tuning-Guide) — Best practices for tuning compaction parameters including `max_bytes_for_level_base`, `target_file_size_base`, and `compaction_pri`.
- [RocksDB GitHub Repository](https://github.com/facebook/rocksdb) — Source code (especially `compaction_picker_level.cc`, `db_impl_compaction_flush.cc`, `version_storage_info.cc`).

### Academic Papers & Related Work

- Dostoevsky: Better Space-Time Trade-offs for LSM-tree based Key-Value Stores. Harvard University, 2017.
- Optimizing Space Amplification in RocksDB. CIDR 2017.
- Characterize LSM-tree Compaction Performance via On-the-fly Parameter Tuning (arXiv, 2026).
- Rethinking LSM-tree based Key-Value Stores: A Survey (arXiv, 2025).
- Low Tail Latency and I/O Amplification in LSM-based KV Stores (arXiv, 2024).

### Additional Resources

- Blog: "Name That Compaction Algorithm" by Mark Callaghan (smalldatum.blogspot.com).
- MSLS: A Low-Latency LSM-tree Implementation (FAST 2016).

---

**Last Updated**: April 2026  
**Note**: All RocksDB wiki links were accessed in April 2026. 

## License
This project is licensed under the MIT License.
