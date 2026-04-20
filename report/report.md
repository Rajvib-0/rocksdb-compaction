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

# System Overview

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

# Compaction Execution Path — Complete Trace

> **Tool used:** `ldb` — RocksDB's official command-line admin tool  
> **Install:** `sudo apt-get install rocksdb-tools`  
> **All commands below are copy-paste runnable in any bash terminal**

---

## Overview — The Six Stages of One Compaction Cycle

```
STAGE 1      STAGE 2        STAGE 3       STAGE 4          STAGE 5       STAGE 6
─────────    ──────────     ──────────    ─────────────    ──────────    ──────────
TRIGGER  →  DISPATCH   →   PICKING   →  MERGE LOOP   →   INSTALL   →   CLEANUP
What fires   Thread pool    Score all     K-way merge,     VersionEdit   Old SSTs
compaction   scheduling     levels,pick   key decisions,   + MANIFEST    deleted
             + gating       files         emit output      write         from disk
```

---

## Stage 1 — Trigger

**What happens:** Every flush completion, compaction completion, or manual call  
checks whether compaction is now needed via `MaybeScheduleFlushOrCompaction()`.

### Step 1.1 — Setup: Install ldb and create the database

```bash
# Install RocksDB tools (Ubuntu/Debian)
sudo apt-get install -y rocksdb-tools

# Verify
ldb --version
```

**Expected output:**
```
ldb from RocksDB 8.9.1
```

---

### Step 1.2 — Write keys, observe L0 files accumulate

```bash
# Clean slate
rm -rf /tmp/rocksdb_demo && mkdir -p /tmp/rocksdb_demo

# Write 20 keys — each write goes WAL → MemTable → L0 SST
for i in $(seq 1 20); do
  ldb --db=/tmp/rocksdb_demo \
      --create_if_missing \
      put "key$(printf '%03d' $i)" "value_$i"
done
```

**Expected output:** `OK` printed 20 times — one per write.

---

### Step 1.3 — Observe: Multiple L0 SST files with OVERLAPPING key ranges

```bash
# Show MANIFEST — the ground truth of which files exist at which level
ldb --db=/tmp/rocksdb_demo manifest_dump
```

**Expected output:**
```
--------------- Column family "default"  (ID 0) --------------
log number: 105
comparator: leveldb.BytewiseComparator
--- level 0 --- version# 1 ---
 103:1021[18 .. 18]['key018' seq:18, type:1 .. 'key018' seq:18, type:1]
 108:1021[19 .. 19]['key019' seq:19, type:1 .. 'key019' seq:19, type:1]
 8:1020[1 .. 1]['key001' seq:1, type:1 .. 'key001' seq:1, type:1]
 13:1020[2 .. 2]['key002' seq:2, type:1 .. 'key002' seq:2, type:1]
 ...
--- level 1 --- version# 1 ---
--- level 2 --- version# 1 ---
...
```

> **What this proves:**
> Each `put` flushed directly to a separate L0 SST file.
> All 19 L0 files have overlapping key ranges (each covers a single key in this small demo).
> In production: L0 files from real MemTable flushes cover wide, overlapping ranges.
> This is the accumulation problem that makes every `Get()` check all L0 files.

---

### Step 1.4 — Observe: WAL file present (durability before MemTable flush)

```bash
# WAL file — every write lands here first, before MemTable
ls -lh /tmp/rocksdb_demo/*.log
```

**Expected output:**
```
-rw-r--r-- 1 root root 36 Apr 20 02:45 /tmp/rocksdb_demo/000109.log
```

> **What this proves:**
> The WAL exists and is non-empty. It will be deleted only after the corresponding
> MemTable is flushed to an SST file. Until then, it is the crash-recovery record.

---

### Step 1.5 — Observe: Count SST files before compaction

```bash
ls /tmp/rocksdb_demo/*.sst | wc -l
ls -lh /tmp/rocksdb_demo/*.sst
```

**Expected output:**
```
19
-rw-r--r-- 1 root root 1020 ... 000008.sst
-rw-r--r-- 1 root root 1020 ... 000013.sst
...
-rw-r--r-- 1 root root 1021 ... 000108.sst
```

> **What this proves:**
> 19 separate SST files exist — one per write in this demo.
> A `Get()` right now must check all 19 files in L0 (Read Amplification = 19).
> This is exactly the state that triggers compaction.

---

## Stage 2 — Dispatch

**What happens:** `MaybeScheduleFlushOrCompaction()` evaluates three gate conditions,  
then submits the work to the background low-priority thread pool via `Env::Schedule()`.  
The calling thread returns immediately — compaction is fully asynchronous.

### Step 2.1 — Confirm compaction config from LOG

