# EXT2-MUT namespace and recovery contract

This document describes the implementation audited for Nilix PRs #72 and #74,
tracked by issue #60. It extends the existing bounded Zero-Intent journal with
unlink, orphan reclamation, mkdir/rmdir and symbolic links. Single-indirect file
allocation and truncation are included; double-indirect allocation remains
outside the scheduled EXT2-MUT work.

## Persistent transitions

Each transaction contains a descriptor, at most six metadata post-images and a
commit containing the exact typed intent and pre-image hashes. Recovery verifies
the complete transaction digest before interpreting its intent. A sealed but
invalid intent is an error, not an uncommitted tail that may be discarded.
Candidate discovery supports every metadata count from one through six, including
three-home namespace transactions whose parent and child share an inode block.

| Kind | Transition | Metadata homes (duplicates coalesced) |
| --- | --- | --- |
| 11, UNLINK | Remove one name, decrement links and place a zero-link inode on the orphan chain | superblock, child inode, parent inode, parent directory data |
| 12, INODE_FREE | Remove an empty orphan and free its inode slot | inode bitmap, group descriptor, superblock, child inode, optional predecessor inode |
| 13, ORPHAN_ALLOC | Allocate an unpublished zero-link inode and add it to the orphan chain | inode bitmap, group descriptor, superblock, child inode |
| 14, PUBLISH | Publish an initialized directory or symlink and remove it from the orphan chain | superblock, child inode, parent inode, parent directory data |

The chain uses Ext2's `s_last_orphan` and the orphan inode's `i_dtime` next pointer.
Only allocated zero-link regular files, directories and symlinks may appear.
Validation bounds the chain to 1024 entries, rejects cycles and checks every
allocated inode's membership. Ordinary directory entries cannot name an orphan;
the detached directory's own `.` and `..` records are special cases.

An unlink does not free storage reachable from an open description. Reclamation
starts when the canonical inode loses its last reference, or during sync/mount
recovery. It frees file blocks from highest to lowest with the existing typed
free transactions, then atomically clears the inode bitmap and inode contents.
It can remove an orphan behind another still-open orphan. Failed admission can
leave a durable orphan for a later retry. Ambiguous write failures fail-stop the
mount; mount-time cleanup must finish before the root is published.

Directory creation allocates an unpublished orphan, writes and flushes `.` and
`..`, then publishes its name and link counts. Symlinks follow the same scheme.
Targets shorter than 60 bytes use the inode's inline storage; a 60-byte target
uses a data block. Targets must be nonempty, contain no NUL, and be shorter than
both the filesystem block size and 4096 bytes, as required by Ext2/e2fsprogs.
Failed or interrupted initialization has a durable reclamation path.

## Admission and recovery proofs

Every intent has a fixed set of reversible edits. Recovery reconstructs the
pre-images, compares their hashes, and checks partially checkpointed homes
against the permitted prefix-torn shape. Both pre- and post-image ownership
graphs must match allocation bitmaps and free counters before replay writes.
Namespace recovery additionally validates complete parent directory records,
name uniqueness, the exact insertion/removal position, directory `.`/`..`
identity, and initialized symlink bytes. Re-signing a malformed transaction does
not authorize it.

Inode allocation/free supports 128- and 256-byte on-disk inodes. The commit
records up to 128 extended inode bytes so deletion and reuse clear timestamps or
inline attributes without losing the exact pre-image proof. Directory counts,
parent links, file links, inode free counts and bitmap edits are checked with
bounded arithmetic. Directory type bytes follow the FILETYPE feature bit; old
16-bit-name-length directories retain a zero high byte. Every live directory
reference must name an allocated inode at mount/recovery. Unlink additionally
proves the target's link count against all live directories before any write,
so malformed low counts cannot free an inode still reachable under another name.
The same proof applies to a recovered UNLINK pre-image.

Lock order is orphan-chain serialization (when needed), parent/child inode
locks, filesystem metadata, then journal. Inode-cache publication is serialized
with allocation/free. Namespace lookup rejects a zero-link cached object; an
existing open description retains access to it. Last-reference cleanup uses
nonblocking lock admission to avoid recursive destruction deadlocks.

Page-cache identities belong to individual canonical inode object lifetimes,
not reusable disk inode numbers. Retirement walks existing cache entries,
clears write-through dirty state and releases reclaimable pages without scanning
sparse file offsets. An old object can never lend cached bytes to a reused inode.

## Validation routes

The hosted VFS suite covers regular-file and directory namespace lifetimes,
non-head orphan removal, inode/block reuse, link-count and type failures,
coalesced inode blocks, inline/block symlink boundaries, every persistence step
with and without prior unflushed writes retained, interrupted recovery, and
complete re-signed malformed logs for all new intent kinds. Boot tests sample
fault boundaries and exercise the production page cache and device path.

The `ext2_mut_image` Cargo example in `scripts/tests` runs against an explicit
disposable mke2fs image and writes only the requested output image. Its recovered
outputs are checked independently with `e2fsck -fn`; private journal bits are
removed only from a copy whose journal is already empty. This is a simulated
write/flush failure model, not a claim about arbitrary physical-media failures.

The Ring3 Ext3 lane tests mkdir/rmdir, both symlink forms, readlink, open-unlink,
replacement-file isolation and a fork reader racing repeated truncate/grow and
writes. Disk-backed kernel gates require the namespace marker. Mutation probes
use scratch files separate from the existing one-byte durability witness.

## Remaining boundaries

Directory insertion uses existing directory space; directory growth and broader
rename semantics remain separate work. Namespace mutations reject unsupported
inode layouts/ACL blocks and directories beyond the direct-block range. File
mutation supports direct and single-indirect allocations; deeper allocation and
reclamation remain unsupported. A crashed truncate can expose a consistent
partly truncated file until the caller retries. Large truncations retain the
metadata gate across their bounded sequence of transactions. Runtime transaction
plans borrow their bounded metadata payloads, keeping nested mkdir/write/truncate
paths within the 32 KiB task stack with interrupt-handler headroom.

The driver does not claim exclusive ownership against privileged concurrent
raw-device writes. Read-only mounts with pending orphans are refused because
cleanup requires writes. Final acceptance evidence is recorded separately from
this implementation contract.
