Dqlite was designed to provide applications with a highly-available database with all characteristics that have made SQLite so successful (simplicity, speed, reliability, etc).

It sits on the shoulders of the giant, as it internally uses SQLite itself for the core SQL engine, augmenting it with [Raft-based replication](https://dqlite.io/docs/replication) and a [wire protocol](https://dqlite.io/docs/protocol) for serving networked clients.

## Embedding

Just like SQLite, Dqlite is meant to be used as a library embedded in your application, rather than a fully separate process.

For example, if your application is some kind of network service that exposes an API over the network and saves its data into a regular SQLite database file, your architecture might look like:

![SQLite architecture](https://assets.ubuntu.com/v1/4472303a-sqlite-model.png)

The remote application client connects to your application process and internally your application uses SQLite as a normal library to run SQL queries against the local database file.

If you want your application to be highly-available, you can use Dqlite instead of SQLite. In that case your architecture will look mostly similar, but you will have several application nodes instead of one:

![Dqlite architecture](https://assets.ubuntu.com/v1/122d236c-dqlite-model.png)

In this case the remote application client can connect to **any** of your application processes, and internally your application uses Dqlite as an embedded and replicated database server.

There is an important difference between the two models above though.

The SQLite model is [serverless](https://www.sqlite.org/serverless.html) and there's no network communication involved when running queries: your application process simply invokes *SQLite library functions* which under the hood read and write data directly from and to the local disk.

The Dqlite model is more similar to a "regular" database with [client/serve](https://en.wikipedia.org/wiki/Client%E2%80%93server_model)r network communication: each of the application processes in your cluster spawns a *Dqlite server thread* and then uses a *Dqlite network client* which under the hood connects to the particular Dqlite server thread that happens to be the current cluster leader. This means that the SQL queries issued by an application process using the Dqlite network client might end up being processed by the Dqlite server thread of that very same process (if that node is currently the leader) or by the Dqlite server thread of another application process. The leader in turn connects to the other Dqlite server threads to replicate transactions and commits a transaction only when a quorum of nodes has persisted it durably to disk.

## Server thread

The Dqlite server thread running in an application process uses [`libuv`](http://libuv.org/) to handle all its network and disk I/O asynchronously.

When spawned, it immediately starts listening to a TCP or abstract Unix socket, accepting network connections coming either from Dqlite clients or from remote Dqlite server threads running on other application nodes. Clients submit SQL queries and fetch the associated result, or issue cluster management commands such as adding or removing an application node. Remote Dqlite server threads send Raft requests to elect a new leader or replicate a write transaction.

The network protocol both for clients and for Raft replication is message-oriented. Since I/O is asynchronous and driven by `libuv`'s event loop, messages are read incrementally from network sockets as soon as new data is available, so reads never block the thread. When a message is fully received, it gets processed immediately and completely. As part of processing a message, the server thread might queue outgoing messages or disk writes, which will be picked up asynchronously by the event loop after the processing completes.

## Client library

Application code communicates with a Dqlite server thread using a Dqlite client library typically written in the same programming language used by the application itself. The client library is in charge of establishing a network connection to the target Dqlite server thread and then handling the message-based communication with it.

When a certain application node wants to run a SQL query, the client library finds the current Dqlite server thread leader and sends it a message to execute the desired SQL query. The connection with the leader Dqlite server thread is then re-used for subsequent queries until that Dqlite server thread remains the leader. If the connection drops or the connected Dqlite server stops being the leader, the client needs to find who the new leader is and start talking to it instead.