```bash
# RocksDB logs all options at DB open time — show compaction-related ones
grep -E "compaction_trigger|slowdown|stop_writes|multiplier|level_base" \
     /tmp/rocksdb_demo/LOG | head -10
```

**Expected output:**
```
Options.level0_file_num_compaction_trigger: 4
Options.level0_slowdown_writes_trigger: 20
Options.level0_stop_writes_trigger: 36
Options.max_bytes_for_level_base: 268435456
Options.max_bytes_for_level_multiplier: 10.000000
```

> **What this proves:**
> `level0_file_num_compaction_trigger = 4` means once L0 has >= 4 files,
> `MaybeScheduleFlushOrCompaction()` will dispatch a compaction task.
> Our 19 L0 files far exceed this — compaction is long overdue.
> `slowdown_writes_trigger = 20` and `stop_writes_trigger = 36` are the write stall thresholds.

---

### Step 2.2 — Observe the thread pool and background job config

```bash
grep -E "max_background|max_subcompaction" \
     /tmp/rocksdb_demo/LOG | head -5
```

**Expected output:**
```
Options.max_background_compactions: -1
Options.max_subcompactions: 1
Options.max_background_flushes: -1
```

> **What this proves:**
> `-1` means RocksDB auto-tunes thread count.
> `max_subcompactions = 1` means no parallel subcompaction by default.
> A separate flush thread pool exists so flush jobs cannot block compaction jobs.

---

## Stage 3 — Picking

**What happens:** `LevelCompactionPicker::PickCompaction()` scores every level,  
selects the highest-score level, picks specific files using a round-robin pointer,  
expands to a "clean cut" boundary, and finds overlapping files in the output level.

### Step 3.1 — Trigger compaction and observe picking in LOG

```bash
# Trigger manual compaction — same internal path as auto compaction
ldb --db=/tmp/rocksdb_demo compact
```

**Expected output:** *(no stdout — compaction runs silently in background)*

---

### Step 3.2 — Read the LOG to see the picking decision

```bash
grep -E "Compacting|compaction_started|Compaction start summary|Manual compaction" \
     /tmp/rocksdb_demo/LOG
```

**Expected output:**
```
[db_impl_compaction_flush.cc:3564] Manual compaction from level-0 to level-6
[compaction_job.cc:2079] [JOB 4] Compacting 30@6 files to L6, score -1.00
[compaction_job.cc:2085] Compaction start summary:
  inputs: [8(1022B) 13(1022B) 18(1022B) 23(1022B) 30(1022B) ...]
EVENT_LOG_v1 {"event": "compaction_started",
  "compaction_reason": "ManualCompaction",
  "files_L6": [8, 13, 18, ...],
  "input_data_size": 30689}
```

> **What this proves:**
> The picker selected ALL input files (since all L0 files overlap each other,
> all must be included — clean-cut rule).
> `input_data_size: 30689` bytes is the total data the merge loop will read.
> `score: -1` = manual compaction (bypasses the scoring system).

---

### Step 3.3 — Understand level scoring with a concrete example

```bash
grep -A 5 "Compaction Stats" /tmp/rocksdb_demo/LOG | head -8
```

**Expected output:**
```
** Compaction Stats [default] **
  L0    1/1  1.00 KB  0.0   0.0  0.0  0.0  0.0  0.0  0.0  1.0 ...
```

> **Reading the score column:**
> ```
> For L0:  score = file_count / level0_file_num_compaction_trigger
>                = 19 / 4 = 4.75  <- far above 1.0, compaction urgent
>
> For L1+: score = actual_level_size / target_level_size
>                = 0 / 256MB = 0.0  <- empty, no compaction needed
> ```
> Highest score wins. L0 score of 4.75 -> L0 is chosen as the compaction base level.

---

## Stage 4 — The Merge Loop

**What happens:** `CompactionJob::Run()` opens iterators over all input SST files,  
builds a MergingIterator (k-way min-heap), and calls `ProcessKeyValueCompaction()`  
which applies the key decision logic: drop stale versions, drop resolved tombstones, emit clean keys.

### Step 4.1 — Stale Version Demo: multiple writes to same key

```bash
rm -rf /tmp/rocksdb_versions && mkdir -p /tmp/rocksdb_versions

# Write 3 versions of the same key "apple"
ldb --db=/tmp/rocksdb_versions --create_if_missing put apple "red"
ldb --db=/tmp/rocksdb_versions put apple "green"
ldb --db=/tmp/rocksdb_versions put apple "yellow"

echo "=== BEFORE COMPACTION: 2 SST files, multiple versions of apple ==="
ldb --db=/tmp/rocksdb_versions manifest_dump
```

**Expected output:**
```
--- level 0 --- version# 1 ---
 13:1016[2 .. 2]['apple' seq:2, type:1 .. 'apple' seq:2, type:1]
 8:1014[1 .. 1]['apple' seq:1, type:1 .. 'apple' seq:1, type:1]
```

