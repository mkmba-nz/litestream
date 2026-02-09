# Specification: Litestream Leader Election and Exec Orchestration

## Status: Draft
## Date: 2026-02-09
## Branch: claude/sync-upstream-main-MM4jG (synced to upstream/main at c32f6c8)

---

## 1. Motivation

We want to run multiple instances of an application backed by a single SQLite database,
with one instance acting as the read-write primary and the others as read-only replicas.
Litestream v0.5 already has all the low-level primitives for this but lacks the
orchestration layer to tie them together.

The goal is to extend Litestream's `replicate` command so that it:

1. Participates in leader election via S3-based locking
2. Runs the child application in the appropriate mode (primary or replica)
3. Handles promotion (replica to primary) and demotion (primary to replica) by
   gracefully restarting the child process

This keeps the complexity inside Litestream. The application only needs to read a single
environment variable (`LITESTREAM_ROLE`) to know its mode.

---

## 2. Existing Primitives (What Already Exists)

All references are to the upstream `main` branch at commit `c32f6c8`.

### 2.1 S3 Leaser (`leaser.go`, `s3/leaser.go`)

**Interface** (`leaser.go`):
```go
type Leaser interface {
    Type() string
    AcquireLease(ctx context.Context) (*Lease, error)
    RenewLease(ctx context.Context, lease *Lease) (*Lease, error)
    ReleaseLease(ctx context.Context, lease *Lease) error
}

type Lease struct {
    Generation int64
    ExpiresAt  time.Time
    Owner      string
    ETag       string
}
```

**S3 implementation** (`s3/leaser.go`, ~300 lines):
- Uses S3 conditional writes (`If-Match`/`If-None-Match` with ETags) on a `lock.json` object
- Default TTL: 30 seconds
- Owner: auto-detected from hostname + PID
- Handles: expired lease takeover, ETag-based race protection (412 PreconditionFailed),
  generation incrementing
- Fully tested with 9 tests including 10-way concurrent acquisition races

**Error types**:
- `LeaseExistsError{Owner, ExpiresAt}` — returned when another instance holds the lease
- `ErrLeaseNotHeld` — returned when renewal fails because another instance took over

**Status**: Complete and tested, but not wired into the `replicate` command or DB lifecycle.

### 2.2 Primary Mode: DB + Replica (`db.go`, `replica.go`)

The `DB` struct monitors a local SQLite database file:
- Watches the WAL for new transactions
- Converts WAL frames into LTX files (compact, page-level diffs)
- The `Replica` uploads LTX files to S3 via `ReplicaClient`
- Managed by a `Store` which handles background compaction across levels (L0-L7)

**Relevant operations**:
- `store.Open(ctx)` — opens all DBs, starts background monitors
- `store.Close(ctx)` — final sync, closes all DBs
- `db.Sync(ctx)` — forces WAL processing
- `db.Replica.Sync(ctx)` — forces upload of pending LTX files

This is the proven, battle-tested code path. Applications write to the local SQLite
file normally; Litestream is invisible.

### 2.3 VFS Mode: Read Replicas (`vfs.go`, `cmd/litestream-vfs/`)

The VFS is a loadable SQLite extension that reads pages directly from S3:

- Registered as SQLite VFS named `"litestream"` via `sqlite3vfs.RegisterVFS()`
- Applications open with `file:mydb.db?vfs=litestream`
- Pages fetched on demand with 10MB LRU cache
- Background poller (default 1s) picks up new LTX files from S3
- Pending index mechanism ensures active readers get a consistent snapshot

**Hydration** (background local restore):
- When `HydrationEnabled=true`, VFS downloads all pages to a local SQLite file
- `Hydrator.Restore()` — initial full restore from LTX files
- `Hydrator.CatchUp()` — incremental updates (fetches only new LTX files)
- Once complete, all reads come from the local file (fast)

**SQL functions** available to applications:
- `litestream_txid()` — returns current transaction ID as hex string
- `litestream_lag()` — returns seconds since last successful poll

**Write support**: Exists behind `LITESTREAM_WRITE_ENABLED=true` with conflict detection
via TXID comparison, but single-writer only. Not relevant to this spec (replicas are
read-only).

