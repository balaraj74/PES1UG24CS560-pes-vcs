# PES-VCS Lab Report

- SRN: PES1UG24CS560
- Platform: Ubuntu 22.04
- Author configuration used:
  - `export PES_AUTHOR="Your Name <PES1UG24CS560>"`

## Code Completion Summary

Implemented all required TODO functions:

- `object.c`
  - `object_write`
  - `object_read`
- `tree.c`
  - `tree_from_index`
- `index.c`
  - `index_load`
  - `index_save`
  - `index_add`
- `commit.c`
  - `commit_create`

## Verification Artifacts

The following files contain command outputs for each required screenshot checkpoint:

- Screenshot 1A: `phase1_test_objects.out`
- Screenshot 1B: `phase1_objects.out`
- Screenshot 2A: `phase2_test_tree.out`
- Screenshot 2B: `phase2_tree_xxd.out`
- Screenshot 3A: `phase3_sequence.out`
- Screenshot 3B: `phase3_index.out`
- Screenshot 4A: `phase4_log.out`
- Screenshot 4B: `phase4_files.out`
- Screenshot 4C: `phase4_refs.out`
- Final integration: `final_integration.out`

## Notes on Integration Test Path

`test_sequence.sh` computes an absolute binary path and executes `$PES` without quotes.
If the repository path has spaces, the script can fail due to shell word splitting.
Validation was done from a no-space symlink path to run the provided script unchanged.

## Analysis Answers

### Q5.1: How to implement `pes checkout <branch>`

Files to update in `.pes/`:

1. Resolve target commit:
   - Read `.pes/refs/heads/<branch>` to get target commit hash.
2. Update `HEAD`:
   - If normal branch checkout, write `ref: refs/heads/<branch>` into `.pes/HEAD`.
   - If checkout by hash, write raw commit hash (detached HEAD).
3. Update index:
   - Rebuild `.pes/index` so it matches the checked-out tree snapshot.

Working directory actions:

1. Read target commit object and its root tree.
2. Materialize the entire tree into working directory files:
   - Create directories as needed.
   - Write blob contents to files.
   - Set executable bit according to mode (`100755`).
3. Remove tracked files/directories present in current snapshot but absent in target snapshot.

Why checkout is complex:

- It is a 3-way state transition across:
  - current working directory,
  - current index,
  - target commit tree.
- It must detect overwrite risks (uncommitted changes).
- It must handle deletes, mode changes, nested trees, and conflicts atomically or safely roll back.

### Q5.2: Detect dirty working-directory conflicts with index + object store

Goal: refuse checkout when local uncommitted changes would be overwritten by switching branches.

Algorithm:

1. Build map A: current index entries (`path -> staged hash, mode, metadata`).
2. Build map B: current `HEAD` tree entries by recursively reading tree/blob hashes from object store.
3. Build map C: target branch tree entries (same recursive traversal).
4. For each tracked path in union of A and B:
   - Compute working-copy state for that path:
     - If file missing -> state = deleted.
     - Else compare stat metadata with index metadata.
     - If metadata differs, re-hash file and compare with index hash to confirm true content change.
   - Path is "dirty" if working file differs from index entry.
5. Conflict rule:
   - If path is dirty AND target version (`C[path]`) differs from current committed version (`B[path]`), refuse checkout.
   - Also refuse when path is dirty and path would be deleted/created in target in a way that overwrites local data.

This uses only:

- index metadata + staged hashes,
- object store trees/blobs for current and target snapshots.

### Q5.3: Detached HEAD behavior and recovery

Detached HEAD means `.pes/HEAD` contains a commit hash directly, not a branch ref.

If commits are made in detached state:

1. New commits are created normally.
2. Parent chain continues from detached commit.
3. No branch pointer moves to the new commits.

Result:

- Commits are reachable only from HEAD temporarily.
- Once HEAD moves elsewhere, these commits can become unreachable and later eligible for GC.

Recovery methods:

1. Immediately create a branch at the detached commit:
   - Write hash to `.pes/refs/heads/<new-branch>`.
2. Or later recover from reflog-like history (if implemented) or from manual hash inspection before GC.

### Q6.1: Find and delete unreachable objects

Mark-and-sweep algorithm:

1. Roots:
   - All branch tips in `.pes/refs/heads/*`.
   - Optionally detached HEAD commit if it is a raw hash.
2. Mark phase:
   - DFS/BFS from each root commit.
   - For commit objects:
     - mark commit hash,
     - traverse `parent` (if any),
     - traverse `tree`.
   - For tree objects:
     - mark tree hash,
     - for each entry:
       - if subtree, traverse child tree hash,
       - if blob, mark blob hash.
3. Sweep phase:
   - Scan `.pes/objects/*/*`.
   - Convert file path to object hash.
   - Delete object file if hash is not in marked set.

Data structure:

- Hash set of 32-byte hashes (or 64-char hex keys) for O(1) expected membership checks.
- Queue/stack for traversal.

Estimated visited objects for 100,000 commits and 50 branches:

- Upper bound branches add little if histories overlap.
- Commits visited: about 100,000 unique commits.
- Trees: roughly one root tree per commit plus subtrees, often reused; commonly around 100,000 to 300,000.
- Blobs: highly workload dependent; often several hundred thousand to a few million unique blobs.

So practical traversal is typically on the order of a few hundred thousand to a few million objects, not `100000 * 50`, because shared history is visited once.

### Q6.2: Why concurrent GC with commit is dangerous

Race example:

1. Commit process writes new blob/tree objects first.
2. Before it writes commit object and updates branch ref, these new objects are not reachable from any ref.
3. Concurrent GC runs mark phase from current refs and does not mark those new objects.
4. GC sweep deletes those "unreachable" objects.
5. Commit process then writes commit object referencing now-deleted objects and updates ref.
6. Repository now has a broken commit pointing to missing tree/blob objects.

How Git avoids this:

1. Uses lock files and coordinated ref updates.
2. Uses conservative GC policies (grace periods, prune windows) so very recent loose objects are not immediately pruned.
3. Packs and references are handled with additional safety around object visibility and atomicity.
4. GC and write operations are designed to avoid deleting objects that might still be in-flight.