> **What this shows:**
> Two separate L0 SST files both contain "apple" at different sequence numbers.
> seq:2 = "yellow" (most recent), seq:1 = "green", seq:0 = "red" (oldest).
> All versions are on disk occupying space — stale version accumulation.

---

### Step 4.2 — Confirm only latest version visible to user

```bash
ldb --db=/tmp/rocksdb_versions get apple
```

**Expected output:**
```
yellow
```

---

### Step 4.3 — Run compaction: merge loop drops stale versions

```bash
echo "=== SST files BEFORE: ==="
ls -lh /tmp/rocksdb_versions/*.sst

ldb --db=/tmp/rocksdb_versions compact

echo "=== SST files AFTER: ==="
ls -lh /tmp/rocksdb_versions/*.sst

echo "=== MANIFEST AFTER: all versions merged to 1 clean SST ==="
ldb --db=/tmp/rocksdb_versions manifest_dump
```

**Expected output:**
```
=== SST files BEFORE: ===
-rw-r--r-- 1 root root 1014 ... 000008.sst
-rw-r--r-- 1 root root 1016 ... 000013.sst

=== SST files AFTER: ===
-rw-r--r-- 1 root root 1040 ... 000023.sst

=== MANIFEST AFTER: ===
--- level 6 ---
 23:1040[0 .. 0]['apple' seq:0, type:1 .. 'apple' seq:0, type:1]
```

> **What this proves — the merge loop in action:**
> Input:  2 SST files containing 3 InternalKeys for "apple" (seq=2, seq=1, seq=0)
> MergingIterator yields them: seq=2 first (most recent = smallest in heap by seqno desc)
> Decision for seq=2: first version seen -> EMIT to output SST
> Decision for seq=1: duplicate user_key, no live snapshot -> DROP
> Decision for seq=0: duplicate user_key, no live snapshot -> DROP
> Output: 1 SST file containing only "apple" at seq:0 (cleaned, final version)

---

### Step 4.4 — Value preserved correctly after compaction

```bash
ldb --db=/tmp/rocksdb_versions get apple
```

**Expected output:**
```
yellow
```

---

### Step 4.5 — Tombstone Demo: Delete creates a marker, compaction removes it

```bash
rm -rf /tmp/rocksdb_tomb && mkdir -p /tmp/rocksdb_tomb

ldb --db=/tmp/rocksdb_tomb --create_if_missing put key001 "hello"
ldb --db=/tmp/rocksdb_tomb put key002 "world"
ldb --db=/tmp/rocksdb_tomb put key003 "foo"

echo "=== BEFORE DELETE ==="
ldb --db=/tmp/rocksdb_tomb scan
```

**Expected output:**
```
key001 ==> hello
key002 ==> world
key003 ==> foo
```

---

```bash
# Delete key002 — writes a kTypeDeletion InternalKey (tombstone), NOT a real removal
ldb --db=/tmp/rocksdb_tomb delete key002

echo "=== AFTER DELETE: key002 gone from user view, tombstone on disk ==="
ldb --db=/tmp/rocksdb_tomb scan
```

**Expected output:**
```
key001 ==> hello
key003 ==> foo
```

> **What this shows:**
> The delete wrote a kTypeDeletion InternalKey to MemTable and WAL.
> The old "world" value still occupies space on disk in its original SST.
> The tombstone record also occupies space. Nothing is freed yet.

---

```bash
echo "=== SST count before compaction (tombstone + original both on disk) ==="
ls /tmp/rocksdb_tomb/*.sst | wc -l

ldb --db=/tmp/rocksdb_tomb compact

echo "=== SST count after compaction (tombstone dropped, space reclaimed) ==="
ls /tmp/rocksdb_tomb/*.sst | wc -l

echo "=== Data still correct ==="
ldb --db=/tmp/rocksdb_tomb scan
```

**Expected output:**
```
=== SST count before: ===
4

=== SST count after: ===
1

=== Data still correct: ===
key001 ==> hello
key003 ==> foo
```

> **What this proves — tombstone handling:**
> Input: 4 SST files with key001, key002(value), key002(tombstone), key003
> MergingIterator yields in sorted order by user_key then seqno
> Decision for key002 tombstone: at bottommost level, no older version below -> DROP
> Decision for key002 value:     shadowed by tombstone, older seqno -> DROP
> Output: 1 clean SST with only key001 and key003
> 4 files -> 1 file. Space reclaimed. Tombstone physically gone.

---

### Step 4.6 — Read the merge loop result from LOG

```bash
grep -E "Generated table|records in|compaction_finished" \
     /tmp/rocksdb_tomb/LOG | tail -5
```