**Configuration**: Via environment variables:
- `LITESTREAM_REPLICA_URL` — required, e.g. `s3://bucket/path`
- `LITESTREAM_LOG_LEVEL` — DEBUG or INFO
- `LITESTREAM_HYDRATION_ENABLED` — true/false
- `LITESTREAM_HYDRATION_PATH` — local file path

### 2.4 Exec Mode (`cmd/litestream/replicate.go`)

The `replicate` command accepts `-exec CMD` or `exec:` in YAML config:

```go
c.cmd = exec.CommandContext(ctx, execArgs[0], execArgs[1:]...)
c.cmd.Env = os.Environ()
c.cmd.Stdout = os.Stdout
c.cmd.Stderr = os.Stderr
c.cmd.Start()
go func() { c.execCh <- c.cmd.Wait() }()
```

- Child inherits Litestream's environment, stdout, stderr
- Litestream waits on `execCh` for child exit, then does final sync and shuts down
- Child is killed via context cancellation if Litestream is signalled

### 2.5 Restore (`replica.go`)

`Replica.Restore(ctx, opts)`:
- Calculates restore plan via `CalcRestorePlan()` (finds minimal set of LTX files)
- Compacts them into a single stream via `ltx.Compactor`
- Decodes directly into output file via `ltx.Decoder.DecodeDatabaseTo()`
- Atomic: writes to temp file, renames on success

`CalcRestorePlan()` supports incremental plans (from TXID X to latest).

---

## 3. What Does NOT Exist

### 3.1 Orchestration Layer
No code connects the leaser to mode selection. There is no "if lease acquired, run as
primary; otherwise run as replica" logic.

### 3.2 Mode Transitions
No mechanism to stop a running child, close one mode (VFS or DB), and reopen in the
other mode. The exec lifecycle is strictly linear: start child, wait for exit, shutdown.

### 3.3 HTTP Proxy / Read-After-Write Consistency
No HTTP proxy, no cookie-based TXID tracking, no read-after-write consistency middleware.
This existed in the sister project LiteFS (`superfly/litefs`, `http/proxy_server.go`)
but was never part of Litestream. The LiteFS proxy used a `__txid` cookie to hold
replica reads until caught up. This is **out of scope for this spec** — it would be
implemented at the application layer (e.g. Rails middleware using `litestream_txid()`).

### 3.4 Live Promotion Without Restart
The VFS and primary DB are fundamentally different code paths. A process running in VFS
mode cannot transition to primary mode without closing the VFS database and reopening as
a local file. This means a child process restart is required on promotion/demotion.

---

## 4. Proposed Design

### 4.1 Overview

Extend the `replicate` command with a state machine that manages two modes and
transitions between them based on S3 lease status.

```
                          ┌──────────────────────────┐
            lease         │      PRIMARY MODE         │
          acquired ──────>│                           │  lease renewal
                          │  DB: local WAL → S3      │    fails
                          │  Child: LITESTREAM_ROLE=  │──────┐
                          │         primary           │      │
                          └──────────────────────────┘      │
                                    ▲                        │
                                    │ lease acquired         │
                                    │                        ▼
                          ┌─────────┴────────────────┐
            lease         │      REPLICA MODE         │
           failed ──────> │                           │
                          │  VFS: reads from S3       │
                          │  Child: LITESTREAM_ROLE=  │
                          │         replica           │
                          └──────────────────────────┘
```

### 4.2 Configuration

New fields in `Config` struct and YAML:

```yaml
# Existing
exec: "bundle exec rails server"

# New
leaser:
  type: s3
  bucket: my-bucket
  path: myapp           # lock.json stored at myapp/lock.json
  ttl: 30s              # default 30s

# Optional: separate commands per mode (overrides exec)
exec-primary: "bundle exec rails server"
exec-replica: "bundle exec rails server --readonly"
```

If `leaser` is not configured, behaviour is unchanged — Litestream runs in primary mode
with the single `exec` command (backwards compatible).

If `leaser` is configured but only `exec` is provided (not `exec-primary`/`exec-replica`),
the same command is used for both modes. The child differentiates via `LITESTREAM_ROLE`.

### 4.3 Environment Variables Passed to Child

