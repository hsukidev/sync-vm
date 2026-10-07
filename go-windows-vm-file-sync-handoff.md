# Go Windows Host ↔ Windows VM File Sync — Implementation Handoff

## Goal

Build a Go-based CLI that continuously synchronizes a directory on a **Windows host** into a directory inside a **Windows VM**.

Both sides are Windows.

**Important constraints:**
- Do not depend on rsync.
- Do not require third-party synchronization software on either machine.
- The host and VM should only need our binaries.
- The sync should be near-real-time.
- We control both the host client and VM agent, so define a purpose-built protocol.
- Optimize for correctness and reliability before maximum throughput.

The desired experience is:

```text
Host:
  edit file
      ↓
  change detected
      ↓
  delta generated
      ↓
  delta sent
      ↓
VM:
  file updated
```

## Architecture

```text
┌──────────────────────── HOST WINDOWS ────────────────────────┐
│                                                              │
│  filesystem                                                 │
│      │                                                       │
│      ▼                                                       │
│  Windows filesystem watcher                                  │
│      │                                                       │
│      ▼                                                       │
│  event normalizer / debouncer                                │
│      │                                                       │
│      ▼                                                       │
│  sync coordinator                                            │
│      │                                                       │
│      ├── snapshot/state                                      │
│      ├── coalescing                                          │
│      ├── hashing                                             │
│      └── sequence numbers                                    │
│      │                                                       │
│      ▼                                                       │
│  transport client                                            │
│                                                              │
└──────────────────────────┬───────────────────────────────────┘
                           │
                    persistent connection
                           │
                           ▼
┌────────────────────────── VM WINDOWS ────────────────────────┐
│                                                              │
│  transport server                                            │
│      │                                                       │
│      ▼                                                       │
│  protocol handler                                            │
│      │                                                       │
│      ▼                                                       │
│  filesystem applier                                          │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

## Core design principle

**Filesystem watcher events are triggers, not the source of truth.**

Windows filesystem notifications can be noisy and may represent intermediate operations.

Do not assume:

```text
1 filesystem event = 1 semantic file operation
```

Instead:

```text
Windows event
    ↓
"something changed"
    ↓
coordinator
    ↓
inspect filesystem
    ↓
determine authoritative state
    ↓
generate/coalesce delta
    ↓
send to VM
```

For example, an editor may generate:

```text
CREATE foo.tmp
WRITE foo.tmp
RENAME foo.tmp → foo.txt
DELETE foo.tmp
```

The sync layer should ideally produce:

```text
PUT foo.txt
```

rather than forwarding every raw event.

---

# Components

## 1. Host watcher

Watch the configured root directory recursively.

Responsibilities:

- detect create
- detect modify/write
- detect delete
- detect rename
- detect directory creation
- detect directory deletion
- trigger rescan/reconciliation when necessary

Use an established Go Windows filesystem watcher library where appropriate, but isolate it behind an internal interface.

Conceptually:

```go
type Watcher interface {
    Events() <-chan Event
    Errors() <-chan error
    Close() error
}
```

Do not let the rest of the system depend directly on the watcher implementation.

---

# 2. Sync coordinator

This is the core of the system.

Responsibilities:

- debounce noisy events
- coalesce multiple events
- maintain synchronization state
- determine actual filesystem state
- calculate file metadata/hashes
- generate semantic operations
- assign sequence numbers
- send operations to transport
- track acknowledgements

Prefer a single coordinator goroutine owning mutable sync state rather than having many goroutines mutate shared maps/state.

Conceptually:

```text
watcher
   │
   ▼
events channel
   │
   ▼
coordinator goroutine
   ├── pending paths
   ├── debounce timer
   ├── filesystem state
   ├── sequence number
   └── ACK state
          │
          ▼
       sender
```

---

# 3. Initial snapshot

When synchronization starts, perform an initial recursive scan of the host directory.

The VM must first be brought to a known state before live deltas are considered authoritative.

Suggested flow:

```text
HOST                              VM

HELLO --------------------------->

SNAPSHOT_START ------------------>