**Expected output:**
```
[compaction_job.cc:1655] Generated table #28: 2 keys, 1050 bytes
[compaction_job.cc:1729] Compacted 4@6 files to L6 => 1050 bytes
EVENT_LOG_v1 {"event": "compaction_finished",
  "num_input_records": 4,
  "num_output_records": 2,
  "output_compression": "Snappy",
  "lsm_state": [0, 0, 0, 0, 0, 0, 1]}
```

> **Reading the LOG:**
> `num_input_records: 4`  = merge loop read 4 InternalKeys
> `num_output_records: 2` = only key001 and key003 emitted (key002 both records dropped)
> `lsm_state: [0,0,0,0,0,0,1]` = 1 file in L6, all other levels empty
> `output_compression: Snappy` = output SST compressed at bottommost level

---

## Stage 5 — Install (VersionEdit + MANIFEST)

**What happens:** `CompactionJob::Install()` creates a `VersionEdit` recording which  
files were removed and which were added, writes it atomically to MANIFEST, and installs  
a new `Version` as current. This is the crash-safe commit point.

### Step 5.1 — Inspect the MANIFEST after compaction

```bash
ldb --db=/tmp/rocksdb_tomb manifest_dump
```

**Expected output:**
```
--------------- Column family "default"  (ID 0) --------------
log number: 20
comparator: leveldb.BytewiseComparator
--- level 6 --- version# 1 ---
 28:1050[0 .. 0]['key001' seq:0, type:1 .. 'key003' seq:0, type:1]
next_file_number 30  last_sequence 4  prev_log_number 0
```

> **What this shows:**
> MANIFEST records the final state: 1 file (#28) at L6 covering key001->key003.
> `next_file_number: 30` = next SST to be created gets number 30.
> `last_sequence: 4` = 4 operations total (3 puts + 1 delete).
> On crash recovery, RocksDB replays all MANIFEST records to reconstruct this state.

---

### Step 5.2 — Observe Version install in LOG

```bash
grep -E "compacted to:|lsm_state|Recovering from manifest" \
     /tmp/rocksdb_tomb/LOG | tail -5
```

**Expected output:**
```
[version_set.cc:5936] Recovering from manifest file: MANIFEST-000020
[compaction_job.cc:908] compacted to: base level 6
  files[0 0 0 0 0 0 1] max score 0.00
  files in(0, 4) out(1)
  records in: 4, records dropped: 0 output_compression: Snappy
EVENT_LOG_v1 {"event": "compaction_finished", "lsm_state": [0,0,0,0,0,0,1]}
```

> **Reading `files in(0, 4) out(1)`:**
> `0` files from input level (L0 was empty at this stage)
> `4` files from output level (L6 had 4 old files)
> `1` new output file written — the single merged, cleaned SST

---

## Stage 6 — Cleanup (Physical File Deletion)

**What happens:** `DeleteObsoleteFiles()` checks which SST files on disk are not  
referenced by any live `Version`. Files with zero reader references are deleted immediately.

### Step 6.1 — Observe file deletion in LOG

```bash
grep "Deleted file" /tmp/rocksdb_demo/LOG | head -8
```

**Expected output:**
```
[delete_scheduler.cc:73] Deleted file /tmp/rocksdb_demo/000008.sst immediately
[delete_scheduler.cc:73] Deleted file /tmp/rocksdb_demo/000013.sst immediately
[delete_scheduler.cc:73] Deleted file /tmp/rocksdb_demo/000018.sst immediately
[delete_scheduler.cc:73] Deleted file /tmp/rocksdb_demo/000023.sst immediately
...
```

> **What this proves:**
> `immediately` = no reader held a reference to the old Version when cleanup ran.
> If any iterator or snapshot was open, deletion would be deferred until
> that reader released its reference via `Version::Unref()`.

---

### Step 6.2 — Confirm old files gone, only new SST remains

```bash
echo "=== Files on disk after full compaction cycle ==="
ls -lh /tmp/rocksdb_demo/

echo ""
echo "=== SST count: before=19 files, after=1 file ==="
ls /tmp/rocksdb_demo/*.sst | wc -l
```

**Expected output:**
```
=== Files on disk after full compaction cycle ===
-rw-r--r-- 1 root root 1.3K ... 000118.sst    <- 1 clean SST (was 19)
-rw-r--r-- 1 root root  32  ... 000119.log
-rw-r--r-- 1 root root  16  ... CURRENT
-rw-r--r-- 1 root root  25K ... LOG
-rw-r--r-- 1 root root  356 ... MANIFEST-000120

=== SST count: ===
1
```

> **What this proves — the full cycle result:**
> 19 L0 SST files (Read Amplification = 19) -> 1 L6 SST file (Read Amplification = 1)
> All stale versions and tombstones resolved and dropped
> MANIFEST updated atomically — crash-safe
> All old input files physically deleted from disk

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