| Variable | Primary Mode | Replica Mode |
|---|---|---|
| `LITESTREAM_ROLE` | `primary` | `replica` |
| `LITESTREAM_REPLICA_URL` | _(not set)_ | `s3://bucket/path` |
| `LITESTREAM_HYDRATION_ENABLED` | _(not set)_ | `true` |
| `LITESTREAM_HYDRATION_PATH` | _(not set)_ | `/data/.litestream/hydration.db` |

The application reads `LITESTREAM_ROLE` to determine its behaviour. For many apps this
means simply choosing between a read-write and read-only database connection.

### 4.4 State Machine

#### 4.4.1 Startup

```
1. Parse config, create leaser
2. Try AcquireLease()
3. If success:
     → enter PRIMARY mode
4. If LeaseExistsError:
     → enter REPLICA mode
5. If other error:
     → retry with backoff (lease service may be temporarily unavailable)
```

#### 4.4.2 Primary Mode

```
1. Restore from S3 if local DB doesn't exist (existing restoreIfNeeded logic)
2. Open Store (starts DB monitoring, WAL processing, S3 upload)
3. Fork child with LITESTREAM_ROLE=primary
4. Start lease renewal goroutine:
     - Every TTL/3 (~10s): call RenewLease()
     - On success: continue
     - On ErrLeaseNotHeld: trigger DEMOTION
     - On transient error: log warning, retry next cycle
       (lease has TTL headroom, a few missed renewals are safe)
5. Wait for:
     a. Child exit → release lease, final sync, shutdown
     b. Signal (SIGTERM/SIGINT) → SIGTERM child, release lease, final sync, shutdown
     c. Demotion trigger → enter transition to REPLICA mode
```

#### 4.4.3 Replica Mode

```
1. Register VFS with S3 replica client (hydration enabled)
2. Wait for VFS initial restore to complete
3. Fork child with LITESTREAM_ROLE=replica
4. Start lease watch goroutine:
     - Every ~5s: call AcquireLease()
     - On LeaseExistsError: continue (master still alive)
     - On success: trigger PROMOTION
     - On transient error: log warning, continue
5. Wait for:
     a. Child exit → shutdown
     b. Signal → SIGTERM child, shutdown
     c. Promotion trigger → enter transition to PRIMARY mode
```

#### 4.4.4 Transition: Demotion (Primary → Replica)

This is an unusual event (we lost our lease, meaning another instance took over or
we had a network partition). The safe response is to stop writing immediately.

```
1. Stop lease renewal goroutine
2. SIGTERM child process
3. Wait for child exit (with timeout, then SIGKILL)
4. Close Store (final sync — may fail if lease truly lost, that's OK)
5. → Enter REPLICA mode (step 1)
```

#### 4.4.5 Transition: Promotion (Replica → Primary)

This is the expected failover path. The old master is gone.

```
1. Stop lease watch goroutine
   (lease already acquired in step 4 of replica mode)
2. SIGTERM child process
3. Wait for child exit (with timeout, then SIGKILL)
4. Close VFS
5. Restore DB from S3 to local path
   - If hydration was active, the hydration file is already up to date.
     Move/copy it to the DB path instead of a full restore from S3.
   - Otherwise, full Replica.Restore() from S3
6. Open Store in primary mode (starts WAL monitoring, S3 upload)
7. → Enter PRIMARY mode (step 3 — fork child)
```

**Hydration optimisation for fast promotion**: If hydration is enabled and complete,
the hydrated local file is already a valid SQLite database at the latest TXID. The
promotion path can simply rename/copy this file to the DB path, skipping the S3
restore entirely. This reduces the promotion outage from seconds to milliseconds.

### 4.5 Estimated Promotion Outage

| Step | Without Hydration | With Hydration |
|---|---|---|
| SIGTERM child | ~0s | ~0s |
| Wait for child exit | 1-5s (app-dependent) | 1-5s |
| Close VFS | ~0s | ~0s |
| Restore from S3 | 0.5-10s (DB size dependent) | ~0s (file rename) |
| Open as primary | ~0s | ~0s |
| Fork new child | 1-3s (app startup) | 1-3s |
| **Total** | **~3-18s** | **~2-8s** |

