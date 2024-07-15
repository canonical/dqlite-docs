## Are Windows and macOS supported?

Not at the moment, because under the hood Dqlite uses the Linux-specific `io_submit` asynchronous file system write API. That code leaves behind an interface that could be adapted to OSX and Windows though. See also this [issue](https://github.com/canonical/go-dqlite/issues/21).

## Is there 24/7 support available?

Not at the moment. But [Canonical](https://www.canonical.com), the company who’s funding Dqlite, can arrange a support contract if desired.

## Is there a commitment to long term releases?

The v1 series will be maintained, improved and bug-fixed for the foreseeable future and backward compatibility is guaranteed.

## How does Dqlite behave during conflict situations? Does Raft select a winning WAL to write and any others in-flight writes are aborted?

Dqlite uses Raft to commit write transactions in the same order across all nodes in the cluster. Only a current Raft leader is allowed to start replicating a new write transaction, so in a healthy cluster where the leader is stable, no conflicts arise. Trying to execute a transaction on a non-leader node will automatically fail.

In a cluster where leadership isn't stable (perhaps because communication between the nodes is disrupted), the Raft logs of distinct nodes can become mismatched. But Raft's rules ensure that a leader that remains in power for long enough will repair these mismatches, and in the meantime no transaction can be committed unless acknowledged by a majority of the cluster.

From the perspective of a Dqlite client, some possible outcomes when submitting a write transaction `T` to a node `N`:

* The request fails immediately because `N` is not the leader.
* The request fails after a delay because `N` lost leadership before fully replicating it. In this case, it's guaranteed that no other transaction will observe the effects of `T`.
* The request succeeds. In this case, `T` will never be rolled back.

See also the [consistency model](./consistency-model.md).

## When not enough nodes are available, are writes hung until consensus?

Yes, however, there’s a (configurable) timeout. This is a consequence of Raft sitting in the CP spectrum of the CAP theorem: in case of a network partition, it chooses consistency and sacrifices availability.

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
