# EXT2-MUT namespace and recovery contract

This document covers the namespace/orphan work in Nilix #60, #72 and #74 and
the follow-up in [#75](https://github.com/Zero-kernel/Nilix/issues/75): allocation,
reads and reclamation through all three classic indirect levels, plus linear
directory growth. Transactions retain the bounded Zero-Intent v1 format.

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
| 14, PUBLISH | Publish an initialized regular file, directory or symlink and remove it from the orphan chain | superblock, child inode, parent inode, parent directory data |
| 15, TREE_ATTACH | Attach one zeroed mapping node or initialized regular-file data block | block bitmap, group descriptor, superblock, inode, optional parent mapping block (4/5 homes) |
| 16, TREE_DETACH | Remove one data leaf or empty mapping node | the same 4/5 homes |
| 17, DIR_GROW | Append one canonical empty directory block and increase size by exactly one block | the same 4/5 homes |
| 18, MAPPED_RENAME | Rename one entry in a mapped directory block | directory data and parent inode (2 homes) |

Kinds 1–14 retain their byte layout and digest offsets. New tree intents encode
a node level (0=data, 1–3=mapping) at byte 384, keeping bytes 385 onward zero.
The authenticated logical block is the canonical start of that subtree; recovery
derives its inode root and ancestor indices instead of trusting a supplied
physical parent. Each step changes exactly one edge, one bitmap bit, the matching
free counters and `i_blocks`. Root edits need four homes; internal edges need five.
The six-home constant is unchanged. Kind 18 authenticates the logical directory
block and physical home and retains the rename name/offset payload.

Mappings are initialized and flushed before attachment. A crash may leave an
empty mapping chain beyond EOF; the ownership proof permits such chains while
rejecting every data leaf beyond EOF. A fixed-depth reverse cursor frees data
from highest to lowest, then frees empty mapping ancestors. Same-size truncate
can finish cleanup of an interrupted empty chain. Traversal visits actual nodes,
not every logical hole, and does not restart at the root after each free.

Directory growth first flushes a canonical empty record (inode/name/type zero,
record length exactly one block), then atomically publishes its pointer, size and
allocation counts. Recovery checks that exact initialization, not merely that the
block parses. Live directories cannot be detached; only durable zero-link
orphans may use directory reclamation. Namespace insertion follows in a separate
transaction, so a failed insertion may leave valid extra capacity. Insertion
skips dirty legacy slack that cannot be reversed exactly, reuses suitable slots
and grows when none remain. Published raw inode and cached size move together.

Ordinary small creates retain their original FILE_CREATE fast path; exhausted or
unusable tail space falls back to ORPHAN_ALLOC/PUBLISH. Original last-direct-block
rename logs remain valid, while kind 18 addresses other directory blocks. Both
rename grammars enforce the source inode, names, NOREPLACE uniqueness and a
parent transition limited to mtime/ctime. FILE_CREATE also checks that the new
inode is an empty regular file, and cannot change unrelated parent fields.
Writers and recovery share the live-parent shape checks: at least two links,
nonzero block-aligned size, no high directory size/ACL word and no deletion time.
Kind 18 preserves untouched directory-entry padding, reversing only the old/new
name span. Extending a name requires zero bytes in the newly overwritten span;
otherwise the writer refuses before persistence.

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
sparse file offsets. An old object can never lend cached bytes to a reused inode. Truncate invalidation
also walks resident entries, so sparse multi-GiB offsets do not cause a logical
page-by-page scan.

## Validation routes

The hosted VFS and real-image suites cover regular-file and directory namespace lifetimes,
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
writes. It also reaches double/triple sparse offsets using the mounted block
geometry, reads and regrows a partial cached tail, and creates enough long names
to exceed all twelve direct directory blocks before reclaiming them. Disk-backed
kernel gates require the namespace marker. Mutation probes
use scratch files separate from the existing one-byte durability witness.

## Remaining boundaries

Cross-directory rename, replacing an existing destination, and in-place renames
whose new name does not fit the old record remain unsupported. Indexed (HTree)
directories are readable but their mutation is rejected; this writer grows linear
directories. Unsupported inode layouts and ACL blocks remain rejected. Directory
growth uses a 32-bit byte size and requires block sizes below 64 KiB, avoiding an
unsupported 64 KiB directory-record encoding. Files use all classic indirect
levels, capped by the u32 logical-block API and checked u32 sector accounting.
Sizes above 2 GiB require the existing LARGE_FILE feature; a feature upgrade is
not silently performed during data I/O.

A crashed truncate can expose a consistent partly truncated file until retry.
Large truncations retain the metadata gate across their bounded transaction
sequence. Metadata plans borrow payloads and traversal retains at most three
small frames, preserving the 32 KiB task-stack budget. No root-depth-sized block
arrays or logical-hole-sized temporary vectors are allocated.

The driver does not claim exclusive ownership against privileged concurrent
raw-device writes. Read-only mounts with pending orphans are refused because
cleanup requires writes. Final acceptance evidence is recorded separately from
this implementation contract.
