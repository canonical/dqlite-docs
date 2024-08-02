## Are Windows and macOS supported?

Not at the moment, because under the hood Dqlite uses the Linux-specific `io_submit` asynchronous file system write API. That code leaves behind an interface that could be adapted to OSX and Windows though. See also this [issue](https://github.com/canonical/go-dqlite/issues/21).

## Is there 24/7 support available?

Not at the moment. But [Canonical](https://www.canonical.com), the company who’s funding Dqlite, can arrange a support contract if desired.

## Is there a commitment to long term releases?

The v1 series will be maintained, improved and bug-fixed for the foreseeable future and backward compatibility is guaranteed.

## How does Dqlite behave during conflict situations?

Dqlite uses Raft to commit write transactions in the same order across all nodes in the cluster. In the Raft model, only a current leader is allowed to start replicating a new transaction, and a transaction is only committed after a majority of nodes acknowledge it. Because of the first rule, conflicts don't arise in a healthy cluster where the leader is stable. In the presence of leadership changes, the Raft logs of distinct nodes can become mismatched, but Raft will repair these differences once the leader stabilises, and because of the majority rule two nodes cannot commit different sequences of transactions. Once committed, a transaction is never cancelled or rolled back.

From the perspective of a Dqlite client, some possible outcomes when submitting a write transaction `T` to a node `N`:

* The request fails immediately because `N` is not the leader.
* The request fails after a delay (during which the client is waiting for leader's response) because `N` lost leadership before fully replicating it. In this case, it's guaranteed that no other transaction will observe the effects of `T`.
* The request succeeds. In this case, `T` will never be rolled back.

See also the [consistency model](./consistency-model.md).

## When not enough nodes are available, are writes hung until consensus?

Yes, however, there’s a (configurable) timeout. This is a consequence of Raft sitting in the CP spectrum of the CAP theorem: in case of a network partition, it chooses consistency and sacrifices availability.

## Does Dqlite support `VACUUM`?

Not currently. Because Dqlite depends the numbering of database pages to replicate changes between nodes, vacuuming needs to be coordinated using the Raft log in the same way that write transactions are. We expect to implement this in a future Dqlite release.

## Does Dqlite support `ATTACH DATABASE`?

Not currently: Dqlite assumes that each write transaction affects only one database. Even if this constraint were removed, it would be difficult to fit such multi-database transactions into the Raft model, since [even in SQLite they are not atomic in WAL mode](https://sqlite.org/lang_attach.html#details).

## How does Dqlite compare to rqlite?

The main differences from [rqlite](https://github.com/rqlite/rqlite) are:

* Can be embedded in any language that can inter-operate with C
* Full support for transactions
* No need for statements to be deterministic (e.g. you can use `time()` )
* Frame-based replication instead of statement-based replication

See the [Comparing Litestream, rqlite, and Dqlite](https://gcore.com/blog/comparing-litestream-rqlite-dqlite/) blog post for a comparison of different SQLite replication implementations.

## Why C?

The first prototype implementation of Dqlite was in Go, leveraging the [`hashicorp/raft`](https://github.com/hashicorp/raft/) implementation of the Raft algorithm. The project was later rewritten entirely in C because of performance problems due to the way Go inter-operates with C: Go considers a function call into C that lasts more than ~20 microseconds as a blocking system call, in that case, it will put the goroutine running that C call in waiting queue and resuming it will effectively cause a context switch, degrading performance (since there were a lot of them happening). See also [this issue](https://github.com/golang/go/issues/19574) in the Go bug tracker.

The added benefit of the rewrite in C is that it’s now easy to embed Dqlite into project written in effectively any language since all major languages have provisions to create C bindings.