MKDIR src ----------------------->
PUT src/main.go ----------------->
PUT src/foo.go ------------------>
DELETE stale.txt ---------------->

SNAPSHOT_END -------------------->

<-------------------------------- ACK
```

Be careful about changes that occur while the initial snapshot is being generated.

Do not introduce a race where:

```text
snapshot scans file
       ↓
file changes
       ↓
watcher misses/reorders change
       ↓
VM has stale data
```

Use a snapshot generation/version or reconciliation approach so changes occurring during initial sync are eventually reflected in the VM.

Correctness is more important than minimizing duplicate work.

---

# 4. Delta protocol

Define a custom host ↔ VM protocol.

The protocol should be independent of the filesystem watcher and transport.

Initial message types:

```text
HELLO
SNAPSHOT_START
PUT
DELETE
MKDIR
RMDIR
RENAME
SNAPSHOT_END
ACK
ERROR
```

Every mutating operation gets a monotonically increasing sequence number.

Example:

```text
seq=100 PUT    src/a.go
seq=101 DELETE src/b.go
seq=102 RENAME src/c.go -> src/d.go
```

VM replies:

```text
ACK 102
```

The ACK means all operations through sequence 102 have been successfully applied.

---

# 5. Protocol framing

Use a framed protocol rather than relying on individual network reads/writes corresponding to messages.

For example:

```text
[length][payload]
```

The payload can initially be JSON.

Example conceptual message:

```go
type Message struct {
    Type string `json:"type"`
    Seq  uint64 `json:"seq"`

    Path    string `json:"path,omitempty"`
    OldPath string `json:"oldPath,omitempty"`

    Size uint64 `json:"size,omitempty"`
    Hash string `json:"hash,omitempty"`

    Data []byte `json:"data,omitempty"`
}
```

The exact schema can be improved during implementation.

Do not tightly couple the protocol to TCP or any specific transport.

---

# 6. Transport

Define a transport abstraction:

```go
type Transport interface {
    Send(ctx context.Context, msg Message) error
    Receive(ctx context.Context) (Message, error)
    Close() error
}
```

The first implementation should use whatever transport is appropriate for the VM environment.

Possible options:

- TCP
- Windows-specific local/VM networking
- VM-specific socket/transport mechanism

The synchronization protocol should not care which one is used.

---

# 7. Reliability and reconnect

The connection should be persistent while synchronization is active.

Track:

```text
next sequence
last sent sequence
last acknowledged sequence
```

Example:

```text
Host → VM: seq=1842 PUT foo
Host → VM: seq=1843 PUT bar
Host → VM: seq=1844 DELETE baz

VM → Host: ACK 1844
```

If the VM disconnects:

1. detect connection failure
2. reconnect
3. determine last acknowledged sequence
4. resume if state is still valid
5. otherwise perform a fresh reconciliation/snapshot

Do not assume a connection is reliable.

The first version does not need an elaborate durable journal, but the architecture should make resumability possible.

---

# 8. File transfer

For each file update, initially send the file contents.

Conceptually:

```text
PUT
path=src/main.go
size=12345
hash=<hash>
data=<contents>
```

Calculate a content hash to help detect redundant transfers.

Use a fast cryptographic hash such as SHA-256 initially.

Future optimization may include:

- content-addressed blobs
- deduplication
- chunking
- compression
- partial-file transfers

Do not implement these prematurely.

The first goal is correctness.

---

# 9. VM filesystem applier

The VM agent receives protocol messages and applies them to its filesystem.

Operations:

```text
PUT
DELETE
MKDIR
RMDIR
RENAME
```

The applier must handle:

- parent directory creation
- replacing existing files
- deleting files/directories
- atomic-ish updates where practical
- Windows path semantics
- locked files
- read-only files
- path normalization

Avoid allowing a malformed protocol message to escape the configured VM sync root.

**Path traversal must be prevented.**

For example, reject paths that would resolve outside the configured root:

```text
..\..\somewhere
C:\Windows\...
\\server\share\...
```

Treat the sync root as a security boundary.

---

# 10. Windows-specific considerations

Both host and VM are Windows.

Explicitly account for:

- Windows path separators
- drive letters
- case-insensitive filesystem behavior
- NTFS rename semantics
- file locking
- antivirus/indexer interference
- temporary files created by editors
- atomic-save patterns
- junctions
- symlinks
- long paths

Do not blindly treat paths as Unix paths.

Define a canonical internal path representation, preferably relative to the sync root.

Example:

```text
src/main.go
src/pkg/foo.go
```

rather than sending absolute paths over the protocol.

---

# 11. CLI

Initial command:

```text
mytool sync <host-directory>
```

Configuration should identify:

```text
host directory
VM target
VM directory
transport connection information
```

Potential future commands:

```text
mytool sync <path>
mytool status
mytool stop
mytool doctor
```

Keep CLI code separate from the sync engine.

---

# Suggested project structure

```text
cmd/
  mytool/
    main.go