### 4.6 Graceful Shutdown

When Litestream receives SIGTERM/SIGINT:

**In primary mode:**
1. Stop lease renewal
2. SIGTERM child, wait for exit
3. Final DB sync + replica sync (existing shutdown logic)
4. Release lease (explicit, so another instance can acquire immediately rather than
   waiting for TTL expiry)
5. Exit

**In replica mode:**
1. SIGTERM child, wait for exit
2. Close VFS
3. Exit (no lease to release)

### 4.7 Split-Brain Prevention

The S3 leaser uses ETags for conditional writes, which prevents two instances from
both believing they hold the lease. However, consider this sequence:

1. Instance A holds lease, is primary
2. Network partition: A can't reach S3, renewal fails
3. Instance B acquires expired lease, becomes primary
4. A's renewal failure triggers demotion, A becomes replica

The risk window is the **TTL duration** (30s default). During this window, A is still
running as primary (its child process is still writing) while B is promoting. This is
mitigated by:

- A detects renewal failure at TTL/3 (~10s), leaving ~20s of TTL remaining
- A immediately stops its child on renewal failure
- B doesn't start writing until after full promotion (restore + child start)
- In practice, A will have stopped before B's child starts

For additional safety, the primary-mode child could periodically verify it still holds
the lease by checking a lease status endpoint (future enhancement, not in initial scope).

---

## 5. Code Changes Required

### 5.1 `cmd/litestream/main.go` — Config

Add to `Config` struct:

```go
type Config struct {
    // ... existing fields ...

    Exec        string `yaml:"exec"`
    ExecPrimary string `yaml:"exec-primary"`
    ExecReplica string `yaml:"exec-replica"`

    Leaser *LeaserConfig `yaml:"leaser"`
}

type LeaserConfig struct {
    Type   string         `yaml:"type"`    // "s3"
    Bucket string         `yaml:"bucket"`
    Path   string         `yaml:"path"`
    Region string         `yaml:"region"`
    TTL    *time.Duration `yaml:"ttl"`
}
```

~30 lines. Config parsing, validation, defaults.

### 5.2 `cmd/litestream/replicate.go` — State Machine

Refactor `ReplicateCommand` to add:

```go
type ReplicateCommand struct {
    // ... existing fields ...

    leaser litestream.Leaser
    lease  *litestream.Lease
    mode   string // "primary" or "replica"

    // VFS state (replica mode)
    vfs       *litestream.VFS
    vfsClient litestream.ReplicaClient
}
```

New methods:
- `runWithLeader(ctx)` — top-level state machine loop (replaces linear exec flow)
- `enterPrimaryMode(ctx)` — open Store, fork child, start renewal
- `enterReplicaMode(ctx)` — register VFS, fork child, start lease watch
- `transitionToPrimary(ctx)` — kill child, close VFS, restore, open Store
- `transitionToReplica(ctx)` — kill child, close Store, open VFS
- `renewLeaseLoop(ctx)` — background goroutine for primary mode
- `watchLeaseLoop(ctx)` — background goroutine for replica mode
- `stopChild()` — SIGTERM, wait with timeout, SIGKILL
- `forkChild(role string)` — start child with LITESTREAM_ROLE env var

~280 lines of new/modified code. The existing non-leaser path (`leaser == nil`)
remains unchanged for backwards compatibility.

### 5.3 `cmd/litestream/replicate.go` — VFS Setup for Replica Mode

When entering replica mode, the replicate command needs to:

1. Create a `ReplicaClient` from the DB config's replica URL
2. Create a `VFS` with hydration enabled
3. Register it via `sqlite3vfs.RegisterVFS()`
4. Set environment variables for the child process

~40 lines. This reuses the same setup logic as `cmd/litestream-vfs/main.go`.

Note: This requires linking the VFS code into the main `litestream` binary (it's
currently only in the separate `litestream-vfs` extension). The VFS Go code is already
in the `litestream` package; the only question is whether `sqlite3vfs.RegisterVFS()`
can be called from the main binary. This needs investigation — the VFS registration
uses CGo via `github.com/psanford/sqlite3vfs`, which may conflict with the pure-Go
`modernc.org/sqlite` used by the primary DB path.

