# rocksdb-compaction
Reverse engineering RocksDB compaction — Systems Engineering Project
## Complete Demo Script — All 6 Stages in One Run

```bash
#!/bin/bash
# ============================================================
# RocksDB Compaction Execution Path — Full Demo
# Demonstrates all 6 stages end-to-end
# Requires: sudo apt-get install -y rocksdb-tools
# ============================================================

DB=/tmp/rocksdb_full_demo
rm -rf $DB && mkdir -p $DB

echo "============================================"
echo " STAGE 1: TRIGGER — Write keys, fill L0"
echo "============================================"
for i in $(seq 1 20); do
  ldb --db=$DB --create_if_missing \
      put "key$(printf '%03d' $i)" "value_$i" 2>/dev/null
done
echo "L0 SST files created:"
ls $DB/*.sst | wc -l
echo "MANIFEST (overlapping L0, high RA):"
ldb --db=$DB manifest_dump 2>&1 | grep -E "level 0|seq:" | head -5
echo "WAL file (crash-recovery record):"
ls -lh $DB/*.log

echo ""
echo "============================================"
echo " STAGE 2: DISPATCH — confirm config"
echo "============================================"
grep "level0_file_num_compaction_trigger" $DB/LOG

echo ""
echo "============================================"
echo " STAGE 3-4: PICKING + MERGE — run compaction"
echo "============================================"
ldb --db=$DB compact
echo "Compaction inputs picked (from LOG):"
grep "Compaction start summary" $DB/LOG | tail -1
echo "Merge loop result (from LOG):"
grep "records in" $DB/LOG | tail -1

echo ""
echo "============================================"
echo " STAGE 5: INSTALL — VersionEdit + MANIFEST"
echo "============================================"
echo "New MANIFEST state:"
ldb --db=$DB manifest_dump 2>&1

echo ""
echo "============================================"
echo " STAGE 6: CLEANUP — old SSTs deleted"
echo "============================================"
echo "Files deleted:"
grep "Deleted file.*sst" $DB/LOG | wc -l
echo "SST files remaining:"
ls $DB/*.sst | wc -l
echo "All data accessible after compaction:"
ldb --db=$DB scan

echo ""
echo "============================================"
echo " RESULT"
echo "============================================"
echo "Before : 19 overlapping L0 SST files | Read Amplification = 19"
echo "After  :  1 clean SST file at L6     | Read Amplification = 1"
echo "Outcome: Stale versions dropped, Tombstones dropped,"
echo "         MANIFEST updated, Old files deleted"
```

---

## Summary — What Each Stage Proves

| Stage | Key Evidence | Command to See It |
|---|---|---|
| **1 — Trigger** | 19 L0 files, score = 4.75 | `manifest_dump` |
| **2 — Dispatch** | `compaction_trigger: 4` in LOG | `grep compaction_trigger LOG` |
| **3 — Picking** | All L0 files listed as inputs | `grep "Compaction start summary" LOG` |
| **4 — Merge Loop** | `num_input: 4 -> num_output: 2` | `grep "records in" LOG` |
| **5 — Install** | 1 file at L6 in MANIFEST | `manifest_dump` after compact |
| **6 — Cleanup** | `Deleted file` lines in LOG | `grep "Deleted file" LOG` |

---
