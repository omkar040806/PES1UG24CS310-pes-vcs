# PES-VCS — Version Control System Implementation Report

**Name:** Omkar  
**SRN:** PES1UG24CS310  
**Repository:** PES1UG24CS310-pes-vcs

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Setup and Prerequisites](#setup-and-prerequisites)
3. [Phase 1: Object Storage](#phase-1-object-storage)
4. [Phase 2: Tree Objects](#phase-2-tree-objects)
5. [Phase 3: Index / Staging Area](#phase-3-index--staging-area)
6. [Phase 4: Commits and History](#phase-4-commits-and-history)
7. [Phase 5: Branching Analysis](#phase-5-branching-analysis)
8. [Phase 6: Garbage Collection Analysis](#phase-6-garbage-collection-analysis)
9. [Screenshots](#screenshots)
10. [Integration Test](#integration-test)

---

## Project Overview

PES-VCS is a local version control system built from scratch in C, modeled after Git's internal design. It implements content-addressable object storage, a staging area (index), tree objects for directory snapshots, and a linked commit history.

The system supports five commands:

```
pes init              Create .pes/ repository structure
pes add <file>...     Stage files (hash + update index)
pes status            Show modified/staged/untracked files
pes commit -m <msg>   Create commit from staged files
pes log               Walk and display commit history
```

The `.pes/` directory structure mirrors Git's `.git/`:

```
.pes/
├── objects/          # Content-addressable blob/tree/commit storage
│   ├── 2f/
│   │   └── 8a3b...   # Sharded by first 2 hex chars of hash
│   └── a1/
│       └── 9c4e...
├── refs/
│   └── heads/
│       └── main      # Branch pointer (file containing commit hash)
├── index             # Staging area (text file)
└── HEAD              # Current branch reference
```

---

## Setup and Prerequisites

```bash
sudo apt update && sudo apt install -y gcc build-essential libssl-dev
```

### Building

```bash
make          # Build the pes binary
make all      # Build pes + test binaries
make clean    # Remove all build artifacts
```

### Author Configuration

```bash
export PES_AUTHOR="Your Name <PES1UG24CS310>"
```

---

## Phase 1: Object Storage

**File:** `object.c`  
**Functions implemented:** `object_write`, `object_read`

### Concept

Every piece of data (file contents, directory listings, commits) is stored as an object named by its SHA-256 hash. Objects are stored under `.pes/objects/XX/YYYYYY...` where `XX` is the first two hex characters (directory sharding).

Object format on disk:
```
"<type> <size>\0<data>"
```
Example for a blob:
```
"blob 6\0hello\n"
```

### Implementation Details

**`object_write`:**
1. Builds a header string: `"blob 16\0"` / `"tree 16\0"` / `"commit 16\0"`
2. Combines header + data into one buffer and computes SHA-256
3. Checks for deduplication — if hash already exists, skips writing
4. Creates shard directory (`.pes/objects/XX/`) using `mkdir`
5. Writes to a temp file using `mkstemp`
6. Calls `fsync()` to flush to disk
7. Atomically renames temp file to final path
8. Returns computed hash in `*id_out`

**`object_read`:**
1. Builds file path from hash using `object_path()`
2. Reads entire file into memory
3. Finds the `\0` separator using `memchr`
4. Recomputes SHA-256 and compares to expected hash for integrity verification
5. Parses type string (`blob`, `tree`, `commit`)
6. Returns data portion after the `\0`

### Testing

```bash
make test_objects
./test_objects
```

**Output:**
```
Stored blob with hash: d58213f5dbe0629b5c2fa28e5c7d4213ea09227ed0221bbe9db5e5c4b9aafc12
Object stored at: .pes/objects/d5/8213f5dbe0629b5c2fa28e5c7d4213ea09227ed0221bbe9db5e5c4b9aafc12
PASS: blob storage
PASS: deduplication
PASS: integrity check
All Phase 1 tests passed.
```

```bash
find .pes/objects -type f
```

**Output:**
```
.pes/objects/d5/8213f5dbe0629b5c2fa28e5c7d4213ea09227ed0221bbe9db5e5c4b9aafc12
.pes/objects/2a/594d39232787fba8eb7287418aec99c8fc2ecdaf5aaf2e650eda471e566fcf
.pes/objects/25/ef1fa07ea68a52f800dc80756ee6b7ae34b337afedb9b46a1af8e11ec4f476
```

---

## Phase 2: Tree Objects

**File:** `tree.c`  
**Functions implemented:** `tree_from_index` (+ internal helper `write_tree_level`)

### Concept

A tree object represents a directory snapshot. Each entry maps a name to either a blob (file) or another tree (subdirectory), along with the file mode.

Binary tree format (per entry):
```
"<mode-octal> <name>\0<32-byte-binary-hash>"
```

Example:
```
100644 README.md\0<32 bytes>
040000 src\0<32 bytes>
```

### Implementation Details

**`write_tree_level` (helper):**
- Recursively processes index entries grouped by their path prefix at a given depth
- For flat files at the current level: adds them directly as tree entries
- For subdirectories: gathers all entries with the same prefix, recurses to build a subtree, adds the subtree hash as a directory entry with mode `040000`

**`tree_from_index`:**
1. Loads the index using `index_load()`
2. Sorts entries by path to group subdirectory entries together
3. Calls `write_tree_level` recursively to build the full tree hierarchy
4. Serializes each tree level using `tree_serialize()` and writes to object store

### Key Fix

The `test_tree` Makefile target doesn't link `index.o`, so we build manually:

```bash
gcc -Wall -Wextra -O2 -c index.c -o index.o
gcc -o test_tree test_tree.o object.o tree.o index.o -lcrypto
./test_tree
```

### Testing

```bash
./test_tree
```

**Output:**
```
Serialized tree: 139 bytes
PASS: tree serialize/parse roundtrip
PASS: tree deterministic serialization
All Phase 2 tests passed.
```

**Tree object raw binary (xxd):**
```bash
xxd .pes/objects/09/b576c2eb51b78fcf9b8493a0909522a75f8ee86df0e926312deeb5fbe7dfb1 | head -20
```
```
00000000: 7472 6565 2034 3900 3130 3036 3434 2066  tree 49.100644 f
00000010: 696c 6531 2e74 7874 002c f8d8 3d9e e295  ile1.txt.,..=...
00000020: 43b3 4a87 7274 21fd ecb7 e3f3 a183 d337  C.J.rt!........7
00000030: 6390 25de 576d b9eb b4                   c.%.Wm...
```

The binary format is clearly visible: `tree 49` header, `100644 file1.txt` entry name, followed by raw 32-byte hash.

---

## Phase 3: Index / Staging Area

**File:** `index.c`  
**Functions implemented:** `index_load`, `index_save`, `index_add`

### Concept

The index is a text file (`.pes/index`) that tracks staged files. Format:
```
<mode-octal> <64-char-hex-hash> <mtime-seconds> <size> <path>
```

Example:
```
100644 2cf8d83d9ee29543... 1699900000 6 file1.txt
100644 e00c50e16a2df38f... 1699900100 6 file2.txt
```

### Implementation Details

**`index_load`:**
- Opens `.pes/index` with `fopen`; returns empty index (not error) if file doesn't exist
- Parses each line with `fscanf` using `%o %64s %llu %u %511s`
- Uses intermediate `unsigned long long` for mtime to avoid type mismatch segfault
- Converts hex hash string to binary using `hex_to_hash()`

**`index_save`:**
- Heap-allocates a sorted copy of the index (`malloc(sizeof(Index))`) — critical fix to avoid stack overflow since `Index` is ~5MB
- Sorts entries by path using `qsort`
- Writes to temp file using `mkstemp` with array syntax `char tmp_path[] = ".pes/index_tmp_XXXXXX"`
- Calls `fflush` + `fsync` + `fclose` before `rename` for atomicity

**`index_add`:**
- Reads file contents with `fopen`/`fread`
- Writes blob to object store using `object_write(OBJ_BLOB, ...)`
- Gets file metadata with `lstat` (mtime, size, mode)
- Updates or creates index entry, then calls `index_save`

### Key Fixes

1. **Stack overflow:** `Index sorted = *index` copied ~5MB onto the stack. Fixed by using `Index *sorted = malloc(sizeof(Index))`
2. **Segfault from implicit declaration:** Added forward declaration `int object_write(...)` at top of `index.c`
3. **fscanf type mismatch:** Used `unsigned long long mtime_tmp` intermediate variable instead of casting `uint64_t*` directly

### Testing

```bash
make pes
./pes init
echo "hello" > file1.txt
echo "world" > file2.txt
./pes add file1.txt file2.txt
./pes status
cat .pes/index
```

**Output:**
```
Initialized empty PES repository in .pes/
Staged changes:
  staged:     file1.txt
  staged:     file2.txt
Unstaged changes:
  (nothing to show)
Untracked files:
  untracked:  README.md
  ...

100644 2cf8d83d9ee29543b34a87727421fdecb7e3f3a183d337639025de576db9ebb4 1776751863 6 file1.txt
100644 e00c50e16a2df38f8d6bf809e181ad0248da6e6719f35f9f7e65d6f606199f7f 1776751863 6 file2.txt
```

---

## Phase 4: Commits and History

**File:** `commit.c`  
**Function implemented:** `commit_create`

### Concept

A commit ties together a tree snapshot, parent pointer, author, timestamp, and message. Commit text format:

```
tree <64-char-hex>
parent <64-char-hex>        ← omitted for first commit
author <name> <timestamp>
committer <name> <timestamp>

<message>
```

Commits form a linked list via parent pointers:
```
C3 ──► C2 ──► C1 ──► (no parent)
```

### Implementation Details

**`commit_create`:**
1. Calls `tree_from_index()` to build and store the tree snapshot
2. Fills a `Commit` struct with tree hash, author (`pes_author()`), timestamp (`time(NULL)`), and message
3. Calls `head_read()` to get the parent commit hash — if HEAD has no commits yet, `has_parent = 0`
4. Serializes with `commit_serialize()` and writes to object store with `object_write(OBJ_COMMIT, ...)`
5. Updates the branch pointer with `head_update()`

### Testing

```bash
make pes
./pes init
echo "Hello" > hello.txt
./pes add hello.txt
./pes commit -m "Initial commit"

echo "World" >> hello.txt
./pes add hello.txt
./pes commit -m "Add world"

echo "Goodbye" > bye.txt
./pes add bye.txt
./pes commit -m "Add farewell"

./pes log
```

**Output:**
```
Committed: a8110ed4f88f... Initial commit
Committed: 0069c919cb7c... Add world
Committed: ff03009aa58b... Add farewell

commit ff03009aa58b6f531b0fb754637d2502ca4ff69f22a773738313ee25b5a428fc
Author: PES User <pes@localhost>
Date:   1776752001
    Add farewell

commit 0069c919cb7cf1495a72240afb5bc051ecf9b6f7542ae5ff187d417b950dc1b0
Author: PES User <pes@localhost>
Date:   1776752001
    Add world

commit a8110ed4f88fcb1e80371d757fb2adc260dfb6fba1b84bfdad69dc67a379386c
Author: PES User <pes@localhost>
Date:   1776752001
    Initial commit
```

**Object store after three commits:**
```bash
find .pes -type f | sort
```
```
.pes/HEAD
.pes/index
.pes/objects/00/69c919cb7c...
.pes/objects/05/c8d777d0be...
.pes/objects/2c/f8d83d9ee2...
.pes/objects/66/224663d23e...
.pes/objects/85/e367b76c24...
.pes/objects/96/c0d5c28dd3...
.pes/objects/a8/110ed4f88f...
.pes/objects/e0/0c50e16a2d...
.pes/objects/e6/7ed66bcb4a...
.pes/objects/f1/0239b62037...
.pes/objects/ff/03009aa58b...
.pes/refs/heads/main
```

**Reference chain:**
```bash
cat .pes/refs/heads/main
# ff03009aa58b6f531b0fb754637d2502ca4ff69f22a773738313ee25b5a428fc

cat .pes/HEAD
# ref: refs/heads/main
```

---

## Phase 5: Branching Analysis

### Q5.1 — How would you implement `pes checkout <branch>`?

To switch branches, three things must change in `.pes/`:

**1. Update HEAD:** Write `ref: refs/heads/<branch>` into `.pes/HEAD` to point to the new branch.

**2. Read the target commit:** Open `.pes/refs/heads/<branch>`, read the commit hash, load the commit object, and get its tree hash.

**3. Update the working directory:** Recursively walk the target tree object. For each blob entry, read the blob from the object store and overwrite the corresponding working directory file. Create directories that exist in the new tree but not currently on disk. Delete files that exist in the old tree but are absent from the new tree.

**What makes this complex:**
- Files must be handled atomically enough that a crash midway doesn't corrupt the working directory
- Deletions are easy to miss — you must diff old tree vs new tree, not just apply new tree
- The operation must be refused if there are uncommitted local changes that would be overwritten (see Q5.2)
- Nested subdirectories require recursive tree traversal in both old and new snapshots

---

### Q5.2 — How to detect a "dirty working directory" conflict before checkout?

For each file tracked in the **current index**, compare three versions:

- **Index entry:** the staged hash, mtime, and size recorded at last `pes add`
- **Working directory file:** current mtime and size from `stat()`
- **Target branch's tree:** the blob hash from the target commit's tree object

A conflict requiring checkout refusal exists when **both** of these are true:
1. The working directory file differs from the index entry — detected via mtime/size mismatch (same fast-diff used in `index_status`), meaning there are local uncommitted changes
2. The target branch has a different blob hash for that file than the current branch's tree, meaning the file would need to be overwritten

If only condition 1 is true (file is dirty but both branches have the same content), checkout can safely proceed. If only condition 2 is true (file is clean, branches differ), checkout can safely overwrite it. Only when both are true would local work be silently destroyed.

No re-hashing is needed — the mtime/size metadata comparison is sufficient for detection, keeping the operation fast.

---

### Q5.3 — What happens in detached HEAD, and how to recover?

In detached HEAD, `.pes/HEAD` contains a raw commit hash directly instead of `ref: refs/heads/<branch>`. For example:
```
a8110ed4f88fcb1e80371d757fb2adc260dfb6fba1b84bfdad69dc67a379386c
```

**What happens when you commit:** New commits are created normally — each commit correctly records its parent. However, `head_update` writes the new commit hash directly into `HEAD` rather than into a branch file. No branch pointer advances to track these commits.

**The problem:** If the user switches to another branch, HEAD now points to that branch, and the detached commits become unreachable — no branch or ref points to them. They will eventually be deleted by garbage collection.

**Recovery:** The user needs the hash of the last detached commit (from terminal history or notes taken while still in detached state). Then:
```bash
# Create a new branch pointing to the detached commit
echo "<commit-hash>" > .pes/refs/heads/recovered
# Switch HEAD to that branch
echo "ref: refs/heads/recovered" > .pes/HEAD
```
This makes the commits reachable again through the new branch pointer.

---

## Phase 6: Garbage Collection Analysis

### Q6.1 — Algorithm to find and delete unreachable objects

Use a **mark-and-sweep** approach:

**Mark phase:** Start from every file in `.pes/refs/heads/`. For each branch:
1. Read the commit hash from the branch file
2. Load the commit object, add its hash to the **reachable set**
3. Load the commit's tree object, add its hash to the reachable set
4. Recursively walk all subtree objects, adding each tree hash
5. For every blob entry in every tree, add the blob hash
6. Follow the commit's parent pointer and repeat until no parent exists

**Sweep phase:** Walk every file under `.pes/objects/` using the two-level shard directory structure. For each object file, reconstruct its hash from the directory name + filename. If the hash is **not** in the reachable set, delete the file.

**Data structure:** A **hash set** (hash table of 64-character hex strings) for O(1) membership checks during the sweep phase.

**Estimate for 100,000 commits, 50 branches:** Assuming an average of 10 objects per commit (1 commit + 2–3 trees + 5–7 blobs), the mark phase visits roughly **1,000,000 objects**. The sweep phase scans every file on disk — also ~1,000,000 files in the object store. Total: approximately 2,000,000 object accesses.

---

### Q6.2 — Race condition between GC and concurrent commit

**The race condition:**

1. A `pes commit` operation calls `object_write` for a new blob — the blob is now stored in `.pes/objects/`
2. GC starts its **mark phase** — it reads all branch refs and walks all reachable objects. At this exact moment, the blob exists on disk but the commit object referencing it has not been written yet. GC does not see the blob as reachable.
3. GC's **sweep phase** runs and deletes the blob as unreachable garbage
4. The commit operation now writes the commit object and calls `head_update` — but the blob it references has been deleted. The repository is now corrupt.

**How Git's real GC avoids this:**

1. **Grace period:** Git's GC never deletes objects newer than 2 weeks old (configurable via `gc.pruneExpire`). Since in-progress commits complete in seconds, any object created recently is safe.

2. **Objects before refs:** Git always writes all objects (blobs, trees, commit) to disk before updating any ref. This means that even if GC runs between object writes and the ref update, all the objects are present and the commit can be completed safely. A partially complete commit leaves dangling objects (cleaned up by GC later) but never references missing ones.

3. **Lock files:** Git uses `.lock` files on refs during updates. GC checks for lock files and defers if a concurrent write is in progress, preventing the sweep from running while a commit is updating a ref.

---

## Integration Test

```bash
make test-integration
```

**Full output:**
```
=== PES-VCS Integration Test ===
--- Repository Initialization ---
Initialized empty PES repository in .pes/
PASS: .pes/objects exists
PASS: .pes/refs/heads exists
PASS: .pes/HEAD exists
--- Staging Files ---
Staged changes:
  staged:     file.txt
  staged:     hello.txt
Unstaged changes:
  (nothing to show)
Untracked files:
  (nothing to show)
--- First Commit ---
Committed: 1efc5e934ba2... Initial commit
--- Second Commit ---
Committed: 69ea93c5c551... Update file.txt
--- Third Commit ---
Committed: fd67c5003341... Add farewell
--- Full History ---
commit fd67c500334192abb4cbaa5a76e4d9b0fad7ace25e55f16087d50df9f04dbbe7
Author: PES User <pes@localhost>
Date:   1776752031
    Add farewell

commit 69ea93c5c5511a8088747ccb650d2033427d396514d9c3d39035975c0021b8d9
Author: PES User <pes@localhost>
Date:   1776752031
    Update file.txt

commit 1efc5e934ba27e0326b566b8f205bf026574538ad769ebae30ca50984970f3f7
Author: PES User <pes@localhost>
Date:   1776752031
    Initial commit
--- Reference Chain ---
HEAD:
ref: refs/heads/main
refs/heads/main:
fd67c500334192abb4cbaa5a76e4d9b0fad7ace25e55f16087d50df9f04dbbe7
--- Object Store ---
Objects created: 10
=== All integration tests completed ===
```

---

## Key Implementation Challenges and Fixes

| Issue | Root Cause | Fix |
|---|---|---|
| Segfault in `index_save` | `Index sorted = *index` copied ~5MB onto the stack, causing stack overflow | Used `Index *sorted = malloc(sizeof(Index))` to heap-allocate |
| Segfault in `index_add` | `object_write` was implicitly declared, causing wrong calling convention | Added forward declaration `int object_write(...)` at top of `index.c` |
| `fscanf` mismatch for mtime | Casting `uint64_t*` directly into `%llu` caused undefined behavior | Used intermediate `unsigned long long mtime_tmp` variable |
| `test_tree` linker error | Makefile for `test_tree` didn't link `index.o` | Built manually: `gcc -o test_tree test_tree.o object.o tree.o index.o -lcrypto` |
| `mkstemp` string literal crash | `char *tmp_path = "..."` is read-only; `mkstemp` modifies the string | Used `char tmp_path[] = "..."` (stack array, writable) |