**Alternative for replica mode**: Instead of registering the VFS in-process, the
replicate command could use the `Hydrator` directly (without the VFS layer) to maintain
a local read-only SQLite file. The child process reads this file with standard SQLite.
This avoids the CGo/VFS dependency entirely, at the cost of slight lag (hydration
catch-up runs every poll interval). This is the simpler path and is recommended for
the initial implementation.

### 5.4 Build Considerations

The VFS extension (`cmd/litestream-vfs/`) is built with CGo build tags
(`SQLITE3VFS_LOADABLE_EXT`). The main `litestream` binary uses `modernc.org/sqlite`
(pure Go). These two SQLite implementations cannot coexist in the same process due to
POSIX per-process file locking.

**Recommended approach**: In replica mode, use the `Hydrator` directly to maintain a
local file, rather than registering the VFS in the main binary. The child process then
reads a normal SQLite file — no VFS extension needed. This keeps the main binary
pure-Go and avoids the CGo dependency.

```
Replica mode (recommended approach):
  litestream binary:
    Hydrator pulls LTX from S3 → writes to /data/replica.db
  child process:
    Opens /data/replica.db as read-only (standard SQLite, any language)
```

---

## 6. Configuration Examples

### 6.1 Minimal (Single Exec Command)

```yaml
exec: "myapp serve"

leaser:
  type: s3
  bucket: my-bucket
  path: myapp

dbs:
  - path: /data/myapp.db
    replica:
      url: s3://my-bucket/myapp
```

Child receives `LITESTREAM_ROLE=primary` or `LITESTREAM_ROLE=replica`.

### 6.2 Separate Commands Per Mode

```yaml
exec-primary: "myapp serve --mode=readwrite"
exec-replica: "myapp serve --mode=readonly"

leaser:
  type: s3
  bucket: my-bucket
  path: myapp

dbs:
  - path: /data/myapp.db
    replica:
      url: s3://my-bucket/myapp
```

### 6.3 Rails Example

```yaml
exec: "bundle exec rails server -b 0.0.0.0"

leaser:
  type: s3
  bucket: my-app-litestream
  path: production/myapp
  ttl: 30s

dbs:
  - path: /data/production.sqlite3
    replica:
      type: s3
      bucket: my-app-litestream
      path: production/myapp
      region: ap-southeast-2

logging:
  level: info
```

Rails initializer:
```ruby
# config/initializers/litestream.rb
LITESTREAM_ROLE = ENV.fetch("LITESTREAM_ROLE", "primary")

Rails.application.configure do
  if LITESTREAM_ROLE == "replica"
    # Open the hydrated replica file as read-only
    config.after_initialize do
      ActiveRecord::Base.connection_pool.disconnect!
      ActiveRecord::Base.establish_connection(
        adapter: "sqlite3",
        database: ENV.fetch("LITESTREAM_HYDRATION_PATH", "/data/production.sqlite3"),
        flags: SQLite3::Constants::Open::READONLY
      )
    end
  end
end
```

---

## 7. Future Enhancements (Out of Scope)

### 7.1 HTTP Proxy Middleware for Read-After-Write Consistency

A Rack middleware (or equivalent) that:
- On write responses: sets a `__txid` cookie with the current transaction ID
- On read requests: if cookie present, waits until replica's `litestream_txid()` >=
  cookie TXID before serving

This is application-layer, not Litestream-layer. The primitives exist (`litestream_txid()`,
`litestream_lag()`). Rails' built-in `ActiveRecord::Middleware::DatabaseSelector` provides
a pluggable framework for this — the default time-based resolver could be replaced with a
TXID-based one.

### 7.2 Write Forwarding

