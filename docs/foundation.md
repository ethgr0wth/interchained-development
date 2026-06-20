# NEDB × ITC: A Technical Foundation for Sovereignty

> This document is grounded in code. Every claim below is traceable to a specific
> file, function, or observed behavior. Nothing is aspirational — the system
> described here is running on mainnet as of June 2026.

---

## What Is Running

**interchainedd** — a Bitcoin-derived full node with one architectural change:
LevelDB has been replaced by NEDB at the storage seam (`src/dbwrapper.cpp`,
`CDBWrapper`). The NEDB engine is compiled in as a Rust staticlib
(`nedb-ffi/libnedb_ffi.a`) via C FFI. There is no separate database process.
There is no cloud dependency. The binary runs on commodity hardware and syncs
directly to the ITC peer-to-peer network.

The macOS x86_64 binary built by Codemagic is currently syncing mainnet.

---

## How NEDB Stores Data

*Source: `rust/nedb-v2/src/store.rs`, `src/db.rs`, `src/lib.rs`*

**One object, one file.** Every write creates a single file at:

```
objects/{blake2b_hash[..2]}/{blake2b_hash[2..]}
```

The file contains the node serialized as JSON, optionally encrypted with
AES-256-GCM. The filename *is* the content hash — the same model git uses for
blobs.

**Writes are atomic.** The code writes to a `.tmp` file then calls `fs::rename`.
Either the object exists or it does not. There is no partial-write state and no
journal to replay.

**Reads verify integrity.** Every `ObjectStore::read()` call recomputes the
BLAKE2b hash of the retrieved bytes and compares it to the filename. A tampered
object is detected on first access:

```rust
let actual = blake2b(&c);
if actual != hash { bail!("object {} tampered..."); }
```

**Cold start is instant.** The code comment at `src/lib.rs:12` is explicit:
*"Instant cold start: no AOF replay."* There is no log to scan, no compaction
to wait for. The objects are on disk and the index files open directly.

---

## The Causal Structure

*Source: `rust/nedb-v2/src/store.rs:21-46`*

Every stored node carries:

```
id, coll, seq, data, prev, caused_by, ts, valid_from, valid_to, hash
```

`prev` and `caused_by` are hashes of prior nodes. They are not metadata — they
are the structure. Each UTXO write in itcd is a node that points to the block
that created it and the transaction output that defines it.

DAG edges are stored as filesystem entries at:

```
graph/{from_hash}/{edge_type}/{to_hash}
```

Traversing provenance is a filesystem `read_dir` walk — no query planner, no
lock.

This means the UTXO set is not a flat table. It is a directed acyclic graph
where every coin's full causal history — who created it, in which block,
spending which prior output — is structurally encoded and BLAKE2b-verifiable at
every node.

---

## Why This Is a Sovereignty Primitive

**You hold the full state.** A full interchainedd node downloads every block,
validates every transaction, and writes the resulting UTXO set to its local NEDB
store. There is no trusted third party serving you chain state. You verify it
yourself.

**The history is tamper-evident.** Because every object is named by its BLAKE2b
hash and every read verifies that hash, a modified object is detected locally —
no remote attestation required. The MANIFEST file at the root is a Merkle hash
of all collection heads. You can prove the state of your UTXO set to anyone with
a hash comparison.

**Encryption is at the storage layer.** The node struct is optionally encrypted
with AES-256-GCM before the hash is computed. This is not TLS — it is encryption
of the data at rest, keyed to whatever key you supply. Your node's chain state
can be encrypted on disk.

**No vendor lock-in.** The object format is BLAKE2b-named JSON files. The graph
is a directory tree. These are readable with standard tools. There is no
proprietary wire format, no SDK required to inspect your own data.

**Causal provenance is non-repudiable.** Because `caused_by` is part of the
hashed node content, you cannot retrospectively alter what caused a state
transition without invalidating every downstream hash. The causal chain is
cryptographically sealed.

---

## What Is Proven vs. What Is Next

**Proven (running code):**

- NEDB replaces LevelDB end-to-end in a Bitcoin-derived full node
- macOS x86_64 binary syncs ITC mainnet
- Atomic writes, BLAKE2b verification, and causal DAG are in production Rust code
- Sync speed is materially faster than expected — consistent with sequential
  atomic file creates vs. LevelDB's write-amplified compaction model

**Next (not yet verified):**

- Windows binary (in progress — CI building)
- RPC exposure of causal trace queries (TRACE provenance not yet surfaced as a
  node RPC endpoint)
- Performance benchmarks under full UTXO set load (>200k blocks)
- AES-256-GCM key management for encrypted node deployments
- Code signing for macOS distribution (Developer ID cert via Elara, pending
  wiring)

---

*© INTERCHAINED LLC — built with Claude Sonnet 4.6*
