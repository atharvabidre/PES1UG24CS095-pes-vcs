# PES-VCS Lab Report
**Name:** Atharva Bidre  
**SRN:** PES1UG24CS095

---

## Phase 1: Object Storage

### Screenshots
- 1A: `./test_objects` — all tests passing
- 1B: `find .pes/objects -type f` — sharded directory structure

### Summary
Implemented `object_write` (stores data with header + SHA-256 hash, atomic write)
and `object_read` (reads, verifies hash integrity, returns data).

---

## Phase 2: Tree Objects

### Screenshots
- 2A: `./test_tree` — all tests passing
- 2B: `xxd` of raw tree object

### Summary
Implemented `tree_from_index` which builds a tree object from all staged
index entries and writes it to the object store.

---

## Phase 3: Staging Area

### Screenshots
- 3A: `pes init` → `pes add` → `pes status` output
- 3B: `cat .pes/index` showing text-format entries

### Summary
Implemented `index_load` (parses text index file), `index_save` (atomic
temp+rename write with qsort), and `index_add` (stages file as blob).

---

## Phase 4: Commits and History

### Screenshots
- 4A: `pes log` showing three commits
- 4B: `find .pes -type f | sort` showing object growth
- 4C: `cat .pes/refs/heads/main` and `cat .pes/HEAD`
- Final: `make test-integration` — all tests passed

### Summary
Implemented `commit_create` which builds a tree from the index, reads the
parent commit from HEAD, serializes the commit object, writes it, and
updates HEAD atomically.

---

## Phase 5: Branching Analysis

**Q5.1:** To implement `pes checkout <branch>`, three things must happen:
update `.pes/HEAD` to contain `ref: refs/heads/<branch>`, read the target
branch's commit hash, walk its tree object recursively, and overwrite all
working directory files to match. The complexity comes from handling files
that exist in one branch but not the other — they must be deleted or created.

**Q5.2:** To detect a dirty working directory, compare each file tracked in
the index against the working directory using `stat()`. If `mtime` or `size`
differs from what is stored in the index entry, the file has been modified
and checkout should refuse to proceed to avoid losing changes.

**Q5.3:** In detached HEAD state, HEAD contains a raw commit hash instead of
a branch reference. New commits are made but no branch pointer moves forward
to track them. If you switch to another branch, those commits become
unreachable by normal means. To recover them, you need the commit hash
(from terminal history or `pes log` output taken before switching) and then
create a new branch pointing to it: `git branch recovered <hash>`.

---

## Phase 6: Garbage Collection Analysis

**Q6.1:** The algorithm is a mark-and-sweep. Start from all branch refs,
walk every reachable commit, and for each commit walk its tree recursively,
collecting every blob, tree, and commit hash into a hash set. Then scan all
files under `.pes/objects/` and delete any whose hash is not in the set.
A hash set gives O(1) lookup. For 100,000 commits with 50 branches, assuming
~10 objects per commit on average, you would need to visit around 1,000,000
objects during the mark phase.

**Q6.2:** The race condition: GC marks an object as unreachable and deletes
it, but at the same moment a commit operation has computed that object's hash
and is about to reference it in a new tree — the object is now gone before
the commit finishes writing. Git avoids this by never deleting objects newer
than a configurable grace period (default 2 weeks), so any in-progress
operations have time to complete before GC can touch new objects.