Replicas forwarding write requests to the primary. Options:
- Load balancer level (route non-GET to primary)
- Application middleware (reverse proxy to lease holder's address)
- Requires lease holder to advertise its address in the lock object

### 7.3 Lease Health Endpoint

Expose a `/health` or `/ready` endpoint from Litestream that returns:
- Current role (primary/replica)
- Current TXID
- Lease status and TTL remaining
- Replication lag (replica mode)

Useful for load balancer health checks and routing.

### 7.4 Zero-Downtime Promotion

Investigate whether the child process could be kept running during promotion by:
- Switching the underlying database file via symlink
- Using SQLite's backup API within the child process
- Hot-reloading the database connection pool

This would eliminate the outage window but adds significant complexity.

---

## 8. Review: Lost Writes and Stale Writer Analysis

### 8.1 Critical: No Fencing on LTX Writes to S3

The S3 `WriteLTXFile` implementation (`s3/replica_client.go:637-680`) performs a plain
`PutObject` with no conditional write semantics (`IfMatch`/`IfNoneMatch`). LTX files are
keyed purely by TXID range (`{path}/{level:04x}/{minTXID:016x}-{maxTXID:016x}.ltx`) with
no generation or writer ID in the path or metadata.

This means **a stale writer can silently overwrite LTX files written by the new primary**.
S3 is last-writer-wins; there is no mechanism to reject writes from a writer that no longer
holds the lease.

**The lease `Generation` field** (`leaser.go:34`) exists but is **only used within
`lock.json` itself**. It is never propagated to LTX file paths, S3 object metadata, or
the LTX file content. There is no fencing token pattern in the write path.

### 8.2 Lost Write Scenario: Split-Brain S3 Overwrite

The spec (section 4.7) acknowledges the split-brain risk window but underestimates its
impact. The dangerous case is not "two primaries writing simultaneously" — it's the
**demotion shutdown path actively pushing stale data to S3 after the lease is lost**.

```
Timeline:
  T=0s   A holds lease, is primary, TXID=100
  T=5s   Network partition begins (or lease renewal fails for any reason)
  T=10s  A's renewal fails at TTL/3 check → demotion triggered
  T=10s  A sends SIGTERM to child process
  T=11s  Child commits in-flight transaction (TXID 101), then exits
  T=12s  A's Store.Close() runs "final sync" → uploads TXID 101 to S3
         This SUCCEEDS (network may have recovered, or was only partially down)

  Meanwhile:
  T=10s  B acquires expired lease, begins promotion
  T=12s  B restores from S3 (gets up to TXID 100)
  T=14s  B's child starts writing → B's first write is also TXID 101
  T=15s  B uploads its own TXID 101 to S3

  Result: B's TXID 101 overwrites A's TXID 101 (or vice versa, depending on
          timing). The two files contain DIFFERENT page data for the same TXID.
          One writer's data is silently lost.
```

The spec's section 4.4.4 step 4 says: "Close Store (final sync — may fail if lease truly
lost, that's OK)". The "that's OK" is incorrect. **If the sync succeeds, that's the
dangerous case** — stale data lands on S3 with no way to distinguish it from legitimate
writes.

### 8.3 Lost Write Scenario: Partial Network Partition

Network partitions are often asymmetric. Instance A might be able to upload LTX files to
S3 (data plane works) while failing to renew the lease (control plane is unreachable, or
the specific `lock.json` key hits a different S3 partition that's affected). This creates
a window where both A and B are actively writing LTX files to overlapping TXID ranges.

### 8.4 Lost Write Scenario: Slow Child Shutdown During Demotion

Section 4.4.4 specifies SIGTERM → wait → SIGKILL for child shutdown. During the wait:
- The child is still writing to SQLite (committing in-flight transactions)
- The DB WAL monitor goroutine is still running
- The Replica sync goroutine is still uploading LTX files to S3

The demotion path does not stop the Replica sync goroutine **before** sending SIGTERM to
the child. Writes that commit during the shutdown window get replicated to S3 without any
lease validation.

### 8.5 VFS Write Path vs Primary Write Path Asymmetry

The VFS write path (`vfs.go:1543-1607`) has a `checkForConflict()` call that compares
`expectedTXID` against the remote before uploading. This provides some protection (though
it's a TOCTOU race, not an atomic conditional write).

The **primary DB replication path** (`replica.go:176-194`) has **no equivalent check**. It
iterates through the TXID range and calls `uploadLTXFile` with no conflict detection.
This is the path that runs during normal primary operation and during the final sync on
shutdown.

### 8.6 Recommended Mitigations

**Option A: Generation-prefixed LTX paths (strongest)**

Include the lease generation in the LTX file path:
```
{path}/{generation:016x}/{level:04x}/{minTXID:016x}-{maxTXID:016x}.ltx
```
Each new primary writes to a new generation namespace. Replicas read from the latest
generation. Old generations are garbage-collected. This completely eliminates cross-writer
overwrites but requires changes to `ReplicaClient`, restore logic, and compaction.

**Option B: Conditional writes on LTX upload (moderate)**

Use `IfNoneMatch: "*"` on `PutObject` for L0 LTX files. If the file already exists (another
writer already wrote that TXID), the upload fails with 412 PreconditionFailed. The stale
writer detects it has been superseded and stops. This is simpler than Option A but doesn't
protect against the case where the stale writer uploads first.

**Option C: Stop replication before SIGTERM on demotion (minimum)**

In the demotion path (section 4.4.4), reorder the steps:
```
1. Stop lease renewal goroutine
2. Stop Replica sync goroutine immediately (prevent further S3 uploads)
3. SIGTERM child process
4. Wait for child exit
5. Do NOT run final sync (we don't hold the lease; any sync is dangerous)
6. Enter REPLICA mode
```
This is the minimum viable fix. It narrows the window but doesn't fully close it because
the child could still commit transactions that the WAL monitor picks up and queues for
sync before the sync goroutine is stopped.

**Recommended approach**: Combine Options A and C. Use generation-prefixed paths to make
cross-writer overwrites impossible, and stop replication before child shutdown to minimize
unnecessary writes during demotion.

---

## 9. Open Questions

1. **LTX write fencing**: Given the analysis in section 8, what level of protection is
   acceptable? Generation-prefixed paths (Option A) are the safest but most invasive.
   Conditional writes (Option B) are a good middle ground. Stopping replication before
   SIGTERM (Option C) is the minimum. This should be decided before implementation begins.

2. **VFS in main binary vs Hydrator-only**: Should the main `litestream` binary support
   VFS registration (requires CGo), or should replica mode exclusively use the Hydrator
   to maintain a local file? The Hydrator-only approach is simpler and keeps the binary
   pure-Go, but means the child process reads a file that is being written to by another
   goroutine (though SQLite WAL mode with `PRAGMA query_only=1` handles this safely as
   long as both sides use SQLite's locking protocol).

3. **Hydrator file safety**: The Hydrator writes pages with raw `WriteAt()` calls,
   bypassing SQLite's locking. Need to verify that a child process reading the same file
   via standard SQLite won't see torn pages or inconsistent state. Options:
   - Write to temp file, atomic rename (safe but causes brief connection drop)
   - Use SQLite backup API instead of raw page writes (safe, respects locking)
   - Accept the risk (4KB page writes are typically atomic on Linux ext4/xfs)

4. **Lease TTL vs poll interval**: The default lease TTL is 30s and the VFS poll interval
   is 1s. During promotion, the new primary's first write needs to land in S3 before any
   remaining replicas poll for updates. Is there a race condition if the old primary's
   last writes and new primary's first writes create overlapping LTX TXID ranges?
   The LTX format's TXID ordering should prevent this, but needs verification.

5. **Multiple databases**: The current design assumes one database. If `dbs:` contains
   multiple databases, should the lease cover all of them (single primary for all DBs)
   or should each DB have its own lease? Single lease is simpler and correct for most
   use cases.

---

## 10. References

- Litestream upstream: https://github.com/benbjohnson/litestream (commit c32f6c8)
- S3 Leaser: `s3/leaser.go`, `s3/leaser_test.go`
- VFS: `vfs.go` (Hydrator at line 542+)
- Exec mode: `cmd/litestream/replicate.go` (line 360+)
- Leaser interface: `leaser.go`
- Restore: `replica.go` (line 520+)
- CalcRestorePlan: `replica.go` (line 994+)
- LiteFS proxy (reference implementation): `superfly/litefs`, `http/proxy_server.go`
- litestream-ruby gem: https://github.com/fractaledmind/litestream-ruby (v0.14.0, pins upstream v0.3.13)
- Rails multi-database: https://guides.rubyonrails.org/active_record_multiple_databases.html
