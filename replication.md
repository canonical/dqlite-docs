Dqlite uses the [Raft algorithm](https://raft.github.io/) to replicate SQLite write transactions across application nodes and to provide fault tolerance. For example say you have a cluster of three application nodes, all sharing the same distributed Dqlite database. If one node fails or is shutdown for maintenance, then the remaining two nodes can continue to access the database and operate normally.

Here follow the details about what SQLite data is actually replicated and how.

## Gateway

As described in the [architecture](https://dqlite.io/docs/architecture) overview, in order to run SQL statements a *Dqlite network client* needs to connect to the current *Dqlite server thread* leader which essentially acts as a gateway between the client and SQLite: the client submits SQL requests using the Dqlite [wire protocol](https://dqlite.io/docs/protocol) and the server internally drives SQLite using calls such as [`sqlite3_prepare()`](https://www.sqlite.org/capi3ref.html#sqlite3_prepare) and [`sqlite3_step()`](https://www.sqlite.org/capi3ref.html#sqlite3_step) in order to serve the client requests.

When a client performs a read transaction (e.g. a transaction that involves only `SELECT` statements) the server merely maps client requests into equivalent SQLite calls and returns the result to the client.

However when the client performs a write transaction (e.g. a transaction that involves `INSERT` or `UPDATE` statements), then the server also takes care of encoding the resulting database changes into a Raft log entry and of replicating it to the rest of the cluster. Once a quorum of nodes has received and persisted those new Raft log entries, the server tells the client that the transaction was successfully committed.

## In-memory file system

When SQLite needs to interact with the operating system to read or write files it does so using an interface called [`sqlite3_vfs`](https://www.sqlite.org/vfs.html) (aka VFS). The default implementation of that interface that comes with SQLite simply invokes system calls such as `read()` and `write()` which persist data on disk using regular files.

Dqlite configures SQLite to use a custom VFS implementation which stores files in regular process memory and not on disk (a bit like `tmpfs`). All I/O operations that SQLite performs are therefore handled by Dqlite, which instead of executing `read()` and `write()` system calls, uses `memcpy` to read and write its in-memory file images.

Roughly speaking when SQLite commits a write transaction, Dqlite collects all I/O writes belonging to that transaction,  encodes them into a Raft a log entry, and replicates that entry to the cluster. The Raft log is therefore what gets persisted on disk, rather than regular SQLite database files. More details on this below.

Earlier versions of Dqlite had a simpler custom VFS than the current one. It was used purely as in-memory storage to avoid persisting data twice (in a SQLite file and in the Raft log) , and had no extra functionality for "watching" SQLite's I/O and extracting transactions. Instead, Dqlite relied on a patch to be applied to the upstream SQLite source code, introducing various hooks during a transaction life cycle and allowing Dqlite to extract the information it needed for replication. That approach has been abandoned because SQLite is [Open-Source, not Open-Contribution](https://www.sqlite.org/copyright.html) and it is very difficult to get any changes merged upstream, for good and for bad.

## Write-Ahead Log

Dqlite also configures SQLite to use a [Write-Ahead Log](https://sqlite.org/wal.html) (WAL) for implementing atomic commits and rollbacks.

In this mode SQLite uses two files to store data: one is the *main database file*, and the other is the *WAL file*. When an SQLite transaction that modifies the database wants to commit those changes, it does not write the database pages it has modified directly to the database file, but it rather appends them to the WAL file. The last page of a write transaction appended to the WAL will contain a *commit marker* that marks the end of the transaction. Once the WAL file grows too big, a checkpoint operation must be performed, and the latest version of each page of each committed transaction it contains gets copied back to the main database file.

This means that read transactions started before or during a write transactions can proceed concurrently with that write transaction while retaining full *isolation* and *consistency*: they just need to take note of the position of the last commit marker in the WAL at the time they started and only read either WAL pages before that position or database pages that were not modified, avoiding pages that were appended to the WAL after the read transaction started.

There is also a third file that SQLite uses in WAL mode: the *WAL index file*. This is a transient file and its content can be re-created by reading the WAL file. It contains a hash table that makes it fast to lookup the position of the most recent database page in the WAL before a certain position. The WAL index file gets normally `mmap`'ed by the default VFS implementation, but Dqlite's custom VFS implementation just uses regular memory since it does not need to share that memory across processes.

A certain region of the WAL index file is used for read and write locks. The default VFS implements them with POSIX advisory locks, while Dqlite's custom one uses simple flags in memory.

## Transactions

At any time at most one write transaction can be started. When SQLite starts a write transaction it acquires an *exclusive write lock* against the WAL file by calling the [`xShmLock()`](https://www.sqlite.org/capi3ref.html#sqlite3_io_methods) method of the VFS against a particular byte of the locks section of the WAL index shared memory. Internally the custom Dqlite VFS implementation takes note of that write lock by setting a flag. Attempts to start other write transactions will fail with `SQLITE_BUSY`.

With the write lock acquired, the write transaction starts modifying the copies of the database pages that it holds in the connection's page cache, marking them as dirty. When the transaction gets committed (either implicitly or explicitly), SQLite appends all these modified database pages to the WAL file. At this point instead of immediately appending those modified pages to the WAL file image, the Dqlite VFS implementation stores those pages into a *separate memory area*, encodes them into a new Raft log entry and submits that entry for replication.

Since replication happens asynchronously, new read transactions from other connected clients might be started concurrently, but since the WAL file image hasn't actually been changed yet they will **not** observe the changes of the write transaction which is being committed.

New write transactions are prevented by re-marking as acquired the exclusive WAL write lock which was released by the original write transaction, which is completed from SQLite's point of view but is effectively still in-flight since it's being replicated.

When a quorum is reached and that Raft log entry gets committed, then Dqlite finally updates the WAL file image stored in the custom VFS to include the new pages, releases the exclusive write lock and sends a success reply to the client indicating that the transaction has completed. At that point a new write transaction can be started.

If the Raft log entry does not get replicated in a timely manner, then leadership is eventually lost and at that point Dqlite sends an error reply to the client, which is supposed to retry (see below).

## State machine

The Raft protocol replicates an ordered log of entries across cluster nodes. Once entries are committed, nodes apply them to their state machines whose state is deterministic and depends only on those entries. Raft-based system often need to maintain very little state outside the state machine.

For example when a leader receives a command from a client, it mostly only needs to keep track that it has an active connection with that client and that the client has submitted a command. Once the command is committed and the state machine applies it, the leader just needs to grab the state machine's result and send it back to the client.

In Dqlite the state machine is represented by only the *database file image* and the *WAL file image*. The two commands that drive the state machine are: append new pages to the WAL as the result of committing a write transaction and checkpoint the WAL into the database for compaction. That's basically the same state that SQLite would otherwise persist on disk.

Differently from simplistic Raft-based systems a Dqlite client can't just produce a state machine Raft command directly: for example for submitting an "append these pages to the WAL" command it would need to know which database pages to modify and how, but it can't since the database is not transferred to the client.

Instead, Dqlite leaders need to maintain more state than a typical Raft system. For each client they maintain a SQLite connection which is used to "accumulate" the various database changes performed during a transaction and finally encode those changes into a Raft log entry at commit time. They also need to maintain the WAL index, modifying it and rolling it back depending on the transaction status.

## Checkpoints

Dqlite nodes need to periodically perform a *checkpoint* operation where the updates described by the WAL are applied to the database file, preventing the WAL from growing too large. In previous versions of Dqlite, all checkpoints were initiated by the leader and communicated via a special log entry to followers. In the current version, all nodes (leader or follower) can run checkpoints independently of one another, but leaders still propagate a special log entry after creating a checkpoint to signal to older followers that they should do the same.

The server implementation automatically runs a checkpoint whenever the WAL grows beyond a fixed size limit.

## Client sessions (in progress, not released yet)

Before being able to issue SQL statements, a Dqlite client needs to register itself using a unique client UUID.  That UUID will be committed in the Raft log, so all Dqlite state machines on all nodes will be aware of the client.

When a Dqlite client sends to to server a message to commit a transaction (either explicitly using `COMMIT` or implicitly by running a standalone statement), it includes in the message also a sequence number which is increased after each successful commit.

If the client receives a retriable error code from the server (for example the server lost leadership) or if the client receives no reply at all (for example the server crashed), that means that the the commit might have failed to reach a quorum and hence will be discarded or that it managed to be replicated to a quorum of nodes but the client could not be  notified about that. The only way the client can figure out what happens is to retry to submit a commit message with the same sequence number to a new leader. A new leader always appends a no-op log entry at the start of their term so it eventually discovers all entries that were committed in previous terms. When the new leader receives the retry request it waits for the no-op log entry to be committed, and then it looks at its state machine to know if the transaction being retried was actually committed or if was discarded, and replies to the client accordingly. If the new leader gets deposed or fails before replying, the client needs to repeat the process.
