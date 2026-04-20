# PES-VCS Lab Report
**Name:** Atharva Bidre  
**SRN:** PES1UG24CS095

---

## Phase 1 — Object Storage

### Screenshot 1A — test_objects passing
![test_objects output](screenshots/phase1A.png)

### Screenshot 1B — Sharded object store
![find .pes/objects](screenshots/phase1B.png)

---

## Phase 2 — Tree Objects

### Screenshot 2A — test_tree passing
![test_tree output](screenshots/phase2A.png)

### Screenshot 2B — xxd of tree object
![xxd tree object](screenshots/phase2B.png)

---

## Phase 3 — Index / Staging Area

### Screenshot 3A — init, add, status
![pes status](screenshots/phase3A.png)

### Screenshot 3B — cat .pes/index
![index file](screenshots/phase3B.png)

---

## Phase 4 — Commits and History

### Screenshot 4A — pes log
![pes log](screenshots/phase4A.png)

### Screenshot 4B — find .pes -type f
![object growth](screenshots/phase4B.png)

### Screenshot 4C — HEAD and refs
![HEAD and refs](screenshots/phase4C.png)

---

## Final Integration Test
![integration test](screenshots/final.png)

---

## Phase 5 — Branching Analysis

### Q5.1
What files change in .pes/: HEAD is the only file that needs to change. You update it to point to the new branch: ref: refs/heads/<new-branch>

What must happen to the working directory:
1. Read the current HEAD → get current commit hash → get current tree
2. Read the target branch → get target commit hash → get target tree
3. Compare the two trees — find files that are different between them
4. For each file that differs: if the file exists in the target tree → write the target version to disk. If the file doesn't exist in the target tree → delete it from disk.
5. Update the index to match the target tree exactly
6. Update HEAD to point to the new branch

What makes it complex: You have to update both the working directory files and the index atomically — if it crashes halfway, the repo is in a broken state. You must handle subdirectories — creating and deleting entire folder trees if needed. If a file was added in the working directory but doesn't exist in either branch (untracked), you must leave it alone. If the file exists in both branches but has different content, and the user has local edits, this becomes a conflict.

### Q5.2
When switching branches, you need to check if any file would be overwritten with local unsaved changes, using only the index and object store:

1. For each file in the current index, compute the hash of the file currently on disk (same way pes add would)
2. Compare that hash to what the index says the hash should be
3. If they differ → the file is locally modified (dirty)
4. Now check: does this dirty file also differ between the two branches? If yes → refuse checkout and print an error. If no → the file is the same on both branches, so the local change is safe and checkout can proceed.

Why this works: The index always stores the hash of the last staged/committed version. Comparing the on-disk hash to the index hash is exactly how pes status detects unstaged changes.

### Q5.3
In detached HEAD state, HEAD contains a raw commit hash instead of a branch reference. New commits are created and chained correctly but since HEAD points directly to a hash and not a branch, no branch file gets updated. The commits exist in the object store but nothing points to them by name. When you checkout another branch, those commits become unreachable and will eventually be deleted by garbage collection.

To recover: create a branch pointing to those commits before leaving detached HEAD state:
- pes branch recovery-branch (create branch at current HEAD)
- pes checkout recovery-branch (now HEAD points to branch, commits are safe)

If you already left and lost the hash, you would have to search through all objects in .pes/objects/ manually — which is why Git has a reflog. PES-VCS does not implement reflog, so losing commits in detached HEAD is permanent once GC runs.

---

## Phase 6 — Garbage Collection Analysis

### Q6.1
The algorithm is mark-and-sweep:

Phase 1 MARK: Start from all branch refs, walk every reachable commit, and for each commit walk its tree recursively, collecting every blob, tree, and commit hash into a hash set.

Phase 2 SWEEP: Scan all files under .pes/objects/ and delete any whose hash is not in the reachable set.

Data structure: A hash set gives O(1) lookup for checking reachability.

Estimate for 100,000 commits and 50 branches: Each commit has roughly 1 commit object + 1 root tree + ~10 blob/tree objects = ~12 objects per commit. Total = 100,000 x 12 = ~1.2 million objects to visit during the mark phase. With O(1) hash set lookups this runs in seconds.

### Q6.2
The race condition: GC marks all currently reachable objects. Then a commit operation writes a new blob to the object store — it exists on disk but nothing points to it yet. GC then sweeps and deletes it since it was not in the reachable set. The commit then writes a tree referencing the now-deleted blob, and updates HEAD — the repo is now corrupt.

How Git avoids this:
1. Grace period: Git never deletes objects newer than 2 weeks by default, making the race essentially impossible since commits take milliseconds.
2. Lock files: Git uses index.lock files to signal a write is in progress. GC checks for these and refuses to run.
3. Write ordering: Git always writes bottom-up (blobs first, then trees, then commits) and updates the branch ref last. GC starts from refs, so new objects are temporarily unreachable but protected by the grace period.