cmd/
  mytool-agent/
    main.go

internal/
  watcher/
  sync/
  snapshot/
  protocol/
  transport/
  filesystem/
  hash/
  cli/
```

The VM agent can share protocol/filesystem packages with the host binary.

Do not over-engineer the package structure if the initial implementation is small.

---

# Event coalescing

This is important.

Given:

```text
WRITE foo
WRITE foo
WRITE foo
WRITE foo
```

produce:

```text
PUT foo
```

Given:

```text
CREATE foo.tmp
WRITE foo.tmp
RENAME foo.tmp → foo
```

produce:

```text
PUT foo
```

Given:

```text
CREATE foo
DELETE foo
```

before synchronization occurs, the net result may be:

```text
no-op
```

The coordinator should operate on the final filesystem state whenever practical.

A short debounce window such as 50–200ms can be used initially, but make it configurable if needed.

---

# Testing

Create unit and integration tests for:

### Event normalization

Test:

- repeated writes
- create + write
- rename
- rename + modify
- create + delete
- directory operations
- editor temporary-file patterns

### Snapshot

Test:

- empty directory
- nested directories
- binary files
- large files
- deleted/stale VM files
- changes occurring during snapshot

### Protocol

Test:

- encode/decode
- malformed messages
- sequence numbers
- ACK handling
- reconnect

### Filesystem applier

Test:

- PUT
- DELETE
- MKDIR
- RMDIR
- RENAME
- replacing existing files
- path traversal rejection

### End-to-end

Use temporary directories:

```text
host temp dir
      ↓
sync
      ↓
VM temp dir
```

Then:

1. create initial files
2. start synchronization
3. verify VM matches host
4. modify files
5. create files
6. delete files
7. rename files
8. verify VM converges to host

Also test VM disconnect/reconnect.

---

# V1 scope

The first working version should support:

1. Windows host watcher
2. Recursive initial snapshot
3. Windows VM agent
4. Persistent host↔VM connection
5. PUT
6. DELETE
7. MKDIR
8. RMDIR
9. RENAME
10. Event coalescing/debouncing
11. Sequence numbers
12. ACKs
13. Basic reconnect/reconciliation
14. File hashing
15. Path traversal protection
16. End-to-end tests

Do **not** initially implement:

- rsync
- block-level delta algorithms
- compression
- chunked transfers
- content-addressed storage
- complicated persistent databases
- distributed consensus/state machinery

Build the simplest correct system first.

# Design principle

The core invariant is:

> **Eventually, the VM filesystem must converge to the host filesystem.**

The watcher is only a mechanism for detecting that reconciliation may be necessary.

The sync engine is responsible for determining the correct state.

The protocol is responsible for reliably communicating that state.

The VM agent is responsible for safely applying it.

The desired pipeline is:

```text
Windows filesystem
      ↓
watch
      ↓
coalesce
      ↓
reconcile
      ↓
semantic delta
      ↓
sequence
      ↓
transport
      ↓
ACK
      ↓
VM filesystem
```

Prioritize **correctness, recovery, and simplicity** over maximum transfer performance in V1.
